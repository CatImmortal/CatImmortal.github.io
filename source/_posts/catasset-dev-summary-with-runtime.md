---
title: CatAsset开发总结：Runtime篇
date: 2022-09-04 08:56:01
tags: 资源管理
categories: 
  - 程序
  - 开发总结
description: Unity资源管理框架
top_img: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithRuntime/CatAssetDevSummaryWithRuntime_01.png
cover: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithRuntime/CatAssetDevSummaryWithRuntime_01.png
---

# 前言

本文主要用于总结[CatAsset](https://github.com/CatImmortal/CatAsset)在Runtime部分的设计思路

代码结构如下：

![](	https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithRuntime/CatAssetDevSummaryWithRuntime_01.png)

CatJson：包含用于进行Json序列化与反序列化的代码

Database：包含清单信息与运行时信息的数据结构

Extensions：包含扩展代码

Misc：包含各类杂项代码

Pool：包含游戏对象池与引用池代码

TaskSystem：包含支持CatAsset Runtime核心功能运转的任务系统代码

Updatable：包含用于可更新模式下的版本检查器与资源组更新器代码



# 资源清单与运行时信息

## 资源清单

**资源清单（CatAssetManifest）**记录了在**Editor**下构建出的所有资源包（`BundleManifest`）及其中资源（`AssetManifest`）的相关信息

对于内置资源而言只有通过`CatAssetManifest`能够读取到相关信息的，才是可被CatAsset管理的



## 运行时信息

对应`BundleManifest`和`AssetManifest`，有`BundleRuntimeInfo`和`AssetRuntimeInfo`，用于保存在游戏运行中产生的相关行为的信息，如**资源实例、引用计数**等



# 任务系统

CatAsset的Runtime中大部分核心功能都是基于**任务系统（TaskSystem）**实现的，此系统主要用于解决下列异步运行相关需求：

- **异步运行间的依赖等待**
- **异步运行到一半需要取消**
- **目标相同的异步运行的合并**
- **异步运行频率的限制**
- **异步运行的优先级**



诸如**加载、卸载、更新**等操作都封装为了对应的**任务（Task）**，被放置于**任务组（TaskGroup）**中，由**任务运行器（TaskRunner）**根据**优先级**对**TaskGroup**进行管理



![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithRuntime/CatAssetDevSummaryWithRuntime_07.png)



## 任务

**任务（Task）**是**TaskSystem**中实际的逻辑运行者，从抽象基类`BaseTask`、接口`ITask`派生

接口`ITask`定义如下：

```csharp
    /// <summary>
    /// 任务接口
    /// </summary>
    public interface ITask : IReference
    {
        /// <summary>
        /// 持有者
        /// </summary>
        TaskRunner Owner { get; }
        
        /// <summary>
        /// 全局id
        /// </summary>
        int GUID { get; }
        
        /// <summary>
        /// 名称
        /// </summary>
        string Name { get; }
        
        /// <summary>
        /// 状态
        /// </summary>
        TaskState State { get; set; }
        
        /// <summary>
        /// 进度
        /// </summary>
        float Progress { get; }

        /// <summary>
        /// 已合并任务数量
        /// </summary>
        public int MergedTaskCount { get; }
        
        /// <summary>
        /// 合并任务
        /// </summary>
        void MergeTask(ITask task);
        
        /// <summary>
        /// 运行任务
        /// </summary>
        void Run();
        
        /// <summary>
        /// 轮询任务
        /// </summary>
        void Update();

        /// <summary>
        /// 取消任务
        /// </summary>
        void Cancel();
    }
```



### 任务状态

**任务状态（TaskState）**表示此`Task`的内部运行情况，其定义如下：

```csharp
	/// <summary>
    /// 任务状态
    /// </summary>
    public enum TaskState
    {
        /// <summary>
        /// 空闲
        /// </summary>
        Free,

        /// <summary>
        /// 等待中
        /// </summary>
        Waiting,

        /// <summary>
        /// 运行中
        /// </summary>
        Running,

        /// <summary>
        /// 已结束
        /// </summary>
        Finished,
    }
```



通常来说

- 未开始运行的`Task`为**Free**状态
- 需要等待其他`Task`运行才能开始运行的`Task`为**Waiting**状态
- 正在运行的`Task`为**Running**状态
- 运行结束的`Task`为**Finished**状态



### 任务合并

对于同名的`Task`，**TaskSystem**会将其放入到已有`Task`的`MergedTaskList`中，已有`Task`运行结束后会将运行结果回调给`MergedTaskList`中的所有`Task`



### 任务取消

`Task`被创建后会获得一个全局唯一的`GUID`，对于支持取消操作的`Task`，可通过此`GUID`进行取消，被取消的`Task`即便运行结束了也不会回调给使用者运行结果



## 任务运行器与任务组

### 任务运行器

**任务运行器（TaskRunner）**是所有`Task`运行的起点

`TaskRunner`在初始化时会按照预定义的**任务优先级（TaskPriority）**去创建对应优先级的**任务组（TaskGroup）**，并按照优先级去轮询`TaskGroup`

```csharp
 	/// <summary>
    /// 任务优先级
    /// </summary>
    public enum TaskPriority
    {
        /// <summary>
        /// 非常低
        /// </summary>
        VeryLow = 0,
        
        /// <summary>
        /// 低
        /// </summary>
        Low = 1,
        
        /// <summary>
        /// 中
        /// </summary>
        Middle = 2,
        
        /// <summary>
        /// 高
        /// </summary>
        Height = 3,
        
        /// <summary>
        /// 非常高
        /// </summary>
        VeryHeight = 4,
        
    }
```

```csharp
 	public TaskRunner()
        {
            //优先级数量
            int priorityNum = Enum.GetNames(typeof(TaskPriority)).Length;

            for (int i = 0; i < priorityNum; i++)
            {
                //按优先级创建任务组
                taskGroups.Add(new TaskGroup((TaskPriority)i));
            }
        }
```



### 任务组

**任务组（TaskGroup）**中保存了以此`TaskGroup`对应优先级来运行的`Task`对象，会在每次`Task`运行后，根据`Task`的`State`来决定是否增加当前运行中的任务计数

```csharp
 		/// <summary>
        /// 运行任务组
        /// </summary>
        public bool Run()
        {

            int index = nextRunningTaskIndex;
            nextRunningTaskIndex++;
            
            ITask task = runningTasks[index];

            try
            {
                if (task.State == TaskState.Free)
                {
                    //运行空闲状态的任务
                    task.Run();
                }

                //轮询任务
                task.Update();
            }
            catch (Exception e)
            {
                //任务出现异常 视为任务结束处理
                task.State = TaskState.Finished;
                throw;
            }
            finally
            {
                switch (task.State)
                {
                    case TaskState.Finished:
                        //任务运行结束 需要删除
                        waitRemoveTasks.Add(index);
                        break;
                };
            }

            switch (task.State)
            {
                case TaskState.Running:
                case TaskState.Finished:
                    return true;
            }

            return false;

        }
```



### 频率限制

`TaskRunner`中定义了单帧最大任务运行数量，在每次`Update`时会根据`TaskGroup`返回结果来统计当前帧运行中的任务数量，如果达到了限制数量就会停止对`TaskGroup`的运行

```csharp
		/// <summary>
        /// 轮询任务运行器
        /// </summary>
        public void Update()
        {
            //当前运行任务次数
            int curRanCount = 0;
            
            for (int i = taskGroups.Count - 1; i >= 0; i--)
            {
                TaskGroup group = taskGroups[i];
                
                group.PreRun();

                while (curRanCount < MaxRunCount && group.CanRun)
                {
                    if (group.Run())
                    {
                        //Run调用返回true 意味着需要增加curRanCount
                        curRanCount++;
                    }
                }
                
                group.PostRun();
            }
        }
```



# 资源加载

## 资源类别的判断

在[CatAsset开发总结：Editor篇](http://cathole.top/2022/09/01/catasset-dev-summary-with-editor/)中提及过3种支持的资源类别：

1. **内置资源包资源**
2. **内置原生资源**
3. **外置原生资源**

CatAsset的`LoadAsse<T>`接口统一了3种类别的资源加载，使用者在加载资源时无需关心其所加载的资源的具体类别



内部判断资源类别的代码如下



### 编辑器资源模式

```csharp
 		/// <summary>
        /// 获取编辑器资源模式下的资源类别
        /// </summary>
        public static AssetCategory GetAssetCategoryWithEditorMode(string assetName, Type assetType)
        {
            if (assetName.StartsWith("Assets/"))
            {
                //资源名以Assets/开头
                if (typeof(UnityEngine.Object).IsAssignableFrom(assetType) || assetType == typeof(object))
                {
                    //以UnityEngine.Object及其派生类型或object为加载类型 
                    //都视为内置资源包资源进行加载
                    return AssetCategory.InternalBundleAsset;
                }
                else
                {
                    //否则视为内置原生资源加载
                    return AssetCategory.InternalRawAsset;
                }
            }
            else
            {
                //资源名不以Assets/开头 视为外置原生资源加载
                return AssetCategory.ExternalRawAsset;
            }
        }
```

（注意：因为编辑器资源模式下无法准确判断以**Assets/**开头的路径加载资源时，是要加载**内置资源包资源**还是加载**内置原生资源**，所以只能规定当加载类型为`UnityEngine.Object`及其派生类型或`object`类型时视为内置资源包资源加载，不过这并不影响最终加载结果）



### 非编辑器资源模式

```csharp
		/// <summary>
        /// 获取资源类别
        /// </summary>
        public static AssetCategory GetAssetCategory(string assetName)
        {
            if (!assetName.StartsWith("Assets/") && !assetName.StartsWith("Packages/"))
            {
                //资源名不以Assets/ 和 Packages/开头 是外置原生资源
                return AssetCategory.ExternalRawAsset;
            }

            AssetRuntimeInfo assetRuntimeInfo = CatAssetDatabase.GetAssetRuntimeInfo(assetName);
            if (assetRuntimeInfo == null)
            {
                Debug.LogError($"GetAssetCategory调用失败，资源{assetName}的AssetRuntimeInfo为空");
                return default;
            }

            if (assetRuntimeInfo.BundleManifest.IsRaw)
            {
                //内置原生资源
                return AssetCategory.InternalRawAsset;
            }

            //内置资源包资源
            return AssetCategory.InternalBundleAsset;
        }
```



## 资源的加载

在得到资源类别后就可以进行后续的加载行为了



### 编辑器资源模式

```csharp
AssetCategory category;
#if UNITY_EDITOR
            if (IsEditorMode)
            {
                category = Util.GetAssetCategoryWithEditorMode(assetName, typeof(T));
                
                object asset;
                try
                {
                    if (category == AssetCategory.InternalBundleAsset)
                    {
                        //加载资源包资源
                        Type assetType = typeof(T);
                        if (assetType == typeof(object))
                        {
                            assetType = typeof(Object);
                        }
                        asset = UnityEditor.AssetDatabase.LoadAssetAtPath(assetName,assetType);
                    }
                    else
                    {   
                        //加载原生资源
                        if (category == AssetCategory.ExternalRawAsset)
                        {
                            assetName = Util.GetReadWritePath(assetName);
                        }
                    
                        asset = File.ReadAllBytes(assetName);
                    }
                }
                catch (Exception e)
                {
                    callback?.Invoke(false, default,default, userdata);
                    throw;
                }
                
                LoadAssetResult result = new LoadAssetResult(asset, category);
                callback?.Invoke(true, result.GetAsset<T>(),result, userdata);
                return default;
            }
#endif
```



无论加载何种类别资源，最终都被封装进了**资源加载结果（LoadAssetResult）**中，并通过`result.GetAsset<T>`接口将其回调给使用者



#### 资源加载结果

**资源加载结果（LoadAssetResult）**是CatAsset对3种类别资源的统一封装，其代码如下：

```csharp
 	/// <summary>
    /// 资源加载结果
    /// </summary>
    public struct LoadAssetResult
    {
        /// <summary>
        /// 已加载的原始资源实例
        /// </summary>
        private object asset;

        /// <summary>
        /// 资源类别
        /// </summary>
        public AssetCategory Category { get; }

        public LoadAssetResult(object asset, AssetCategory category)
        {
            this.asset = asset;
            Category = category;
        }

        /// <summary>
        /// 获取已加载的原始资源实例
        /// </summary>
        public object GetAsset()
        {
            return asset;
        }

        /// <summary>
        /// 获取已加载的指定类型资源实例
        /// </summary>
        public T GetAsset<T>()
        {
            if (asset == null)
            {
                return default;
            }

            Type type = typeof(T);
            
            if (type == typeof(object))
            {
                return (T)asset;
            }
            
            switch (Category)
            {
                case AssetCategory.InternalBundleAsset:
                    if (typeof(UnityEngine.Object).IsAssignableFrom(type))
                    {
                        return (T) asset;
                    }
                    else
                    {
                        Debug.LogError($"LoadAssetResult.GetAsset<T>调用失败，资源类别为{Category}，但是T为{type}");
                        return default;
                    }
                
                case AssetCategory.InternalRawAsset:
                case AssetCategory.ExternalRawAsset:

                    if (type == typeof(byte[]))
                    {
                        return (T)asset;
                    }

                    CustomRawAssetConverter converter = CatAssetManager.GetCustomRawAssetConverter(type);
                    if (converter == null)
                    {
                        Debug.LogError($"LoadAssetResult.GetAsset<T>调用失败，没有注册类型{type}的CustomRawAssetConverter");
                        return default;
                    }

                    object convertedAsset = converter((byte[])asset);
                    return (T) convertedAsset;

            }

            
            return default;
        }
    }
```



`LoadAssetResult`的主要功能就是在调用`GetAsset<T>`时根据资源类别和指定的类型进行不同处理：

1. 如果指定类型为`object`，直接返回原始资源实例
2. 如果资源类别为**内置资源包资源**，并且指定类型为`UnityEngine.Object`及其派生类型，则按指定类型返回原始资源实例，否则报错（因为资源包资源只能以`UnityEngine.Object`及其派生类型加载）
3. 如果资源类别为**内置/外置原生资源**，并且指定类型为`byte[]`，直接返回原始资源实例（因为原生资源都是按照`byte[]`加载的），否则使用已注册的**自定义原生资源转换器（CustomRawAssetConverter）**将`byte[]`转换为指定类型并返回



#### 自定义原生资源转换器

想要统一对3种类别资源的使用，重点就在于**统一资源包资源与原生资源的使用**，在加载调用代码保持不变的情况下，即使所加载的资源从资源包资源变成了内置/外置原生资源，也能保证后续逻辑的正常运行

比如：

```csharp
string assetName = ???;
CatAssetManager.LoadAsset<Sprite>(assetName, null, callback);
```

CatAsset所保证的即是无论上述代码中的`assetName`表示一个内置资源包资源，还是表示一个内置/外置原生资源，`callback`中的逻辑都无需关心这件事，而只需要处理加载得到的`Sprite`对象



想要做到这点就需要使用**自定义原生资源转换器（CustomRawAssetConverter）**，其是一个委托类型，将`byte[]`转换为`object`，定义如下：

```csharp
/// <summary>
/// 自定义原生资源转换方法的原型
/// </summary>
public delegate object CustomRawAssetConverter(byte[] bytes);
```



CatAsset默认提供了对`Texture2D`、`Sprite`、`TextAsset`的转换器：

```csharp
static CatAssetManager()
{
    RegisterCustomRawAssetConverter(typeof(Texture2D),(bytes =>
    {
        Texture2D texture2D = new Texture2D(0, 0);
        texture2D.LoadImage(bytes);
        return texture2D;
    }));
    
    RegisterCustomRawAssetConverter(typeof(Sprite),(bytes =>
    {
        Texture2D texture2D = new Texture2D(0, 0);
        texture2D.LoadImage(bytes);
        Sprite sp = Sprite.Create(texture2D, new Rect(0, 0, texture2D.width, texture2D.height), Vector2.zero);
        return sp;
    }));
    
    RegisterCustomRawAssetConverter(typeof(TextAsset),(bytes =>
    {
        string text = Encoding.UTF8.GetString(bytes);
        TextAsset textAsset = new TextAsset(text);
        return textAsset;
    }));
    
}
```



### 非编辑器资源模式

在非编辑器资源模式下会根据资源类别使用不同的**Task**处理加载任务

```csharp
category = Util.GetAssetCategory(assetName);
if (category == AssetCategory.ExternalRawAsset)
{
    CatAssetDatabase.TryCreateExternalRawAssetRuntimeInfo(assetName);
}

switch (category)
{
    case AssetCategory.None:
        callback?.Invoke(false, default,default, userdata);
        return default;

    case AssetCategory.InternalBundleAsset:
        //加载内置资源包资源
        LoadBundleAssetTask<T> loadBundleAssetTask = LoadBundleAssetTask<T>.Create(loadTaskRunner, assetName, userdata, callback);
        loadTaskRunner.AddTask(loadBundleAssetTask, priority);
        return loadBundleAssetTask.GUID;
    
    
    case AssetCategory.InternalRawAsset:
    case AssetCategory.ExternalRawAsset:
        //加载原生资源
        LoadRawAssetTask<T> loadRawAssetTask = LoadRawAssetTask<T>.Create(loadTaskRunner,assetName,category,userdata,callback);
        loadTaskRunner.AddTask(loadRawAssetTask, priority);

        return loadRawAssetTask.GUID;
}
```

在加载**外置原生资源**时，会尝试为此外置原生资源创建对应的清单信息和运行时信息，以进行统一管理



#### 资源包资源加载任务

**资源包资源加载任务(LoadBundleAssetTask)**是所有`Task`中最为复杂的，其将整个加载过程分为了6个阶段进行处理：

1. **BundleLoading（资源包加载中）**
2. **BundleLoaded（资源包加载结束）**
3. **DependenciesLoading（依赖资源加载中）**
4. **DependenciesLoaded（依赖资源加载结束）**
5. **AssetLoading（资源加载中）**
6. **AssetLoaded（资源加载结束）**



```csharp
public override void Update()
{
    switch (loadBundleAssetState)
    {
        case LoadBundleAssetState.BundleLoading:
            //1.资源包加载中
            CheckStateWithBundleLoading();
            break;
        
        case LoadBundleAssetState.BundleLoaded:
            //2.资源包加载结束，开始加载依赖资源
            CheckStateWithBundleLoaded();
            break;
        
        case LoadBundleAssetState.DependenciesLoading:
            //3.依赖资源加载中
            CheckStateWithDependenciesLoading();
            break;
        
        case LoadBundleAssetState.DependenciesLoaded:
            //4.依赖资源加载结束，开始加载主资源
            CheckStateWithDependenciesLoaded();
            break;
        
        case LoadBundleAssetState.AssetLoading:
            //5.检查主资源是否加载结束
            CheckStateWithAssetLoading();
            break;
        
        case LoadBundleAssetState.AssetLoaded:
            //6.主资源加载结束，检查是否加载成功
            CheckStateWithAssetLoaded();
            break;
    }
}
```



#### 原生资源加载任务

无论是内置还是外置的原生资源，都统一通过**原生资源加载任务（LoadRawAssetTask）**进行处理

由于原生资源无需处理资源包与依赖资源的加载，所以实现也比较简单，只被划分为了2个阶段：

1. **Loading（资源加载中）**
2. **Loaded（资源加载结束）**

```csharp
public override void Update()
{
    switch (loadRawAssetState)
    {
        case LoadRawAssetState.Loading:
            //加载中
            CheckStateWithLoading();
            break;
        
        case LoadRawAssetState.Loaded:
            //加载结束
            CheckStateWithLoaded();
            break;
    }

}
```



# 资源卸载

资源的卸载需要调用`UnloadAsset`接口传入原始资源实例，并且**卸载要与加载成对的调用**，资源才能被正确卸载



## 引用计数规则



CatAsset通过引用计数来管理资源的卸载，其计数规则如下：

- 每次调用`LoadAsset`或`UnloadAsset`时，将目标资源的引用计数+1或-1
- 当1个资源的引用计数从0变为1或从1变为0时，其直接依赖的资源的引用计数+1或-1



举例来说，假设有**资源A依赖B，B依赖C**，那么调用1次`LoadAsset(A)`后的引用计数状态为：

![](	https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithRuntime/CatAssetDevSummaryWithRuntime_02.png)



再调用2次`LoadAsset(A)`

![](	https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithRuntime/CatAssetDevSummaryWithRuntime_03.png)



最后调用1次`LoadAsset(B)`

![](	https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithRuntime/CatAssetDevSummaryWithRuntime_04.png)





现在开始按照加载调用的次数来调用卸载

调用3次`UnloadAsset(A)`

![](	https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithRuntime/CatAssetDevSummaryWithRuntime_05.png)

由于A的引用计数变为了0，导致B的引用计数被-1



再调用1次`UnloadAsset(B)`

![](	https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithRuntime/CatAssetDevSummaryWithRuntime_06.png)

由于B的引用计数变为了0，导致C的引用计数被-1，此时A、B、C的引用计数都正确归0了



### 为什么不在每次加载或卸载资源时，都增加或减少目标资源所有依赖资源的引用计数？

这样做也是可以的

目前的计数规则方案采取了**【 主资源通过依赖加载，只对其直接依赖资源最多贡献1个引用计数】**的原则

目的在于可以通过将1个资源的引用计数减去它被依赖加载的次数，得到它被主动加载的次数，从而方便查出一些因为使用者没有成对调用加载/卸载接口导致的资源无法被卸载的问题

用上面的例子来说，B的引用计数为2，因为被依赖加载的次数为1，两者相减就知道了B被主动加载的次数为1



## 资源包资源的卸载

每当资源包资源的引用计数从0变为1或从1变为0时，就会从此资源所在资源包的**使用中资源记录（UsedAssets）**添加或删除

而当资源包的`UsedAssets`为空时，就会通过**资源包卸载任务（UnloadBundleTask）**开始卸载倒计时

在倒计时过程中，如果`UsedAssets`不为空，会马上结束`Task`的运行，反之则会在倒计时结束后，通过`AssetBundle.Unload(true)`来将资源包及其中已加载的资源**真正的从内存中删除**



## 原生资源的卸载

对于原生资源而言，由于加载的只是`byte[]`对象，所以卸载也只是将缓存的`byte[]`对象引用置空而已



# 资源更新

## 资源版本检查

要进行资源的更新，需要先通过读取资源清单文件以进行版本检查

CatAsset通过检查**只读区、读写区、远端**三方的资源清单进行版本对比

```csharp
/// <summary>
/// 检查版本
/// </summary>
public static void CheckVersion(OnVersionChecked callback)
{
    if (isChecking)
    {
        return;
    }
    isChecking = true;
    
    onVersionChecked = callback;
    
    //进行只读区 读写区 远端三方的资源清单检查
    string readOnlyManifestPath = Util.GetReadOnlyPath(Util.ManifestFileName);
    string readWriteManifestPath = Util.GetReadWritePath(Util.ManifestFileName);
    string remoteManifestPath = Util.GetRemotePath(Util.ManifestFileName);
    
    CatAssetManager.CheckUpdatableManifest(readOnlyManifestPath,CheckReadOnlyManifest);
    CatAssetManager.CheckUpdatableManifest(readWriteManifestPath,CheckReadWriteManifest);
    CatAssetManager.CheckUpdatableManifest(remoteManifestPath, CheckRemoteManifest);
    
}
```



在三方资源清单都读取到后，会为资源清单里记录的每一条资源清单信息建立对应的**检查信息（CheckInfo）**，并将资源清单信息赋值到此`CheckInfo`中

以检查只读区资源清单的回调方法为例：

```csharp
/// <summary>
/// 检查只读区资源清单
/// </summary>
private static void CheckReadOnlyManifest(bool success, UnityWebRequest uwr, object userdata)
{
    if (!success)
    {
        isReadOnlyLoaded = true;
        RefreshCheckInfos();
        return;
    }
    
    CatAssetManifest manifest = JsonParser.ParseJson<CatAssetManifest>(uwr.downloadHandler.text);

    foreach (BundleManifestInfo item in manifest.Bundles)
    {
        CheckInfo checkInfo = GetOrAddCheckInfo(item.RelativePath);
        checkInfo.ReadOnlyInfo = item;
    }

    isReadOnlyLoaded = true;
    RefreshCheckInfos();
    
}
```



而在三方资源清单都读取完毕后，就会开始刷新`CheckInfo`的**版本检查状态（CheckState）**，然后根据`CheckState`进行后续处理



### 版本检查状态

CatAsset中定义了4种版本检查状态：

1. **NeedUpdate（需要更新）**
2. **InReadWrite（最新版本存在于读写区）**
3. **InReadOnly（最新版本存在于只读区）**
4. **Disuse（已废弃）**



其计算规则如下：

1. 如果此资源包不存在远端信息，则`State`为`Disuse`，并需要删除读写区那份
2. 如果此资源包存在只读区信息，且和远端信息一致，则`State`为`InReadOnly`，并需要删除读写区那份
3. 如果此资源包存在读写区信息，且和远端信息一致，则`State`为`InReadWrite`
4. 如果此资源包存在远端信息，但本地不存在，或本地信息与远端不一致，则`State`为`NeedUpdate`，并需要删除读写区那份



具体代码如下：

```csharp
 	/// <summary>
    /// 版本检查信息
    /// </summary>
    public class CheckInfo
    {
        /// <summary>
        /// 资源包名
        /// </summary>
        public string Name;
        
        /// <summary>
        /// 版本检查状态
        /// </summary>
        public CheckState State;
        
        /// <summary>
        /// 是否需要删除此资源包存在于读写区的文件
        /// </summary>
        public bool NeedRemove;
        
        //此资源包的三方资源清单信息
        public BundleManifestInfo ReadOnlyInfo;
        public BundleManifestInfo ReadWriteInfo;
        public BundleManifestInfo RemoteInfo;
        
        public CheckInfo(string name)
        {
            Name = name;
        }

        /// <summary>
        /// 刷新资源版本检查状态
        /// </summary>
        public void RefreshState()
        {
            if (RemoteInfo == null)
            {
                //此资源包不存在远端 需要删掉读写区那份（如果存在）
                State = CheckState.Disuse;
                NeedRemove = ReadWriteInfo != null;
                return;
            }

            if (ReadOnlyInfo != null && ReadOnlyInfo.Equals(RemoteInfo))
            {
                //此资源包最新版本存在于只读区 需要删掉读写区那份（如果存在）
                State = CheckState.InReadOnly;
                NeedRemove = ReadWriteInfo != null;
                return;
            }

            if (ReadWriteInfo != null && ReadWriteInfo.Equals(RemoteInfo))
            {
                //此资源包最新版本存在于读写区
                State = CheckState.InReadWrite;
                NeedRemove = false;
                return;
            }
            
            //此资源包存在于远端，但本地不是最新版本或本地不存在，需要删掉读写区那份（如果存在）并更新
            State = CheckState.NeedUpdate;
            NeedRemove = ReadWriteInfo != null;
        }
    }
```



## 刷新资源组与更新器信息

在计算过CheckInfo的CheckState后，会根据CheckState刷新此资源的资源组信息（GroupInfo）及此资源的资源组更新器（GroupUpdater）



规则如下：

1. 对于`State`不为`Disuse`的资源，会添加到资源组的远端资源包信息中
2. 如果`State`为`NeedUpdate`，会添加到对应的资源组更新器中
3. 如果`State`为`InReadWrite`或`InReadOnly`，会添加到资源组的本地资源包信息中
4. 如果此资源需要删除，则会从读写区删除，并重新生成读写区资源清单文件



具体代码如下：

```csharp
/// <summary>
/// 刷新资源检查信息
/// </summary>
private static void RefreshCheckInfos()
{
    if (!isReadOnlyLoaded || !isReadWriteLoaded || !isRemoteLoaded)
    {
        //三方资源清单未加载完毕
        return;
    }

    //需要更新的所有资源包的数量与长度
    int totalCount = 0;
    long totalLength = 0;

    bool needGenerateReadWriteManifest = false;

    foreach (KeyValuePair<string,CheckInfo> pair in checkInfoDict)
    {
        CheckInfo checkInfo = pair.Value;
        checkInfo.RefreshState();

        if (checkInfo.State != CheckState.Disuse)
        {
            //添加资源组的远端资源包信息
            GroupInfo groupInfo = CatAssetDatabase.GetOrAddGroupInfo(checkInfo.RemoteInfo.Group);
            groupInfo.AddRemoteBundle(checkInfo.RemoteInfo.RelativePath);
            groupInfo.RemoteCount++;
            groupInfo.RemoteLength += checkInfo.RemoteInfo.Length;
        }
        
        switch (checkInfo.State)
        {

            case CheckState.NeedUpdate:
                //需要更新
                totalCount++;
                totalLength += checkInfo.RemoteInfo.Length;

                GroupUpdater groupUpdater = CatAssetUpdater.GetOrAddGroupUpdater(checkInfo.RemoteInfo.Group);
                groupUpdater.AddUpdateBundle(checkInfo.RemoteInfo);
                groupUpdater.TotalCount++;
                groupUpdater.TotalLength += checkInfo.RemoteInfo.Length;
                
                break;
            
            case CheckState.InReadWrite:
                //不需要更新 最新版本存在于读写区
                
                GroupInfo groupInfo = CatAssetDatabase.GetOrAddGroupInfo(checkInfo.ReadWriteInfo.Group);
                groupInfo.AddLocalBundle(checkInfo.ReadWriteInfo.RelativePath);
                groupInfo.LocalCount++;
                groupInfo.LocalLength += checkInfo.ReadWriteInfo.Length;
                
                CatAssetDatabase.InitRuntimeInfo(checkInfo.ReadWriteInfo,true);
                CatAssetUpdater.AddReadWriteManifestInfo(checkInfo.ReadWriteInfo);
                
                break;
            
            case CheckState.InReadOnly:
                //不需要更新 最新版本存在于只读区

                groupInfo = CatAssetDatabase.GetOrAddGroupInfo(checkInfo.ReadOnlyInfo.Group);
                groupInfo.AddLocalBundle(checkInfo.ReadOnlyInfo.RelativePath);
                groupInfo.LocalCount++;
                groupInfo.LocalLength += checkInfo.ReadOnlyInfo.Length;
                
                CatAssetDatabase.InitRuntimeInfo(checkInfo.ReadOnlyInfo,false);
                break;
        }

        if (checkInfo.NeedRemove)
        {
            //需要删除读写区的那份
            Debug.Log($"删除读写区资源:{checkInfo.Name}");
            string path = Util.GetReadWritePath(checkInfo.Name);
            File.Delete(path);

            needGenerateReadWriteManifest = true;
        }
    }

    if (needGenerateReadWriteManifest)
    {
        //删除过读写区资源 需要重新生成读写区资源清单
        CatAssetUpdater.GenerateReadWriteManifest();
    }
    
    //调用版本检查完毕回调
    VersionCheckResult result = new VersionCheckResult(string.Empty,totalCount, totalLength);
    onVersionChecked?.Invoke(result);
        
    Clear();
}
```



## 资源的更新

资源更新是以资源组为单位进行的，在启动**资源组更新器（GroupUpdater）**后，会根据此资源组需要更新的资源创建**资源包下载任务（DownloadBundleTask）**

```csharp
/// <summary>
/// 更新资源组
/// </summary>
internal void UpdateGroup(OnBundleUpdated callback)
{
    if (State != GroupUpdaterState.Free)
    {
        //非空闲状态 不处理
        return;
    }
    
    State = GroupUpdaterState.Running;
    foreach (BundleManifestInfo info in updateBundles)
    {
        //创建下载文件的任务
        string localFilePath = Util.GetReadWritePath(info.RelativePath);
        string downloadUri = Path.Combine(CatAssetUpdater.UpdateUriPrefix, info.RelativePath);
        CatAssetManager.DownloadBundle(this,info,downloadUri,localFilePath,onBundleDownloaded);
    }
    
    onBundleUpdated = callback;
}
```

```csharp
/// <summary>
/// 下载资源包
/// </summary>
internal static void DownloadBundle(GroupUpdater groupUpdater, BundleManifestInfo info,string downloadUri,string localFilePath,DownloadBundleCallback callback)
{
    DownloadBundleTask task =
        DownloadBundleTask.Create(downloadTaskRunner, downloadUri, info, groupUpdater, downloadUri, localFilePath, callback);
    downloadTaskRunner.AddTask(task,TaskPriority.Height);
}
```



### 资源包下载任务

**资源包下载任务（DownloadBundleTask）**通过`UnityWebRequest`和`DownloadHandlerFile`实现了低GC的文件下载，并且在启动时会通过检查本地已下载文件的字节长度进行断点续传操作

```csharp
public override void Run()
{
    if (groupUpdater.State == GroupUpdaterState.Paused)
    {
        //处理下载暂停 暂停只对还未开始执行的下载任务有效
        return;
    }

    //开始位置
    int startLength = 0;
    
    //先检查本地是否已存在临时下载文件
    if (File.Exists(localTempFilePath))
    {
        using (FileStream fs = File.OpenWrite(localTempFilePath))
        {
            //检查已下载的字节数
            fs.Seek(0, SeekOrigin.End);
            startLength = (int)fs.Length;
        }
    }
    
    UnityWebRequest uwr = new UnityWebRequest(downloadUri);
    if (startLength > 0)
    {
        //处理断点续传
        uwr.SetRequestHeader("Range", $"bytes={{{startLength}}}-");
    }
    uwr.downloadHandler = new DownloadHandlerFile(localTempFilePath, startLength > 0);
    op = uwr.SendWebRequest();
}
```



### 重试与校验

在下载失败后会尝试重新下载

如果下载成功了会先校验文件长度，若长度相同再校验MD5值，校验失败则会删除下载文件并尝试重新下载

校验都通过后会认为下载成功，并回调结果给使用者

```csharp
public override void Update()
{
    if (op == null)
    {
        //被暂停了
        State = TaskState.Free;
        return;
    }
    
    if (!op.webRequest.isDone)
    {
        //下载中
        State = TaskState.Running;
        return;
    }

    if (op.webRequest.result == UnityWebRequest.Result.ConnectionError || op.webRequest.result == UnityWebRequest.Result.ProtocolError)
    {
        //下载失败 重试
        if (RetryDownload())
        {
            Debug.LogError($"下载失败准备重试：{Name},错误信息：{op.webRequest.error}，当前重试次数：{retriedCount}");
        }
        else
        {
            //重试次数达到上限 通知失败
            Debug.LogError($"重试次数达到上限：{Name},错误信息：{op.webRequest.error}，当前重试次数：{retriedCount}");
            State = TaskState.Finished;
            onFinished?.Invoke(false ,bundleManifestInfo);
        }
        return;
    }
    
    //下载成功 开始校验
    //先对比文件长度
    FileInfo fi = new FileInfo(localTempFilePath);
    bool isVerify = fi.Length == bundleManifestInfo.Length;
    if (isVerify)
    {
        //文件长度对得上 再校验MD5
        string md5 = Util.GetFileMD5(localTempFilePath);
        isVerify = md5 == bundleManifestInfo.MD5;
    }

    if (!isVerify)
    {
        //校验失败 删除临时下载文件 尝试重新下载
        File.Delete(localTempFilePath);
        
        if (RetryDownload())
        {
            Debug.LogError($"校验失败准备重试：{Name}，当前重试次数：{retriedCount}");
        }
        else
        {
            //重试次数达到上限 通知失败
            Debug.LogError($"重试次数达到上限：{Name}，当前重试次数：{retriedCount}");
            State = TaskState.Finished;
            onFinished?.Invoke(false ,bundleManifestInfo);
              
        }
        
        return;
    }
    
    
    //校验成功
    State = TaskState.Finished;
            
    //将临时下载文件覆盖到正式文件上
    if (File.Exists(localFilePath))
    {
        File.Delete(localFilePath);
    }
    File.Move(localTempFilePath, localFilePath);
    onFinished?.Invoke(true,bundleManifestInfo);
}
```



### 重新生成读写区资源清单

在`GroupUpdater`的`OnBundleDownloaded`回调被调用时，会刷新自身保存的已下载资源信息，并且在所有资源下载完毕或已下载字节数达到要求后，重新生成读写区资源清单

而如果都成功更新完了，就会将自身移除掉，否则将会等待下一次启动更新器

```csharp
/// <summary>
/// 资源包下载完毕的回调
/// </summary>
private void OnBundleDownloaded(bool success, BundleManifestInfo info)
{
    callbackCount++;
    if (callbackCount == TotalCount)
    {
        //所有需要下载的资源包都回调过 就将状态改为Free
        //若此时有下载失败的资源包，导致UpdatedCount < TotalCount，则可通过重新启动此Updater来进行下载
        State = GroupUpdaterState.Free;
    }

    BundleUpdateResult result;
    if (!success)
    {
        Debug.LogError($"更新{info.RelativePath}失败");
        result = new BundleUpdateResult(false,info.RelativePath,this);
        onBundleUpdated?.Invoke(result);
        return;
    }

    updateBundles.Remove(info);
    
    //刷新已下载资源信息
    UpdatedCount++;
    UpdatedLength += info.Length;
    deltaUpdatedLength += info.Length;
    
    //将下载好的资源包的运行时信息添加到CatAssetDatabase中
    CatAssetDatabase.InitRuntimeInfo(info, true);
    
    //刷新读写区资源信息列表
    CatAssetUpdater.AddReadWriteManifestInfo(info);
    
    //刷新资源组本地资源信息
    GroupInfo groupInfo = CatAssetDatabase.GetOrAddGroupInfo(info.Group);
    groupInfo.AddLocalBundle(info.RelativePath);
    groupInfo.LocalCount++;
    groupInfo.LocalLength += info.Length;
    
    bool allDownloaded = UpdatedCount >= TotalCount;
    if (allDownloaded || deltaUpdatedLength >= generateManifestLength)
    {
        //资源下载完毕 或者已下载字节数达到要求 就重新生成一次读写区资源清单
        deltaUpdatedLength = 0;
        CatAssetUpdater.GenerateReadWriteManifest();
    }

    if (allDownloaded)
    {
        //该组资源都更新完毕，可以删掉updater了
        CatAssetUpdater.RemoveGroupUpdater(GroupName);
    }
    
    //调用外部回调
    result = new BundleUpdateResult(true,info.RelativePath,this);
    onBundleUpdated?.Invoke(result);
}
```

