---
title: Unity资源框架设计中不同级别依赖管理的对比
date: 2021-10-29 20:47:28
tags: 资源管理
categories: 程序
description: Bundle级别与Asset级别的依赖管理的对比
top_img: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/DependencyManageCompare/DependencyManageCompare_4.png
cover: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/DependencyManageCompare/DependencyManageCompare_4.png
---

# 前言

在Unity资源框架的设计中，资源的依赖管理可谓是重中之重，框架设计者往往会使用一张资源清单来记录资源间的依赖关系，而依赖管理的粒度可以分为2个级别：

1. **Bundle与Bundle间的依赖**
2. **Asset与Asset间的依赖**

本文将对这两个不同级别的依赖管理粒度进行比较分析，最终给出一个较好的可行方案



# Bundle级的依赖管理

Bundle级的依赖管理是Unity构建管线中所默认使用的依赖管理，在调用`BuildPipeline.BuildAssetBundles`后会返回一个 `AssetBundleManifest`对象，调用它的`GetAllDependencies(string assetBundleName)`会返回指定名称的AssetBundle所依赖的所有AssetBundle名称。



![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/DependencyManageCompare/DependencyManageCompare_1.png)

如上图所示，A包的资源a1依赖B包的资源b1，a2依赖c2，那么在Bundle级别的依赖管理中，就会记录A包的依赖资源为B包与C包，在加载A包前会首先加载好B包与C包并增加B与C的引用计数，在卸载A包时会减少B和C的引用计数

许多现有的Unity资源框架都采用这一方式进行资源依赖管理，然而这一方式存在3个明显的缺陷：

1. **依赖文件过度加载**
2. **对Asset的依赖资源状态无知**
3. **更容易出现的循环依赖**



## 依赖文件过度加载

Bundle级别的依赖管理要求在加载一个Bundle前首先准备好其所依赖的所有Bundle

但在实践中，一个Bundle会包含许多Asset，而这些Asset又会分别依赖其他Asset，这些被依赖的Asset又会被打包在不同的Bundle中，这就导致了一个Bundle会依赖大量其他Bundle的现象出现



我们可以考虑一个比较极端但直白的例子：

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/DependencyManageCompare/DependencyManageCompare_2.png)

如上图，A包的资源a1到a100分别依赖b1到b100，此时资源清单会将A包的依赖资源记录为B1到B100

而如果此时只是想加载a1到内存中，Bundle级别的依赖管理都会要求首先将B1到B100这100个Bundle都加载到内存中，从而导致极大的IO开销和内存浪费

想要避免这种情况就要求在构建Bundle时就得做好Asset的划分，避免Bundle依赖到过多的其他Bundle



## 对Asset的依赖资源状态无知

Bundle级别的依赖管理之所以能运转，完全得益于Unity底层的对依赖加载的自动处理：在加载一个Asset前，只需要准备好它所依赖的其他Asset的Bundle，那么Unity底层便会对依赖的Asset进行自动依赖加载

但把依赖加载完全交给Unity底层的代价便是使用者对被依赖加载的Asset的使用状态一无所知，导致对可能出现的资源不卸载Bug难以排查

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/DependencyManageCompare/DependencyManageCompare_3.png)

如上图，如果出现了C包应该被卸载但无法被卸载的情况，那么Bundle级的依赖管理最多只能知道C无法卸载是因为C中的某个Asset被A或B中的某个使用中的Asset依赖着，而无法知道究竟是C中的哪个Asset被A或B中的哪个Asset依赖，由此加大了对资源不卸载Bug的排查难度



## 更容易出现的循环依赖

在前文中提到过，Bundle级别的依赖管理要求在加载一个Bundle前准备好它所依赖的其他Bundle

但由于一个Bundle中可以包含多个Asset，而一个Bundle的依赖又是根据Asset依赖的其他Asset所在的Bundle决定的，这也就导致了十分容易出现Bundle间的循环依赖



![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/DependencyManageCompare/DependencyManageCompare_4.png)

在上图的情况中，哪怕Asset间并没有产生循环依赖，依然会在Bundle间产生循环依赖，最终导致加载时爆栈

同样的，想避免这种情况就要求必须在构建Bundle时就得做好对Asset的划分



# Asset级的依赖管理



想要进行Asset级别的依赖管理，就需要抛弃Unity提供的AssetBundleManifest那一套，使用自定义的资源清单，以[CatAsset](https://github.com/CatImmortal/CatAsset)为例，其资源清单结构如下：

```csharp
    /// <summary>
    /// CatAsset资源清单
    /// </summary>
    public class CatAssetManifest
    {
        /// <summary>
        /// 游戏版本号
        /// </summary>
        public string GameVersion;

        /// <summary>
        /// 清单版本号
        /// </summary>
        public int ManifestVersion;

        /// <summary>
        /// 所有AssetBundle清单信息
        /// </summary>
        public AssetBundleManifestInfo[] AssetBundles;
    }

    /// <summary>
    /// AssetBundle清单信息
    /// </summary>
    public class AssetBundleManifestInfo
    {
        /// <summary>
        /// AssetBundle名
        /// </summary>
        public string AssetBundleName;

        /// <summary>
        /// 文件长度
        /// </summary>
        public long Length;

        /// <summary>
        /// 文件Hash
        /// </summary>
        public Hash128 Hash;

        /// <summary>
        /// 是否为场景的AssetBundle包
        /// </summary>
        public bool IsScene;

        /// <summary>
        /// 资源组
        /// </summary>
        public string Group;

        /// <summary>
        /// 所有Asset清单信息
        /// </summary>
        public AssetManifestInfo[] Assets;
    }

    /// <summary>
    /// Asset清单信息
    /// </summary>
    public class AssetManifestInfo
    {
        /// <summary>
        /// Asset名
        /// </summary>
        public string AssetName;

        /// <summary>
        /// 依赖的所有Asset
        /// </summary>
        public string[] Dependencies;
    }
```



通过在构建Bundle时调用`AssetDatabase.GetDependencies`即可获取指定Asset所依赖的其他Asset，然后便可以根据获得的依赖信息自行构建资源清单进行依赖管理

Asset级别的依赖管理可以很好的避免Bundle级别的依赖管理所容易出现的3个问题：

**首先，Asset级别的依赖管理只要求在加载Asset前准备好其所依赖的所有Asset，因此不会出现过度依赖加载的情况**

**其次，由于所有依赖资源都将通过资源框架进行加载而不再完全交给Unity底层，因此可以对依赖资源也进行显式的引用计数，这样如果有Bundle无法卸载便可以很直观的排查到是哪个Asset被引用着，被谁引用着**

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/DependencyManageCompare/DependencyManageCompare_4.png)

**最后，只有在Asset间出现循环依赖时才会导致加载爆栈，而像上图中的情况则不会对加载有任何影响**



但是Asset级别的依赖管理中存在一个较严重的问题——**卸载依赖Bundle后重新加载Asset会导致对依赖资源的引用丢失（Missing）**



## 依赖资源引用丢失

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/DependencyManageCompare/DependencyManageCompare_5.png)

由于Asset级别的依赖管理不会考虑Bundle之间的依赖，在上图所示的情况中，如果依次进行：

1. 加载a1
2. 加载a2
3. 卸载a1
4. 加载a1

就会发现a1对b1的资源引用是处于Missing状态的



那么为什么会出现这种情况呢？



因为在第3步中，当我们卸载了a1后，b1也会被作为依赖卸载，此时B包处于没有Asset在被使用中的状态，因此也就进行了对B包的卸载，**因为卸载了B包，b1已不在内存中，但A包还没被卸载，a1仍然在内存中存在，只是不处于被使用中的状态(注意：*只有AssetBundle.Unload(true)和Resources.UnloadUnusedAssets才能将AssetBundle中的已加载Asset从内存中销毁*)，所以a1对b1的引用就是Missing的**

到了第4步，我们重新加载a1，由于a1对b1的引用已经Missing，所以**即便我们再次把b1依赖加载到内存中，也不会重新恢复丢失的引用**



# 最终方案

要处理Asset级别的依赖管理中可能出现的依赖资源引用丢失问题，就需要在加载Asset时，对被依赖加载的Asset所属的Bundle进行记录，并对其增加引用计数，然后在卸载Bundle时根据记录减少对应Bundle的引用计数

同样的，在判定一个Bundle是否可以卸载时，需要不仅仅只是根据其是否有Asset处于使用中状态（即Asset引用计数>0）来判断，还需要结合此Bundle的引用计数来进行判断（也就是一个Bundle除了没有Asset在使用外，还必须没有被其他Bundle引用才能进行卸载）

通过Bundle间的引用计数虽然可以避免依赖资源引用丢失问题，但也引入了一个新的问题：

**如果产生了Bundle间的循环依赖，那么会导致Bundle间互相等待对方卸载，结果都无法卸载自身**

这个问题类似于多线程编程中的死锁问题，要处理这个问题，只能通过在构建Bundle时加入Bundle间循环依赖的检测，力图在Editor阶段就查出问题，以避免Runtime时的错误
