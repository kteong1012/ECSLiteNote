# 类
## Entity
Entity(实体)主要就是一个int值，是一个id。
```CSharp
// 使用World创建一个新实体
int entity = _world.NewEntity ();

// 任何实体都可以删除，首先将自动删除所有组件，然后实体才会被视为已销毁。
world.DelEntity (entity);

// 任何实体中的组件都可以复制到另一个实体。如果源实体或目标实体不存在，则会在 DEBUG 版本中引发异常。
world.CopyEntity (srcEntity, dstEntity);
```
实体必须有组件，若没有删除所有组件则实体自动销毁。

## Component
Component(组件)是用户数据的容器，不应包含逻辑（允许写ToString之类的帮助逻辑，但不能包含基本逻辑）
```CSharp
struct Component1 {
    public int Id;
    public string Name;
}
```
可以通过EcsPool添加、请求或删除。

## System
System(系统)是用于处理筛选后实体的主逻辑的容器。作为自定义类存在，该类实现至少一个接口：IEcsInitSystem,IEcsDestroySystem,IEcsRunSystem
```CSharp
class UserSystem : IEcsPreInitSystem, IEcsInitSystem, IEcsRunSystem, IEcsDestroySystem, IEcsPostDestroySystem {
    public void PreInit (IEcsSystems systems) {
        // 将在 IEcsSystems.Init() 运行时和触发之前调用一次
    }
    
    public void Init (IEcsSystems systems) {
         // 将在 IEcsSystems.Init() 运行时和触发之后调用一次
    }
    
    public void Run (IEcsSystems systems) {
        //  将在IEcsSystems.Run() 运行时调用一次 
    }

    public void Destroy (IEcsSystems systems) {
        // 将在IEcsSystems.Destroy() 运行时和触发IEcsPostDestroySystem.PostDestroy() 之前调用一次。
    }
    
    public void PostDestroy (IEcsSystems systems) {
        // 将在 EcsSystems.Destroy() 运行时和在 IEcsDestroySystem.Destroy() 触发后调用一次。
    }
}
```

# 共享数据
任何自定义类型(class)的实例都可以同时被所有系统访问
```CSharp
class SharedData {
    public string PrefabsPath;
}

SharedData sharedData = new SharedData { PrefabsPath = "Items/{0}" };
IEcsSystems systems = new EcsSystems (world, sharedData);
systems
    .Add (new TestSystem1 ())
    .Init ();

class TestSystem1 : IEcsInitSystem {
    public void Init(IEcsSystems systems) {
        SharedData shared = systems.GetShared<SharedData> (); 
        string prefabPath = string.Format (shared.PrefabsPath, 123);
        // prefabPath = "Items/123"
    } 
}
```

# 特殊类型
## EcsPool（组件池）
EcsPool(组件池)是组件的容器，提供用于在实体上添加/请求/删除组件的 api：
```CSharp
int entity = world.NewEntity ();
EcsPool<Component1> pool = world.GetPool<Component1> (); 

// Add() 将组件添加到实体。如果该组件已存在，则会在 DEBUG 版本中引发异常。
ref Component1 c1 = ref pool.Add (entity);

// Has() 检查实体上是否存在组件。
bool c1Exists = pool.Has (entity);

// Get() 返回实体上的现有组件。如果该组件不存在，则会在 DEBUG 版本中引发异常。
ref Component1 c1 = ref pool.Get (entity);

// Del() 从实体中删除组件。如果没有组件，则不会出现任何错误。如果它是最后一个组件，则将自动删除该实体。
pool.Del (entity);

// Copy() 将所有组件从一个实体复制到另一个实体。如果源实体或目标实体不存在，则会在 DEBUG 版本中引发异常。
pool.Copy (srcEntity, dstEntity);
```
从组件池中被卸载的组件将被放置在池中以供以后重复使用。组件中的所有字段将自动重置为其默认值。

## EcsFilter（过滤器）
它是一个容器，用于存储特定组件组合的实体：
```CSharp
class WeaponSystem : IEcsInitSystem, IEcsRunSystem {
    public void Init (IEcsSystems systems) {
        // 获取默认的World实例。
        EcsWorld world = systems.GetWorld ();
        
        // 创建一个新Entity
        int entity = world.NewEntity ();
        
        // 给Entity添加Weapon组件
        var weapons = world.GetPool<Weapon>();
        weapons.Add (entity);
    }

    public void Run (IEcsSystems systems) {
        EcsWorld world = systems.GetWorld ();
        // 我们希望获取具有Weapon组件且不包含Health组件的所有实体。
        // 过滤器可以每次动态收集，也可以缓存在某个地方。
        var filter = world.Filter<Weapon> ().Exc<Health> ().End ();
        // 过滤器仅存储本身位于Weapon组件池中的实体。
        // 池也可以缓存在某个地方。
        var weapons = world.GetPool<Weapon>();
        foreach (int entity in filter)
        {
            ref Weapon weapon = ref weapons.Get (entity);
            weapon.Ammo = System.Math.Max (0, weapon.Ammo - 1);
        }
    }
}
```
重要地！筛选器支持任意数量的组件要求，但同一组件不能同时位于包含和排除列表中。

## EcsWorld（世界）
作为所有实体、组件池和筛选器的容器，每个实例的数据都是唯一的，并且与其他世界隔离。

如果不再需要世界实例，则必须调用它。EcsWorld.Destroy()

## EcsSystems（系统组）
处理EcsWorld实例的系统容器：
```CSharp
class Startup : MonoBehaviour {
    EcsWorld _world;
    IEcsSystems _systems;

    void Start () {
        // 创建一个EcsWorld实例和一个EcsSystems实例并绑定。然后在_systems加上WeaponSystem系统并初始化。
        _world = new EcsWorld ();
        _systems = new EcsSystems (_world);
        _systems
            .Add (new WeaponSystem ())
            .Init ();
    }
    
    void Update () {
        // 每帧运行一次_systems.Run()
        _systems?.Run ();
    }

    void OnDestroy () {
        // 销毁系统组
        if (_systems != null) {
            _systems.Destroy ();
            _systems = null;
        }
        // 销毁世界
        if (_world != null) {
            _world.Destroy ();
            _world = null;
        }
    }
}
```
如果不再需要某个实例上的系统组，则必须调用它。IEcsSystems.Destroy()
