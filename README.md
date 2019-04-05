# BeauPools

**Current Version: 0.1.0**  
Updated 5 April 2019 | Changelog (Not yet available)

## About
BeauPools is an object pooling library for Unity. It contains both generic pools and a Unity-specific prefab pool, along with utilities for managing those prefab pools.

### Table of Contents
1. [Usage](#usage)
    * [Installing BeauPools](#installing-beaupools)
    * [Pool Types](#pool-types)
    * [Creating/Destroying a Pool](#creatingdestroying-a-pool)
    * [Populating a Pool](#populating-a-pool)
    * [Using a Pool](#using-a-pool)
    * [Callbacks and Events](#callbacks-and-events)
2. [PrefabPool](#prefabpool)
	* [PrefabPool Requirements](#prefabpool-requirements)
	* [SerializablePool](#serializablepool)
3. [Reference](#reference)
	* [Pool Members](#pool-members)
	* [PoolConfig Members](#poolconfig-members)
	* [PrefabPool Members](#prefabpool-members)
	* [SerializablePool Members](#serializablepool-members)
	* [Misc Utilities](#misc-utilities)
----------------

## Usage

### Installing BeauPools

Download [BeauPools.unitypackage](https://gitlab.filamentgames.com/internal/tools/beaupools/raw/master/BeauPools.unitypackage) from the repo. Unpack it into your project.

BeauData uses the ``BeauPools`` namespace. You'll need to add the statement ``using BeauPools;`` to the top of any scripts using it.

### Pool Types

There are three types of pools in BeauPools: Fixed, Dynamic, and Prefab.

| Pool Type | Capacity | Requires Constructor | Notes |
| ----- | -------| - | - |
| Fixed | Fixed capacity. If allocating from empty pool, an exception is thrown. | ✓ | |
| Dynamic | Dynamic capacity. If allocating from empty pool, a new element will be constructed. | ✓ | |
| Prefab | Dynamic pool (see above). | X | Unity-specific behavior documented [below](#prefabpool). |

### Creating/Destroying a Pool

```csharp
public class SomeObject { ... }

// -- Creating a FixedPool --

// Max size for the FixedPool
int fixedPoolCapacity = 32;

// Constructor, required for generating pool elements
// This should return identical elements every call
// Pool.DefaultConstructor returns a default constructor for types with empty constructors
Constructor<SomeObject> someObjectConstructor = Pool.DefaultConstructor<SomeObject>();

// Strict typing
// True by default. Set this to false if you don't know
// exactly the type of object being generated by the constructor
bool fixedPoolStrictTyping = true;

// Finally, generating the pool
FixedPool<SomeObject> fixedPool = new FixedPool<SomeObject>( fixedPoolCapacity, someObjectConstructor, fixedPoolStrictTyping );

// -- Creating a DynamicPool --

// You can use more specific constructors if desired
Constructor<SomeObject> customObjectConstructor = (IPool<SomeObject> pool) => { return new SomeObject(); };

DynamicPool<SomeObject> dynamicPool = new DynamicPool<SomeObject>( 16, customObjectConstructor );

// -- Destroying a Pool ---

// To destroy a pool, call Dispose()
fixedPool.Dispose();
dynamicPool.Dispose();
```

### Populating a Pool

By default, pools do not start with any elements. You must prewarm the pool with its initial set of elements.

```csharp
// This will ensure the pool contains 16 elements
somePool.Prewarm( 16 );

// This will fill the pool up to its current capacity.
somePool.Prewarm();
```

You can shrink the number of elements in the pool down as well.

```csharp
// This will ensure the pool contains no more than 16 elements.
somePool.Shrink( 16 );
```

You can also clear a pool of all its elements with ``Clear()``.

### Using a Pool

To allocate an element from the pool, call ``Alloc()``. To return it to the pool, call ``Free( [element] )``.

```csharp
// Allocate the element from the pool
var element = somePool.Alloc();

// Do something with the pooled element
...

// Return the element to the pool
somePool.Free(element);
```

You can also allocate and free in batches.

```csharp
List<SomeObject> objectInUse = new List<SomeObject>();

// This will allocate 16 elements and add them to the list
somePool.Alloc(objectsInUse, 16);

// Do something with those elements
...

// Free them all at once
somePool.Free(objectsInUse);
```

### Callbacks and Events

If your pooled object inherits from the ``IPooledObject<T>`` interface, it will receive certain calls from the pool during the object's lifecycle. These events are ``OnConstruct``, ``OnAlloc``, ``OnFree``, and finally ``OnDestruct``.

```csharp
public class PooledObjectWithCallbacks : IPooledObject<PooledObjectWithCallbacks>
{
	// Called when this object is constructed for the given pool 
	public void OnConstruct(IPool<T> inPool)
    {
		...
	}
    
    // Called when the element is allocated from the pool
    public void OnAlloc()
    {
		...
	}
    
    // Called when the element is returned to the pool
    public void OnFree()
    {
		...
	}
    
    // Called when the element is destroyed from within the pool
    public void OnDestruct()
    {
    	...
    }
}
```

Some suggested uses for these callbacks:
* Use the ``OnAlloc`` or ``OnFree`` callbacks to help reset state.
* Use ``OnConstruct`` to remember the pool that owns this object. This would allow you to write a method to return the object to its owning pool.

Every pool also contains an ``PoolConfig`` object. This allows you to register additional callbacks on a per-pool rather than per-element basis. See [PoolConfig Members](#poolconfig-members) below for documentation.

## PrefabPool

### PrefabPool Requirements

PrefabPools are pools intended for instantiating and pooling Unity prefab objects. To this end, they have the following properties.
* New instances are assigned as children of a specific Transform. You should ensure this Transform is disabled to avoid unwanted behaviour from your pooled instances.
* By default, allocated instances are assigned as children of a specific Transform, or the root of the scene if no Transform is provided.
* By default, allocated instances reset their position and orientation on allocation. This can be disabled.
* PrefabPool provides overridden versions of the ``Alloc`` method to mirror some of Unity's ``Instantiate`` methods. See [PrefabPool Members](#prefabpool-members) below for documentation.

### SerializablePool

BeauPools provides the ``SerializablePool`` object for easy setup of a PrefabPool in the inspector. Note that due to Unity's serialization rules, you must create a subclass of SerializablePool for each type of component you with to pool.

```csharp
// This is necessary, since Unity won't serialize generic classes
[Serializable]
public SomePooledBehaviourPool : SerializablePool<SomePooledBehaviour> { }

public class TestSerializedPool : MonoBehaviour
{
	[SerializeField]
	private SomePooledBehaviorPool myPool;
    
    private void Start()
    {
  		// Initialize must be called before the pool is used
		myPool.Initialize();
	}
    
    private void OnDestroy()
    {
    	// Make sure to destroy your pool before it goes out of scope
    	myPool.Destroy();
    }
    
    public SomePooledBehaviour Spawn()
    {
    	// Once the pool is initialized, you can retrieve its inner PrefabPool
        // and call its methods
    	return myPool.InnerPool.Alloc();
    }
}

```

## Reference

### Pool Members

| Member | Type | Description |
| - | - | - |
| ``Capacity`` | Property | Current capacity of the pool. In fixed pools, this is a hard limit. In dynamic pools, this is a softer limit. |
| ``Count`` | Property | Current number of elements in the pool. |
| ``InUse`` | Property | Current number of elements allocated from the pool. |
| ``Config`` | Property | Event configuration. Use this to hook up callbacks to specific pool events. |
| ``Prewarm(int count)`` | Method | Ensures the pool contains at least ``[count]`` number of elements. |
| ``Prewarm()`` | Method | Ensures the pool contains up to its current capacity in elements. |
| ``Shrink(int count)`` | Method | Ensures the pool contains no more than ``[count]`` elements. |
| ``Clear()`` | Method | Clears all elements from the pool. |
| ``Dispose()`` | Method | Clears all elements and cleans up memory used by the pool. |
| ``Alloc()`` | Method | Allocates an element from the pool. |
| ``Alloc(T[] array)`` | Method | Allocates elements from the pool to empty slots in the given array. |
| ``Alloc(ICollection<T> collection, int count)`` | Method | Allocates ``[count]`` elements and adds them to the given collection. |
| ``TryAlloc(out T element)`` | Method | Attempts to allocate an element from the pool. |
| ``TryAlloc(T[] array)`` | Method | Attempts to allocate elements from the pool to empty slots in the given array. |
| ``TryAlloc(ICollection<T> collection, int count)`` | Method | Attempts to allocate ``[count]`` elements and add them to the given collection. |
| ``Free(T element)`` | Method | Returns an element to the pool. |
| ``Free(T[] array)`` | Method | Returns elements from the given array to the pool. |
| ``Free(ICollection<T> collection)`` | Method | Returns elements from the given collection to the pool. |
| ``UseIDisposable()`` | Method | Hooks up the ``IDisposable.Dispose()`` call to object destruction. (Only when ``T`` inherits from ``IDisposable`` )

### PoolConfig Members

| Member | Type | Description |
| - | - | - |
| ``RegisterOnConstruct(LifecycleEvent<T> callback)`` | Method | Registers a callback to be called when an element is constructed. |
| ``RegisterOnDestruct(LifecycleEvent<T> callback)`` | Method | Registers a callback to be called when an element is destroyed. |
| ``RegisterOnAlloc(LifecycleEvent<T> callback)`` | Method | Registers a callback to be called when an element is allocated. |
| ``RegisterOnFree(LifecycleEvent<T> callback)`` | Method | Registers a callback to be called when an element is freed. |

### PrefabPool Members

| Member | Type | Description |
| - | - | - |
| ``Name`` | Property | Name of the pool. Used when naming prefab instances. |
| ``PoolTransform`` | Property | Parent Transform for all inactive elements in the pool. |
| ``DefaultSpawnTransform`` | Property | The default parent Transform to which allocated elements are attached. |
| ``Alloc(Transform parent)`` | Method | Allocates a new element as a child of the given transform. |
| ``Alloc(Vector3 position, bool worldSpace)`` | Method | Allocates a new element with the given position. |
| ``Alloc(Vector3 position, Quaternion orientation, bool worldSpace)`` | Method | Allocates a new element with the given position and orientation. |
| ``Alloc(Vector3 position, Quaternion orientation, Transform parent, bool worldSpace)`` | Method | Allocates a new element as a child of the given transform with the given position and orientation |

### SerializablePool Members

| Member | Type | Description |
| - | - | - |
| ``Name`` | Property | Name of the pool. |
| ``ConfigureCapacity(int capacity, int prewarm, bool prewarmOnInit)`` | Method | Configures capacity and prewarm settings. |
| ``ConfigureTransforms(Transform poolRoot, Transform spawnRoot, bool resetTransformOnAlloc)`` | Method | Configures transform and reset settings. |
| ``Initialize()`` | Method | Initializes the internal pool. |
| ``IsInitialized()`` | Method | Returns if the internal pool has been initialized. |
| ``Prewarm()`` | Method | Prewarms the pool to ensure a certain amount of elements are available. |
| ``InnerPool`` | Property | Returns the internal pool. |
| ``ActiveObjects()`` | Method | Returns the set of currently allocated objects. |
| ``ActiveObjects(ICollection<T>)`` | Method | Adds the set of currently allocated objects to the given collection. |
| ``ActiveObjectCount()`` | Method | Returns the number of currently allocated objects. |
| ``Reset()``| Method | Returns all currently allocated objects to the internal pool. |
| ``Destroy()`` | Method | Calls ``Reset()`` and destroys the internal pool. |

### Misc Utilities

| Utility | Description |
| - | - |
| ``PooledList`` | Pooled list. Useful when you need a temporary ordered collection. |
| ``PooledSet`` | Pooled hashset. Useful when you need a temporary unordered collection. |
| ``PooledStringBuilder`` | Pooled StringBuilder. Useful when you need to temporarily concatenate strings. |
| ``TransformPool`` | ``Transform`` specialization of ``SerializablePool`` |
| ``SpriteRendererPool`` | ``SpriteRenderer`` specialization of ``SerializablePool`` |
| ``RectTransformPool`` | ``RectTransform`` specialization of ``SerializablePool`` |