---
title: CatAsset使用教程
date: 2022-08-30 15:40:17
tags: 资源管理
categories: 程序
---

# 框架版本

此教程文档基于[CatAsset](https://github.com/CatImmortal/CatAsset)在Github上main主干的最新版本编写



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

只需要通过自定义类实现`IBundleBuildRule`接口，并将其放置于**CatAsset/Editor/Rule**文件夹下即可



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

点击**选中**按钮可在Unity内选中此资源



### 循环依赖检测

点击**检测资源循环依赖**与**检测资源包循环依赖**即可在编辑器下检查相关循环依赖问题



### 冗余分析

在勾选**冗余分析**的情况下，点击**刷新**，CatAsset会自动将冗余资源按照其关联划分为多个冗余资源包



## 设定构建配置

在预览完成后，切换分页到**构建配置**，可进行相关设置

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_08.png)



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



## 运行模式

CatAsset提供了2种运行模式

- PackageOnly（仅使用安装包内的资源，单机模式）
- Updatable（可更新模式）

*后续将以单机模式为例*



## 资源清单检查

单机模式下，在加载资源前需要先进行资源清单检查，才能正确初始化`CatAssetManager`

```csharp
CatAssetManager.CheckPackageManifest(Action<bool> callback)
```

回调以bool值为参数，表示是否检查成功



## 编辑器资源模式

若勾选**编辑器资源模式**，即可在不进行资源包构建与资源清单检查的前提下快速运行游戏



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
CatAssetManager.LoadAsset<T>(string assetName, object userdata, LoadAssetCallback<T> callback,TaskPriority priority = TaskPriority.Middle)
```

**assetName**为资源的加载路径，若资源类别为1和2，则是一个**Assets/**开头的路径，若为3，则是1个相对于读写区的路径

**userdata**为调用者的自定义数据，会通过加载回调传递回去

**LoadAssetCallback**为资源加载回调，其定义为

```csharp
 public delegate void LoadAssetCallback<in T>(bool success, T asset,LoadAssetResult result, object userdata);
```

**success**表示此资源是否加载成功

**asset**为加载到的资源，其类型为调用加载接口时传入的泛型类型

**LoadAssetResult**为加载结果，其封装了原始资源实例与`GetAsset`接口

**userdata**为调用加载接口时传入的userdata



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



### 加载结果

`LoadAssetResult`封装了加载结果，包括**原始资源实例**，**资源类别**和用于获取资源实例的`GetAsset`及`GetAsset<T>`，通常用于在加载了被转换过的原生资源后，获取原始资源实例以进行卸载操作

`GetAsset<T>`会根据传入的`T`类型以及**资源类别**来判断应该返回什么

其判断规则如下：

1. 当`T`为`object`类型时会直接返回原始资源实例
2. 当资源类别为原生资源时，如果`T`为`byte[]`类型，会直接返回原始资源实例，否则调用转换器将原始资源实例转换为对应类型的资源并返回
3. 当资源类别为资源包资源时，如果`T`为`UnityEngine.Objec`t及其派生类型，会直接返回原始资源实例，否则会报错

（`GetAsset`等价于`GetAsset<object>`）



### 批量资源加载

CatAsset支持对一组资源进行批量加载的操作，其接口为：

```csharp
CatAssetManager.BatchLoadAsset(List<string> assetNames, object userdata, BatchLoadAssetCallback callback, TaskPriority priority = TaskPriority.Middle)
```



*支持对3种类别资源的混合批量加载*



### 取消加载

调用加载接口后会返回一个int值，其表示加载任务的**GUID**，可以通过此**GUID**取消加载任务，取消加载的接口为：

```csharp
CatAssetManager.CancelTask(int guid)
```



## 资源卸载

CatAsset采用**基于引用计数的资源卸载**方式，因此在加载资源后需要有与其成对调用的卸载调用才能正确将资源卸载掉，卸载接口为：

```csharp
CatAssetManager.UnloadAsset(object asset)
```



*asset需要为加载的原始资源实例，否则无法正确卸载*



## 场景管理

对于场景的加载与卸载使用与非场景资源不同的接口，分别为：

```csharp
CatAssetManager.LoadScene(string sceneName, object userdata, LoadSceneCallback callback,
            TaskPriority priority = TaskPriority.Middle)
```

```csharp
CatAssetManager.UnloadScene(Scene scene)
```



*同样的，对于场景的加载也可以通过GUID进行取消*



## 资源生命周期绑定

考虑到手动调用卸载可能产生的不便性与易错漏性，可通过资源绑定接口将资源的生命周期与游戏物体/场景的生命周期进行绑定以实现自动卸载

绑定后会在游戏物体销毁时/场景卸载时自动对被绑定的资源调用卸载接口

资源生命周期绑定接口为：

```csharp
CatAssetManager.BindToGameObject(GameObject target,object asset)
CatAssetManager.BindToScene(Scene scene,object asset)
```



## 运行时信息窗口

为了方便进行资源加载/卸载相关的调试，CatAsset提供了用于展示资源运行时信息的窗口，点击上方工具栏**CatAsset/打开运行时信息窗口**即可打开

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_11.png)



### 资源信息

资源信息窗口提供了已加载的资源的相关信息

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_12.png)



#### 文件长度

其中资源包信息条目的文件长度表示资源包文件的实际长度，资源信息条目的长度表示此资源的原始文件长度



#### 只显示主动加载的资源

主动加载：在业务层通过加载接口进行资源加载的行为

依赖加载：框架层自身通过依赖加载进行资源加载的行为



若勾选**只显示主动加载的资源**，则会将通过自动依赖加载出的资源的相关信息隐藏，使得开发者能够更专注于那些被主动加载的资源信息



#### 引用计数

引用计数由主动加载的计数和依赖加载的计数2部分组成

引用计数的增减规则：

1. 每一次对资源的主动加载或主动卸载会+1或-1此资源的引用计数
2. 当资源的引用计数从0增加到1或从1减少为0时，会+1或-1直接依赖资源的引用计数（**也就是说1个资源最多只会对它的所有直接依赖资源贡献1个引用计数**）



#### 引用与依赖

引用资源的定义：若资源A依赖资源B，则称A是B的引用资源

引用资源包的定义：若资源包A中有资源X依赖资源包B中的资源Y，则称A是B的引用资源包



*引用计数 - 引用资源的数量 = 此资源被主动加载的次数*



### 任务信息

在资源包满足卸载条件时，不会立即进行卸载，而是开启一个卸载任务，进行倒计时卸载

此时可以在**任务信息**分页中查看相关的任务

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_13.png)



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

此时可以通过遍历资源组信息列表，获取其对应的更新器来判断其是否有需要更新的资源

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
CatAssetManager.UpdateGroup(string group, OnBundleUpdated callback)
```

`OnBundleUpdated`为资源包更新回调，其定义如下：

```csharp
 public delegate void OnBundleUpdated(BundleUpdateResult result);
```

`BundleUpdateResult`表示资源包更新结果：

```csharp
	/// <summary>
    /// 资源包更新结果
    /// </summary>
    public struct BundleUpdateResult
    {
        /// <summary>
        /// 是否更新成功
        /// </summary>
        public bool Success;
        
        /// <summary>
        /// 资源包相对路径
        /// </summary>
        public string BundleRelativePath;
        
        /// <summary>
        /// 此资源包的资源组更新器
        /// </summary>
        public GroupUpdater Updater;

        //省略...
    }
```

可从`BundleUpdateResult`中获取到`GroupUpdater`，以取得更详细的信息，如资源组名、需要更新的资源包数量/长度、已更新的资源包数量/长度





## 暂停/恢复更新

可通过暂停更新接口暂停/恢复指定资源组更新器的资源更新行为：

```csharp
CatAssetManager.PauseGroupUpdater(string group, bool isPause)
```



## 运行时信息窗口

可在**运行时信息窗口**的**资源组信息**分页和**更新器信息**分页中查看与资源更新有关的信息

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_14.png)

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetGuide/CatAssetGuide_15.png)
