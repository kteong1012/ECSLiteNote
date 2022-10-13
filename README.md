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
可以通过ComponentPool添加、请求或删除。

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
...
SharedData sharedData = new SharedData { PrefabsPath = "Items/{0}" };
IEcsSystems systems = new EcsSystems (world, sharedData);
systems
    .Add (new TestSystem1 ())
    .Init ();
...
class TestSystem1 : IEcsInitSystem {
    public void Init(IEcsSystems systems) {
        SharedData shared = systems.GetShared<SharedData> (); 
        string prefabPath = string.Format (shared.PrefabsPath, 123);
        // prefabPath = "Items/123"
    } 
}
```
