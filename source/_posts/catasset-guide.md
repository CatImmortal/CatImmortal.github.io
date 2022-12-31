---
title: CatAsset使用教程
date: 2022-08-30 15:40:17
tags: 资源管理
categories: 程序
---

# 资源构建

## 指定资源目录

CatAsset基于**资源目录**与**构建规则**以进行批量资源构建，所以构建资源的第一步便是指定资源目录

操作方法为：**右键目录-添加为资源包构建目录**

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_01.png)



点击上方工具栏**CatAsset-打开资源包构建窗口**

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_02.png)



点击**构建目录**页签，即可看到此目录的信息

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_03.png)



## 选择构建规则

**构建规则**决定了此目录下所有资源文件会按照什么样的方式去构建成为资源包

点击`NAssetToOneBundle`处的下拉按钮展开下拉列表，选择**构建规则**

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_04.png)



CatAsset默认提供了4种**构建规则**：

1. `NAssetToNBundle`（将指定目录下所有资源分别构建为一个资源包）
2. `NAssetToNRawBundle`（将指定目录下所有资源分别构建为一个原生资源包）
3. `NAssetToOneBundle`（将指定目录下所有资源构建为一个资源包）
4. `NAssetToOneBundleWithTopDirectory`（将指定目录下所有一级子目录各自使用`NAssetToOneBundle`规则进行构建）



### 如何扩展构建规则？

只需要通过自定义类实现`IBundleBuildRule`接口即可



### 为什么没有NAssetToOneRawBundle？

因为原生资源不使用**AssetBundle**进行构建，原生资源包只是被虚拟出来方便进行统一管理的，实际上并不存在对应的资源包文件，所以也不存在一个原生资源包内有多个原生资源的情况，对于原生资源而言，资源文件即是资源包文件



## 正则筛选

在正则一栏中输入正则表达式，即可仅将**资源路径（以Assets/开头）匹配此表达式**的资源纳入构建范围内



## 资源组

资源组设置用于可更新模式下更新资源，组名相同的资源目录所构建的资源包会被视为同一个资源组，**单机模式下可忽略此项设置**



## 预览资源包

切换分页到**资源包预览**，点击**刷新**即可预览到构建后的资源包的内容

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_05.png)



**bundleres/prefabb.bundle**部分即是此资源包在构建后的，相对于只读区/读写区的路径

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_06.png)



**资源数**表示此资源包内的资源数量

**总长度**表示此资源包内的资源文件长度总和（因此并不表示此资源包在构建后的文件长度，因为在进行预览时并未实际构建出此资源包）



### 预览资源

点击**资源包预览**条目左侧的三角形，即可展开资源列表，查看资源相关信息

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_07.png)

**Assets/BundleRes/PrefabB/B1.prefab**为此资源的加载路径

**长度**即为此资源的文件长度



### 循环依赖检测

点击**检测资源循环依赖**与**检测资源包循环依赖**即可在编辑器下检查相关循环依赖问题



### 冗余分析

CatAsset会自动将冗余资源按照其关联划分为多个冗余资源包



## 设定构建配置

在预览完成后，切换分页到**构建配置**，可进行相关设置

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_08.png)



### 构建设置

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_16.png)

构建设置分为4种：

1. WriteLinkXML：会通过SBP在资源包构建完成后自动生成link.xml文件到输出目录
2. ForceRebuild：强制全量构建
3. AppendMD5：在资源包名上附加MD5值
4. ChunkBasedCompression：使用LZ4进行压缩



### 最终输出目录

资源包在构建完成后的最终输出目录为：**资源包构建输出根目录/资源包构建平台/游戏版本号_资源清单版本号/**，如**AssetBundles/StandaloneWindows/0.1_1**

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_09.png)

*资源清单CatAssetManifest.json也会被写入到此目录下*



### 仅构建原生资源包

当勾选**仅构建原生资源包**时，CatAsset会跳过AssetBundle构建流程，直接将被指定构建的原生资源最新版本与前一个版本的AssetBundle资源合并输出

*此选项适合仅只有原生资源发生变化时勾选*



# 资源的加载与卸载

## 运行前的准备

在运行前，需要先将**CatAssetComponent**挂载至游戏物体上

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_10.png)



或者通过项目中自定义的Manager来进行CatAssetManager.Update的调用



## 运行模式

CatAsset提供了2种运行模式

- PackageOnly（单机模式，仅使用安装包内的资源）
- Updatable（可更新模式）

*后续将以单机模式为例*



## 资源清单检查

单机模式下，在加载资源前需要先进行资源清单检查，才能正确初始化`CatAssetManager`

```csharp
CatAssetManager.CheckPackageManifest(Action<bool> callback)
```

回调以bool值为参数，表示是否检查成功



## 编辑器资源模式

若勾选**编辑器资源模式**，即可在不进行资源包构建的前提下快速运行游戏，此时CheckPackageManifest会直接返回true



## 资源类别

CatAsset支持3种类别的资源：

1. 内置资源包资源（在Unity工程中使用CatAsset构建，从AssetBundle中加载的资源，如Prefab、Scene文件等）
2. 内置原生资源（在Unity工程中使用CatAsset构建，不基于AssetBundle而是直接加载其二进制数据的资源，如DLL、Lua文件等）
3. 外置原生资源（不使用CatAsset构建，直接从读写区加载其二进制数据的资源，如玩家自定义的图片、文本等）



### 导入外部的资源包

可通过将读写区的外部资源包导入`CatAssetManager`，以将其视为内置资源包资源加载（这些被导入的资源包必须是通过CatAsset构建出来的，这样才有清单文件），接口为：

```csharp
CatAssetManager.ImportInternalAsset(string manifestPath,Action<bool> callback,string bundleRelativePathPrefix = null)
```

`manifestPath`为要导入的资源包的资源清单相对于读写区的路径

`callback`为导入完毕的回调

`bundleRelativePathPrefix`为资源包相对于读写区路径的额外前缀，若为null，则加载资源包时的路径为**Application.persistentDataPath/bundleRelativePath**，否则为**Application.persistentDataPath/bundleRelativePathPrefix/bundleRelativePath**，此参数主要用于方便自定义需要导入的资源包的位置



*此功能主要用于实现AssetBundle类型的Mod文件加载功能*



## 加载资源

在资源清单检查完毕后即可进行资源的加载（**所有加载都是异步的**）



3类资源（非场景）统一通过以下接口进行加载：

```csharp
AssetHandler<T> handler = CatAssetManager.LoadAssetAsync<T>(string assetName, CancellationToken token = default,TaskPriority priority = TaskPriority.Low)
```

**assetName**为资源的加载路径，若资源类别为1和2，则是一个**Assets/**开头的路径，若为3，则是1个相对于读写区的路径

**token**用于进行加载的取消操作

**priority**表示此加载任务的优先级



### 资源句柄

调用加载资源的接口后会返回一个表示资源句柄的`AssetHandler<T>`对象，其封装了对被加载的资源的相关操作，如注册此资源加载结束回调，`await handler`以等待加载结束，资源是否加载成功以及最重要的获取被加载的资源、卸载句柄等



可通过注册`handler.OnLoaded`回调或直接`await handler`等待加载完成，然后通过`handler.Success`判断是否加载成功，`handler.Asset`获取到加载到的资源，或是通过`handler.Unload()`卸载句柄

```csharp
CatAssetManager.LoadAssetAsync<TextAsset>("Assets/BundleRes/RawText/rawText1.txt").OnLoaded +=
    handler =>
    {
        if (handler.IsSuccess)
        {
            Debug.Log($"加载原生资源 文本文件:{handler.Asset.text}");
        }

        handler.Unload();
    };
```

```csharp
AssetHandler<TextAsset> handler = await CatAssetManager.LoadAssetAsync<TextAsset>("Assets/BundleRes/RawText/rawText1.txt");
if (handler.IsSuccess)
{
	Debug.Log($"加载原生资源 文本文件:{handler.Asset.text}");
}
handler.Unload();
```



### 句柄中对资源对象的处理

`AssetHandler<T>`封装了资源加载结果，包括**原始资源实例**，**资源类别**和用于获取资源的`AssetObj`,`Asset`及`AssetAs<T>`

`AssetObj`表示原始资源实例

`Asset`表示`AssetObj`通过`AssetAs<T>`转换后的结果

`AssetAs<T>`会根据传入的`T`类型以及**资源类别**来判断应该返回什么

其判断规则如下：

1. 当`T`为`object`类型时会直接返回原始资源实例
2. 当资源类别为原生资源时，如果`T`为`byte[]`类型，会直接返回原始资源实例，否则调用转换器将原始资源实例转换为对应类型的资源并返回
3. 当资源类别为资源包资源时，如果`T`为`UnityEngine.Objec`t及其派生类型，会直接返回原始资源实例，否则会报错



*通常来说直接使用`handler.Asset`即可*



### 原生资源的加载

对于原生资源而言，底层只能加载到`byte[]`数据

而为了能够进行统一的加载接口调用，需要将`byte[]`转换为调用加载接口时指定的类型

如何进行转换则是由自定义原生资源转换器`CustomRawAssetConverter`决定



CatAsset默认提供了3种类型的转换器：

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

若要注册自定义的转换器，需要调用

```csharp
CatAssetManager.RegisterCustomRawAssetConverter(Type type, CustomRawAssetConverter converter)
```



### 批量资源加载

CatAsset支持对一组资源进行批量加载的操作，其接口为：

```csharp
BatchAssetHandler handler = CatAssetManager.BatchLoadAssetAsync(List<string> assetNames,CancellationToken token = default, TaskPriority priority = TaskPriority.Low)
```



*支持对3种类别资源的混合批量加载*

*注意：由于在编辑器资源模式下，对原生资源的加载必须显式指定加载类型为byte[]，否则会被判断为内置资源包资源，而批量加载接口无法显式指定加载类型，所以无法在编辑器资源模式下对原生资源进行批量加载操作*



## 资源卸载

CatAsset采用**基于引用计数的资源卸载**方式，因此在加载资源后需要有与其成对调用的卸载调用才能正确将资源卸载掉，卸载接口为：

```csharp
CatAssetManager.UnloadAsset(AssetHandler handler)
```

或

```csharp
assethandler.Unload()
```



*可通过`BatchAssetHandler.Unload`来进行对批量加载到的资源的统一卸载*



## 卸载句柄

通过`handler.Unload()`会进行对句柄的卸载，其中会根据句柄当前状态进行处理：

1. 若当前句柄无效，则不做处理
2. 若当前资源加载中，则取消加载
3. 若当前资源加载成功，则进行对资源的卸载
4. 若当前资源加载失败，则仅释放句柄



## 场景管理

对于场景的加载与卸载使用与非场景资源不同的接口，分别为：

```csharp
SceneHandler handler = CatAssetManager.LoadSceneAsync(string sceneName,CancellationToken token = default, TaskPriority priority = TaskPriority.Low)
```

```csharp
CatAssetManager.UnloadScene(SceneHandler scene)
```

```
sceneHandler.Unload()
```



## 资源生命周期绑定

考虑到手动调用卸载可能产生的不便性与易错漏性，可通过资源绑定接口将资源的生命周期与游戏物体/场景的生命周期进行绑定以实现自动卸载

绑定后会在游戏物体销毁时/场景卸载时自动对被绑定的资源句柄调用卸载接口

资源生命周期绑定接口为：

```csharp
CatAssetManager.BindToGameObject(GameObject target, IBindableHandler handler)
CatAssetManager.BindToScene(Scene scene, IBindableHandler handler)
```

或

```
scene.BindTo(handler)
gameobject.BindTo(handler)

handler.BindTo(scene)
handler.BindTo(gameObject)
```



# 资源更新

在运行模式为可更新模式时可以进行资源更新的操作



## 设置资源更新Uri前缀

要进行资源更新，首先需要主动设置`CatAssetManager.UpdateUriPrefix`

下载资源文件时会以 **UpdateUriPrefix/BundleRelativePath** 为下载地址



## 检查资源版本

进行资源更新前，需要调用检查资源版本的接口

```csharp
CatAssetManager.CheckVersion(OnVersionChecked onVersionChecked)
```



## 获取更新器信息

在资源版本检查完毕后，CatAsset会创建出需要进行更新的资源组所对应的更新器

此时可以通过`CheckVersion`传递给回调的`VersionCheckResult`的`GroupUpdaters`字段获取到所有需要更新资源的更新器



或者通过遍历资源组信息列表，获取其对应的更新器来判断其是否有需要更新的资源

```csharp
		//遍历所有资源组的更新器
        List<GroupInfo> groups = CatAssetManager.GetAllGroupInfo();
        foreach (GroupInfo groupInfo in groups)
        {
            GroupUpdater updater = CatAssetManager.GetGroupUpdater(groupInfo.GroupName);
            if (updater != null)
            {
                //存在资源组对应的更新器 就说明此资源组有需要更新的资源
                Debug.Log($"{groupInfo.GroupName}组的资源需要更新");
            }
        }
```

*在资源组的所有待更新资源都成功更新完毕后，会删除此资源组对应的更新器*



## 更新资源组

使用更新资源组接口即可开始调用指定资源组的更新器，开始更新资源：

```csharp
CatAssetManager.UpdateGroup(string group, BundleUpdatedCallback callback)
```

`BundleUpdatedCallback`为资源包更新回调，其定义如下：

```csharp
public delegate void BundleUpdatedCallback(BundleUpdateResult result);
```

`BundleUpdateResult`表示资源包更新结果：

```csharp
	/// <summary>
    /// 资源包更新结果
    /// </summary>
    public readonly struct BundleUpdateResult
    {
        /// <summary>
        /// 是否更新成功
        /// </summary>
        public readonly bool Success;

        /// <summary>
        /// 资源包更新器信息
        /// </summary>
        public readonly UpdateInfo UpdateInfo;
        
        /// <summary>
        /// 此资源包的资源组更新器
        /// </summary>
        public readonly GroupUpdater Updater;

        //省略...
    }
```

可从`BundleUpdateResult`中获取到`GroupUpdater`，以取得更详细的信息，如资源组名、需要更新的资源包数量/长度、已更新的资源包数量/长度





## 暂停/恢复更新

可通过相关接口暂停/恢复指定资源组更新器的资源更新行为：

```csharp
CatAssetManager.PauseGroupUpdater(string group)
CatAssetManager.ResumeGroupUpdater(string group)
```



# 调试分析器

为了方便进行资源框架相关的调试，CatAsset提供了用于展示运行时信息的调试分析器窗口，点击上方工具栏**CatAsset/打开调试分析器窗口**即可打开



调试分析器可用于进行**真机调试**，只需要在构建安装包时勾选**developmentBuild**和**Autoconnect Profiler**即可



## 通用说明

通过点击**采样**按钮，即可获取当前帧的分析器信息

通过滑动Slider或直接输入帧号，可切换到不同帧对应的分析器信息



## 资源包信息

资源包信息窗口提供了**尚在内存中的资源包**的相关信息

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_12.png)



### 文件长度

其中资源包信息条目的文件长度表示资源包文件的实际长度，资源信息条目的长度表示此资源的原始文件长度



### 只显示主动加载的资源

主动加载：在业务层通过加载接口进行资源加载的行为

依赖加载：框架层自身通过依赖加载进行资源加载的行为



若勾选**只显示主动加载的资源**，则会将通过自动依赖加载出的资源的相关信息隐藏，使得开发者能够更专注于那些被主动加载的资源信息



### 引用计数

引用计数由主动加载的计数和依赖加载的计数2部分组成

引用计数的增减规则：

1. 每一次对资源的主动加载或主动卸载会+1或-1此资源的引用计数
2. 当资源的引用计数从0增加到1或从1减少为0时，会+1或-1直接依赖资源的引用计数（**也就是说1个资源最多只会对它的所有直接依赖资源贡献1个引用计数**）



### 上游与下游

在依赖链中，若A依赖B，B依赖C，则称

C是B的上游节点，B是A的上游节点

A是B的下游节点，B是C的下游节点

通过点击**查看依赖关系图**按钮，即可查看与当前资源包/资源有直接/间接上下游关系的依赖关系图

*引用计数 - 下游节点的数量 = 此资源被主动加载的次数*



## 资源信息

资源信息窗口提供了**尚在内存中的资源**的相关信息，各列含义大致与资源包信息窗口的一致



## 任务信息

在CatAsset中，如加载、卸载、更新等异步操作都是由内置的任务系统实现的，可通过任务信息窗口查看当前进行中的任务

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_13.png)



## 资源组信息

此窗口显示与资源组相关的信息

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_14.png)



## 更新器信息

此窗口显示资源更新相关的信息

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_15.png)
