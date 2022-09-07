---
title: CatAsset开发总结：Editor篇
date: 2022-09-01 21:02:08
tags: 资源管理
categories: 
  - 程序
  - 开发总结
description: Unity资源管理框架
top_img: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithEditor/CatAssetDevSummaryWithEditor_01.png
cover: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithEditor/CatAssetDevSummaryWithEditor_01.png
---

# 前言

本文主要用于总结[CatAsset](https://github.com/CatImmortal/CatAsset)在Editor部分的设计思路

代码结构如下：

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithEditor/CatAssetDevSummaryWithEditor_01.png)

- BuildInfo：包含用于保存资源构建相关信息的数据结构
- BuidPipeline：包含基于ScriptableBuildPipeline的资源构建管线实现
- Config：包含与资源构建相关的配置
- Misc：包含循环依赖分析器、冗余资源分析器与一些工具类的代码
- Rule：包含应用于资源目录的构建规则的接口与实现
- Window：包含资源包构建窗口与运行时信息窗口的编辑器窗口类代码



# 构建信息

构建信息分为2种：

1. `AssetBuildInfo`（资源构建信息）
2. `BundBuildInfo`（资源包构建信息）



`AssetBuildInfo`表示单个资源的构建信息：

```csharp
 /// <summary>
    /// 资源构建信息
    /// </summary>
    [Serializable]
    public class AssetBuildInfo : IComparable<AssetBuildInfo>,IEquatable<AssetBuildInfo>
    {
        /// <summary>
        /// 资源名
        /// </summary>
        public string Name;

        /// <summary>
        /// 资源类型名
        /// </summary>
        public string TypeName;

        private Type type;
        /// <summary>
        /// 资源类型
        /// </summary>
        public Type Type => type ??= AssetDatabase.GetMainAssetTypeAtPath(Name);

        /// <summary>
        /// 资源文件长度
        /// </summary>
        public long Length;
       
        //省略...
    }
```



`BundleBuildInfo`用于表示资源包级别的构建信息：

```csharp
 	/// <summary>
    /// 资源包构建信息
    /// </summary>
    [Serializable]
    public class BundleBuildInfo : IComparable<BundleBuildInfo>,IEquatable<BundleBuildInfo>
    {
        /// <summary>
        /// 相对路径
        /// </summary>
        public string RelativePath;
        
        /// <summary>
        /// 目录名
        /// </summary>
        public string DirectoryName;
        
        /// <summary>
        /// 资源包名
        /// </summary>
        public string BundleName;

        /// <summary>
        /// 资源组
        /// </summary>
        public string Group;

        /// <summary>
        /// 是否为原生资源包
        /// </summary>
        public bool IsRaw;
        
        /// <summary>
        /// 总资源长度
        /// </summary>
        public long AssetsLength;
        
        /// <summary>
        /// 资源构建信息列表
        /// </summary>
        public List<AssetBuildInfo> Assets = new List<AssetBuildInfo>();

        //省略...

        /// <summary>
        /// 获取用于构建资源包的AssetBundleBuild
        /// </summary>
        public AssetBundleBuild GetAssetBundleBuild()
        {
            AssetBundleBuild bundleBuild = new AssetBundleBuild
            {
                assetBundleName = RelativePath
            };

            List<string> assetNames = new List<string>();
            foreach (AssetBuildInfo assetBuildInfo in Assets)
            {
                assetNames.Add(assetBuildInfo.Name);
            }

            bundleBuild.assetNames = assetNames.ToArray();

            return bundleBuild;
        }
        
        //省略...
    
     
        
    }
```

`BuildBundleInfo`除了保存必要的构建信息外，还提供了从资源构建信息中生成用于构建`AssetBundle`的`AssetBundleBuild`的接口



# 资源类别

`BundleBuildInfo`中有一个**IsRaw**字段来区分当前**Bundle**是否为原生资源的**Bundle**

这里需要引入对资源类别的划分，CatAsset通过2个维度划分了3种资源类别：

|                                           | 内置资源（通过CatAsset构建） | 外置资源（不通过CatAsset构建） |
| ----------------------------------------- | ---------------------------- | ------------------------------ |
| 资源包资源（需要构建为AssetBundle的资源） | 内置资源包资源               |                                |
| 原生资源（直接使用本体的资源）            | 内置原生资源                 | 外置原生资源                   |

CatAsset的Runtime接口统一了这3种类别资源的使用，使用者在使用资源时只需要路径符合规范，无需关心具体资源类别



## 为什么没有外置资源包资源这一类别？

如果一个**AssetBundle**不是通过CatAsset进行构建的，那么是无法对其进行统一管理的



## Mod资源是哪一种类别？

如果要为游戏提供Mod功能，那么对于Mod资源的处理可以使用2种方案：

1.基于外置原生资源的方案：将Mod资源限制在图片、文本、音频等可以直接从二进制构造的资源类型，Mod开发者将编辑好的资源文件本体直接放入读写区内，此方案优点是Mod开发门槛低，上手简单，但缺点是支持的资源类型少

2.基于导入内置资源包的方案：Mod开发者通过Unity编辑好资源后，使用CatAsset进行资源包构建，然后通过CatAsset提供的从外部导入资源包接口（详见[CatAsset使用教程](http://cathole.top/2022/08/30/catasset-guide/)中**导入外部的资源包**一节）来将Mod资源包信息导入`CatAssetManager`中，以实现对Mod资源的加载，此方案缺点是Mod开发门槛高，上手略微复杂，但优点是支持全部的资源类型



# 资源目录与构建规则

要想生成构建信息，就需要通过指定**资源目录**与其所使用的**构建规则**进行

构建规则接口`IBundleBuildRule`定义如下：

```csharp
    /// <summary>
    /// 资源包构建规则接口
    /// </summary>
    public interface IBundleBuildRule
    {
        /// <summary>
        /// 此规则所构建的是否为原生资源包
        /// </summary>
        /// <returns></returns>
        bool IsRaw { get; }
        
        /// <summary>
        /// 获取使用此规则构建的资源包构建信息列表
        /// </summary>
        List<BundleBuildInfo> GetBundleList(BundleBuildDirectory bundleBuildDirectory);
    }
```



目前CatAsset内置了4种构建规则：

1. `NAssetToNBundle`（将指定目录下所有资源分别构建为一个资源包）
2. `NAssetToNRawBundle`（将指定目录下所有资源分别构建为一个原生资源包）
3. `NAssetToOneBundle`（将指定目录下所有资源构建为一个资源包）
4. `NAssetToOneBundleWithTopDirectory`（将指定目录下所有一级子目录各自使用`NAssetToOneBundle`规则进行构建）



## 原生资源的特殊性

对于原生资源而言，其不会被加入到**AssetBundle**的构建中，而是直接使用其资源文件本体并以二进制格式加载，所以也就不存在一个物理意义上的资源包文件

但是为了方便进行统一管理，CatAsset会在构建原生资源时，为其虚拟一个`BundleBuildInfo`，并将**BundleName**设置为此原生资源的文件名：

```csharp
  string assetsDir = Util.FullNameToAssetName(file.Directory.FullName); //Assets/xxx
                    string directoryName = assetsDir.Substring(assetsDir.IndexOf("/") + 1); //去掉Assets/
                    string bundleName;
                    if (!isRaw)
                    { 
                        bundleName = file.Name.Replace('.','_').ToLower() + ".bundle"; 
                    }
                    else
                    {
                        //直接以文件名作为原生资源包名
                        bundleName = file.Name;
                    }
                    
                    BundleBuildInfo bundleBuildInfo =
                        new BundleBuildInfo(directoryName,bundleName, group, isRaw);
```

也因此，1个原生资源包的`BundleBuildInfo`固定只会包含1个原生资源的`AssetBuildInfo`

*（可以理解为原生资源 = 原生资源包）*

# 生成构建信息

在指定好资源目录与构建规则后，即可生成构建信息，生成后的构建信息被保存在**BundleBuildConfig.asset**文件中

CatAsset将构建信息的生成分为6个步骤：

1. **初始化资源包构建规则**

2. **根据构建规则初始化资源包构建信息**

3. **将隐式依赖转为显式构建资源**

4. **冗余资源分析**

5. **分割场景资源包中的非场景资源**

6. **初始化原生资源包构建信息**

   

## 初始化资源包构建规则

此步骤通过反射来创建已实现的构建规则对象

```csharp
		/// <summary>
        /// 初始化资源包构建规则字典
        /// </summary>
        private void InitRuleDict()
        {
            Type[] types = typeof(BundleBuildConfigSO).Assembly.GetTypes();
            foreach (Type type in types)
            {
                if (!type.IsInterface && typeof(IBundleBuildRule).IsAssignableFrom(type) &&
                    !ruleDict.ContainsKey(type.Name))
                {
                    IBundleBuildRule rule = (IBundleBuildRule) Activator.CreateInstance(type);
                    ruleDict.Add(type.Name, rule);
                }
            }
        }
```



## 根据构建规则初始化资源包构建信息

此步骤通过在资源目录上调用对应构建规则，以获得初始的`BundleBuildInfo`列表

```csharp
		/// <summary>
        /// 根据构建规则获取初始的资源包构建信息列表
        /// </summary>
        private void InitBundleBuildInfo(bool isRaw)
        {
            for (int i = 0; i < Directories.Count; i++)
            {
                BundleBuildDirectory bundleBuildDirectory = Directories[i];

                IBundleBuildRule rule = ruleDict[bundleBuildDirectory.BuildRuleName];
                if (rule.IsRaw == isRaw)
                {
                    List<BundleBuildInfo> bundles = rule.GetBundleList(bundleBuildDirectory);
                    Bundles.AddRange(bundles);
                }
            }
        }
```

*（此步骤不初始化原生资源包的构建信息）*



## 将隐式依赖转为显式构建资源

**隐式依赖**：若资源A依赖资源B，资源A出现在初始资源包构建信息列表中但是资源B没有，那么就称资源B为资源A的隐式依赖



因为CatAsset Runtime部分的依赖管理是基于**Asset**的（关于此方面的探讨见[Unity资源框架设计中不同级别依赖管理的对比](http://cathole.top/2021/10/29/dependency-manage-compare/)），所有**依赖Asset**都要通过框架加载，所以必须在这一个步骤中将隐式依赖都找出来，和依赖其的资源一并构建进同一个资源包中



## 冗余资源分析

在上述步骤中，如果资源A、B同时隐式依赖资源C，并且A、B并未被构建进同一个资源包中，那么C就会被分别构建进A和B所在的资源包中，从而产生资源冗余的问题



此步骤便会将冗余资源单独构建进额外创建的冗余资源包中



由于CatAsset的卸载机制是基于资源包的卸载（只会在资源包卸载时才会真正将从此资源包中加载的资源从内存中删除），所以会尽量按照冗余资源间的关联性进行资源包划分，以保证冗余资源及依赖其的资源在生命周期上的一致性



举例来说

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/CatAssetDevSummaryWithEditor/CatAssetDevSummaryWithEditor_02.png)

在上图中，冗余资源X、Y会被划分为同一个资源包，Z则是另一个资源包



具体做法为遍历所有冗余资源，从被遍历的每一个冗余资源出发去搜索其他所有可达冗余资源（如果在之前已被搜索过则忽略），搜索结束后将所有记录到的冗余资源划分为同一个冗余资源包



## 分割场景资源包中的非场景资源

在将隐式依赖转换为显式构建资源后，可能出现场景资源和非场景资源被放进了同一个资源包的情况

而这是Unity不允许的，会在构建资源包时报错，所以需要在此步骤将其拆开



## 初始化原生资源包构建信息

如果出现普通资源包中的资源依赖原生资源（一般来说很少出现这种情况），那么需要冗余一份原生资源到普通资源包中，因为本质上原生资源是不从**AssetBundle**加载的，只作为二进制数据被加载，所以依赖加载是无效的



这里将原生资源包的构建放到最后处理，这样通过前面的隐式依赖转换为显式构建资源这一步骤就可以达成原生资源在依赖它的普通资源包中的冗余了



# 循环依赖分析

CatAsset提供了基于资源的与基于资源包的循环依赖分析功能



做法也很简单，通过对资源/资源包的依赖进行深度搜索，在搜索过程中记录当前依赖链，如果出现重复记录则意味着依赖链是环状的，也就是出现了循环依赖



# 资源构建管线

CatAsset的资源构建管线基于ScriptableBuildPipeline实现



## 默认构建任务

在SBP原有的Task基础，增加了自定义的Task：

1. `BuildRawBundles`（构建原生资源包）
2. `BuildManifest`（构建资源清单）
3. `WriteManifestFile`（写入资源清单文件）
4. `CopyToReadOnlyDirectory`（复制指定资源组的资源到只读目录下）



执行顺序为：

```csharp
			//添加构建任务
            IList<IBuildTask> taskList = DefaultBuildTasks.Create(DefaultBuildTasks.Preset.AssetBundleCompatible);
            taskList.Add(new BuildRawBundles());
            taskList.Add(new BuildManifest());
            taskList.Add(new WriteManifestFile());
            if (bundleBuildConfig.IsCopyToReadOnlyDirectory && bundleBuildConfig.TargetPlatforms.Count == 1)
            {
                //需要复制资源包到只读目录下
                taskList.Add(new CopyToReadOnlyDirectory());
                taskList.Add(new WriteManifestFile());
            }
```



### BuildRawBundles

此Task直接将原生资源复制到指定路径之下

```csharp
//遍历原生资源包列表
            foreach (BundleBuildInfo rawBundleBuildInfo in rawBundleBuilds)
            {
                string rawAssetName = rawBundleBuildInfo.Assets[0].Name;
                string rawBundleDirectory = Path.Combine(directory, rawBundleBuildInfo.DirectoryName.ToLower());
                if (!Directory.Exists(rawBundleDirectory))
                {
                    Directory.CreateDirectory(rawBundleDirectory);
                }

                string targetFileName = Path.Combine(directory, rawBundleBuildInfo.RelativePath);
                File.Copy(rawAssetName, targetFileName); //直接将原生资源复制过去
            }
```



### BuildManifest

此Task负责创建出`CatAssetManifest`对象并将其传递到构建管线的后续环节

```csharp
    /// <summary>
    /// 构建资源清单
    /// </summary>
    public class BuildManifest : IBuildTask
    {
      
        //省略...

        [InjectContext(ContextUsage.Out)] 
        private IManifestParam manifestParam;
        
        /// <inheritdoc />
        public int Version => 1;

        
        /// <inheritdoc />
        public ReturnCode Run()
        {
            string outputFolder = ((BundleBuildParameters) buildParam).OutputFolder;

            //创建资源清单
            CatAssetManifest manifest = new CatAssetManifest
            {
                GameVersion = Application.version,
                ManifestVersion = configParam.Config.ManifestVersion,
                Bundles = new List<BundleManifestInfo>(),
            };

            //省略...

            manifestParam = new ManifestParam(manifest,outputFolder);
            
            return ReturnCode.Success;
        }
    }
```



### WriteManifestFile

此Task负责将CatAssetManifest对象写入到指定路径下

```csharp
string writePath = manifestParam.WritePath;
            CatAssetManifest manifest = manifestParam.Manifest;
                
            //写入清单文件json
            string json = CatJson.JsonParser.ToJson(manifest);
            using (StreamWriter sw = new StreamWriter(Path.Combine(writePath, CatAsset.Runtime.Util.ManifestFileName)))
            {
                sw.Write(json);
            }
```



### CopyToReadOnlyDirectory

此Task负责将指定资源组的资源复制到只读区内，并生成仅包含被复制的资源信息的CatAssetManifest并写入只读区内



## 仅构建原生资源包

CatAsset的资源构建管线提供了仅构建原生资源包的功能

此时因为不涉及**AssetBundle**的构建，所以不会使用SBP提供的Task

```csharp
			//添加构建任务
            IList<IBuildTask> taskList = new List<IBuildTask>();
            taskList.Add(new BuildRawBundles());
            taskList.Add(new BuildManifest());
            taskList.Add(new MergeManifestAndBundles());
            taskList.Add(new WriteManifestFile());
            if (bundleBuildConfig.IsCopyToReadOnlyDirectory && bundleBuildConfig.TargetPlatforms.Count == 1)
            {
                //需要复制资源包到只读目录下
                taskList.Add(new CopyToReadOnlyDirectory());
                taskList.Add(new WriteManifestFile());
            }
```



与默认构建任务不同的是，多了一个**MergeManifestAndBundles**（合并资源清单与资源包）任务



### MergeManifestAndBundles

此任务会将前一个版本的**AssetBundle**资源包和当前被构建的原生资源包合并输出到指定目录下（资源清单也会被同时合并），保证每次构建后得到的都是完整资源包输出
