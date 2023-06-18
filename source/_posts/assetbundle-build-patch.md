---
title: 在Unity中构建AssetBundle补丁包
date: 2023-04-20 21:04:54
tags: 资源管理
categories: 
  - 程序
description: 如何仅构建发生了变化的资源
---

# 前言

前段时间隔壁项目组主程为了能更快速的打包项目里的**AssetBundle**，提了个增加补丁包构建的需求，于是费了几番功夫后便在[CatAsset](https://github.com/CatImmortal/CatAsset)中加入了此功能，这篇文章便用来记录此功能的实现思路



# 何为补丁包

首先需要明确几个**AssetBundle**构建相关的概念：

1. **全量构建**：将所有需要打包的资源提交给Unity的**AssetBundle**构建管线，然后Unity对所有**AssetBundle**全部进行重新构建**（此构建方式很少被使用）**
2. **增量构建**：将所有需要打包的资源提交给Unity的**AssetBundle**构建管线，然后Unity仅对发生了变化的**AssetBundle**进行重新构建**（这是最常见的标准构建方式）**
3. **补丁构建**：仅将发生了变化的资源及其相关资源提交给Unity的**AssetBundle**构建管线，然后由Unity进行**AssetBundle**构建



所谓**补丁包**，即是补丁构建的产物

举例来说，若有**资源a、b、c、d被打包为资源包A**，**e、f、g、h被打包为资源包B**（这些资源之间没有依赖关系），且其中资源b发生了变化

那么在进行**增量构建**时，会将这8个资源全部提交给Unity底层的构建管线，然后Unity会将a、b、c、d所属的资源包A进行重新构建，资源包B则从缓存中复制

而如果进行的是**补丁构建**，那么只会将发生了变化的资源b提交给Unity单独打包为资源包A_patch，与资源包A、B共存，且后续在加载资源b时只会从A_patch中加载



# 补丁构建的优缺点

补丁构建的优点主要在于：

1. 资源数量庞大的时候可以很好的加快打包速度，节省时间，减少因为**版本日打包导致的加班风险**，
2. 在进行资源更新时可以有效降低玩家需要更新的资源大小



而其缺点则主要是：

1. 因为补丁包与正式包是共存的，所以会产生一定程度的资源冗余
2. 补丁资源的计算基于与上一次完整打包（非补丁构建）产生的缓存信息的对比，所以随着变化的资源增多，会提交给Unity的资源数也会增多，构建速度也会逐渐趋近于完整打包的速度，这时需要通过重新进行一次新的完整打包来更新缓存信息



# 构建补丁包时要注意的地方

想要进行补丁包构建，首先需要记录上次完整打包的缓存信息，在CatAsset中是通过记录文件MD5实现，但不仅仅是资源文件的MD5，还需要包括资源文件对应的Meta文件的MD5才行，否则会导致补丁资源计算出错，因为有些对资源的修改是被反应到Meta文件里的



另外在计算补丁资源时，不仅是需要判断资源自身是否有变化，还需要考虑它所在的依赖链上的其他资源是否有变化，若依赖链上任意一个资源发生了变化，则此依赖链上所有资源都必须视为补丁资源处理，**因为只有在同一批AssetBundle打包的资源之间才能正确进行互相的依赖引用**，否则就会产生加载后资源依赖丢失的问题





# 构建补丁包的具体步骤

CatAsset使用SBP进行AssetBundle构建，并自定义了一些构建任务来满足需求

```csharp
        /// <summary>
        /// 构建资源包
        /// </summary>
        public static ReturnCode BuildBundles(BuildTarget targetPlatform,bool isBuildPatch)
        {
            BundleBuildConfigSO config = BundleBuildConfigSO.Instance;
            
            //预处理
            var preData = new BundleBuildPreProcessData
            {
                Config = config,
                TargetPlatform = targetPlatform
            };
            OnBundleBuildPreProcess(preData);
            
           
            if (isBuildPatch)
            {
                //检查构建补丁包的缓存文件是否存在
                //若不存在就进行完整资源包构建
                string manifestPath = EditorUtil.GetCacheManifestPath(config.OutputRootDirectory, targetPlatform);
                string assetCacheManifestPath = EditorUtil.GetAssetCacheManifestPath(config.OutputRootDirectory);
                if (!File.Exists(manifestPath) || !File.Exists(assetCacheManifestPath))
                {
                    isBuildPatch = false;
                }
            }
            
            //准备参数
            string fullOutputFolder = CreateFullOutputFolder(config, targetPlatform);
            BundleBuildInfoParam buildInfoParam = new BundleBuildInfoParam();
            BundleBuildConfigParam configParam = new BundleBuildConfigParam(config, targetPlatform,isBuildPatch);
            BundleBuildParameters buildParam = GetSBPParameters(config, targetPlatform, fullOutputFolder);
            BundleBuildContent content = new BundleBuildContent();

            //添加构建任务
            List<IBuildTask> taskList = GetSBPInternalBuildTask(!isBuildPatch);
            taskList.Insert(0,new CalculateBundleBuilds());
            taskList.Add(new BuildRawBundles());
            taskList.Add(new BuildManifest());
            taskList.Add(new EncryptBundles());
            taskList.Add(new CalculateVerifyInfo());
            if (HasOption(config.Options,BundleBuildOptions.AppendMD5))
            {
                //附加MD5到包名中
                taskList.Add(new AppendMD5());
            }
            if (isBuildPatch)
            {
                //补丁包需要合并资源清单
                taskList.Add(new MergePatchManifest());
            }
            taskList.Add(new WriteManifestFile());
            if (!isBuildPatch)
            {
                //非补丁包 写入缓存
                taskList.Add(new WriteCacheFile());
            }
            if (config.IsCopyToReadOnlyDirectory && config.TargetPlatforms.Count == 1)
            {
                //需要复制资源包到只读目录下
                taskList.Add(new CopyToReadOnlyDirectory());
                taskList.Add(new WriteManifestFile());
            }

            //开始构建资源包
            Stopwatch sw = Stopwatch.StartNew();
            
            //调用SBP的构建管线
            ReturnCode returnCode = ContentPipeline.BuildAssetBundles(buildParam, content,
                out IBundleBuildResults result, taskList, buildInfoParam,configParam);

            //检查结果
            if (returnCode == ReturnCode.Success)
            {
                Debug.Log($"资源包构建成功:{returnCode},耗时:{sw.Elapsed.Hours}时{sw.Elapsed.Minutes}分{sw.Elapsed.Seconds}秒");
            }
            else
            {
                Debug.LogError($"资源包构建未成功:{returnCode},耗时:{sw.Elapsed.Hours}时{sw.Elapsed.Minutes}分{sw.Elapsed.Seconds}秒");
            }
        
            //后处理
            var postData = new BundleBuildPostProcessData
            {
                Config = config,
                TargetPlatform = targetPlatform,
                OutputFolder = fullOutputFolder,
                ReturnCode = returnCode,
                Result = result,
            };
            OnBundleBuildPostProcess(postData);
            
            return returnCode;
        }
```



总的来说，构建补丁包需要3个步骤：

1. 在进行完整构建时记录资源缓存信息
2. 计算补丁资源
3. 合并资源清单



## 生成完整构建时的资源缓存

资源缓存信息清单的类定义

```csharp
using System;
using System.Collections.Generic;
using CatAsset.Runtime;

namespace CatAsset.Editor
{
    /// <summary>
    /// 资源缓存清单
    /// </summary>
    [Serializable]
    public class AssetCacheManifest
    {
        /// <summary>
        /// 资源缓存信息
        /// </summary>
        [Serializable]
        public struct AssetCacheInfo : IEquatable<AssetCacheInfo>
        {
            public string Name;
            public string MD5;
            public string MetaMD5;
            
            public bool Equals(AssetCacheInfo other)
            {
                return Name == other.Name && MD5 == other.MD5 && MetaMD5 == other.MetaMD5;
            }

            public override bool Equals(object obj)
            {
                return obj is AssetCacheInfo other && Equals(other);
            }

            public override int GetHashCode()
            {
                unchecked
                {
                    var hashCode = (Name != null ? Name.GetHashCode() : 0);
                    hashCode = (hashCode * 397) ^ (MD5 != null ? MD5.GetHashCode() : 0);
                    hashCode = (hashCode * 397) ^ (MetaMD5 != null ? MetaMD5.GetHashCode() : 0);
                    return hashCode;
                }
            }
            
            public static bool operator ==(AssetCacheInfo a,AssetCacheInfo b)
            {
                return Equals(a, b);
            }
            
            public static bool operator !=(AssetCacheInfo a,AssetCacheInfo b)
            {
                return !(a == b);
            }

            public static AssetCacheInfo Create(string assetName)
            {
                AssetCacheInfo assetCacheInfo = new AssetCacheInfo
                {
                    Name = assetName,
                    MD5 = RuntimeUtil.GetFileMD5(assetName),
                    MetaMD5 = RuntimeUtil.GetFileMD5($"{assetName}.meta")
                };
                return assetCacheInfo;
            }
          
        }
        
        /// <summary>
        /// 资源清单Json文件名
        /// </summary>
        public const string ManifestJsonFileName = "AssetCacheManifest.json";


        public List<AssetCacheInfo> Caches = new List<AssetCacheInfo>();

        public Dictionary<string, AssetCacheInfo> GetCacheDict()
        {
            Dictionary<string, AssetCacheInfo> result = new Dictionary<string, AssetCacheInfo>();
            foreach (AssetCacheInfo assetCache in Caches)
            {
                result.Add(assetCache.Name,assetCache);
            }

            return result;
        }
        
        
    }
}
```

正如之前提到的，除了资源文件本身的MD5外还需要记录Meta文件的MD5



```csharp
using System.IO;
using CatAsset.Runtime;
using UnityEditor;
using UnityEditor.Build.Pipeline;
using UnityEditor.Build.Pipeline.Injector;
using UnityEditor.Build.Pipeline.Interfaces;
using UnityEngine;

namespace CatAsset.Editor
{
    /// <summary>
    /// 写入缓存文件
    /// </summary>
    public class WriteCacheFile : IBuildTask
    {
        [InjectContext(ContextUsage.In)]
        private IBundleBuildConfigParam configParam;
        
        [InjectContext(ContextUsage.In)]
        private IManifestParam manifestParam;

        [InjectContext(ContextUsage.In)] 
        private IBundleBuildParameters buildParam;
        
        public int Version { get; }
        
        public ReturnCode Run()
        {
            //复制资源包构建输出结果到缓存文件夹中
            string folder = EditorUtil.GetBundleCacheFolder(configParam.Config.OutputRootDirectory,
                configParam.TargetPlatform);
            EditorUtil.CreateEmptyDirectory(folder);
            EditorUtil.CopyDirectory(((BundleBuildParameters)buildParam).OutputFolder,folder);
            
            //写入资源缓存清单
            folder = EditorUtil.GetAssetCacheManifestFolder(configParam.Config.OutputRootDirectory);
            EditorUtil.CreateEmptyDirectory(folder);
            AssetCacheManifest assetCacheManifest = new AssetCacheManifest();
            foreach (BundleManifestInfo bundleManifestInfo in manifestParam.Manifest.Bundles)
            {
                foreach (AssetManifestInfo assetManifestInfo in bundleManifestInfo.Assets)
                {
                    AssetCacheManifest.AssetCacheInfo cacheInfo = AssetCacheManifest.AssetCacheInfo.Create(assetManifestInfo.Name);
                    assetCacheManifest.Caches.Add(cacheInfo);
                }
            }
            string json = EditorJsonUtility.ToJson(assetCacheManifest,true);
            string path = RuntimeUtil.GetRegularPath(Path.Combine(folder, AssetCacheManifest.ManifestJsonFileName));
            using (StreamWriter sw = new StreamWriter(path))
            {
                sw.Write(json);
            }
            
            return ReturnCode.Success;
        }
    }
}
```

除了记录资源MD5信息外，还需要将完整构建产出的资源包复制到资源包缓存目录下，用于在补丁构建后进行合并形成最终的完整资源包输出



## 计算补丁资源

补丁资源的计算大致由以下步骤组成：

1. 判断自身是否变化
2. 判断自身依赖的资源是否变化
3. 判断依赖自身的资源是否变化



判断【资源是否变化】的标准则为：

1. 是否为新资源
2. 是否为变化的旧资源
3. 是否为被移动到新包里的旧资源



具体代码如下：

```csharp
using System.Collections.Generic;
using System.Diagnostics;
using System.IO;
using System.Linq;
using CatAsset.Runtime;
using UnityEditor;
using UnityEditor.Build.Pipeline;
using UnityEditor.Build.Pipeline.Injector;
using UnityEditor.Build.Pipeline.Interfaces;
using UnityEditor.Build.Pipeline.Utilities;
using UnityEngine;
using Debug = UnityEngine.Debug;

namespace CatAsset.Editor
{
    /// <summary>
    /// 计算资源包构建信息
    /// </summary>
    public class CalculateBundleBuilds : IBuildTask
    {
        public int Version { get; }

        [InjectContext(ContextUsage.In)] 
        private IBundleBuildParameters buildParam;

        [InjectContext(ContextUsage.InOut)] 
        private IBundleBuildInfoParam buildInfoParam;

        [InjectContext(ContextUsage.In)] 
        private IBundleBuildConfigParam configParam;

        [InjectContext(ContextUsage.In)] 
        private IBuildCache buildCache;

        [InjectContext(ContextUsage.InOut)] 
        private IBundleBuildContent content;

        [InjectContext(ContextUsage.InOut)] 
        private IBuildContent content2;

        public ReturnCode Run()
        {
            BundleBuildConfigSO config = configParam.Config;

            if (!configParam.IsBuildPatch)
            {
                //构建完整资源包
                buildInfoParam = new BundleBuildInfoParam(config.GetAssetBundleBuilds(), config.GetNormalBundleBuilds(),
                    config.GetRawBundleBuilds());
            }
            else
            {
                //构建补丁资源包
                Stopwatch sw = Stopwatch.StartNew();
                
                //双向依赖记录
                Dictionary<string, List<string>> upStreamDict = new Dictionary<string, List<string>>();
                Dictionary<string, List<string>> downStreamDict = new Dictionary<string, List<string>>();

                //资源名 -> 本次构建时所属的资源包名
                Dictionary<string, string> assetToBundle = new Dictionary<string, string>();

                //资源名 -> 上次完整构建时所属的资源包名
                Dictionary<string, string> cacheAssetToBundle = new Dictionary<string, string>();
                
                //读取上次完整构建时的资源缓存清单
                AssetCacheManifest assetCacheManifest = ReadAssetCache(config);
                
                //资源名 -> 上次完整构建时的资源缓存信息
                Dictionary<string, AssetCacheManifest.AssetCacheInfo>
                    assetCacheDict = assetCacheManifest.GetCacheDict();

                //资源名 -> 当前资源缓存信息
                Dictionary<string, AssetCacheManifest.AssetCacheInfo> curAssetCacheDict =
                    new Dictionary<string, AssetCacheManifest.AssetCacheInfo>();

                //资源名 -> 是否已变化
                Dictionary<string, bool> assetChangeStateDict = new Dictionary<string, bool>();

                //深拷贝一份构建配置进行操作
                BundleBuildConfigSO clonedConfig = Object.Instantiate(config);
                
                //获取双向依赖
                GetDependencyChain(config, upStreamDict, downStreamDict, assetToBundle);
                
                //读取上次完整构建时的资源包信息
                ReadCachedManifest(config, cacheAssetToBundle);
                
                //计算补丁资源
                CalPatchAsset(config, clonedConfig, upStreamDict, downStreamDict, assetToBundle, cacheAssetToBundle,
                    assetCacheDict, curAssetCacheDict, assetChangeStateDict);

                sw.Stop();
                Debug.Log($"计算补丁资源耗时:{sw.Elapsed.TotalSeconds:0.00}秒");
                
                buildInfoParam = new BundleBuildInfoParam(clonedConfig.GetAssetBundleBuilds(),
                    clonedConfig.GetNormalBundleBuilds(),
                    clonedConfig.GetRawBundleBuilds());
            }

            ((BundleBuildParameters)buildParam).SetBundleBuilds(buildInfoParam.NormalBundleBuilds);
            content = new BundleBuildContent(buildInfoParam.AssetBundleBuilds);
            content2 = content;

            return ReturnCode.Success;
        }

        /// <summary>
        /// 获取双向依赖
        /// </summary>
        private void GetDependencyChain(BundleBuildConfigSO config, Dictionary<string, List<string>> upStreamDict,
            Dictionary<string, List<string>> downStreamDict,
            Dictionary<string, string> assetToBundle)
        {
            int index = 0;
            foreach (var bundle in config.Bundles)
            {
                foreach (var asset in bundle.Assets)
                {
                    index++;
                    EditorUtility.DisplayProgressBar($"获取依赖信息", $"{asset.Name}", index / (config.AssetCount * 1.0f));
                    
                    //上游依赖
                    var deps = EditorUtil.GetDependencies(asset.Name);
                    upStreamDict.Add(asset.Name, deps);

                    //下游依赖
                    foreach (string dep in deps)
                    {
                        if (!downStreamDict.TryGetValue(dep, out List<string> list))
                        {
                            list = new List<string>();
                            downStreamDict.Add(dep, list);
                        }

                        list.Add(asset.Name);
                    }

                    assetToBundle.Add(asset.Name, bundle.BundleIdentifyName);
                }
            }
            
            EditorUtility.ClearProgressBar();
        }

        /// <summary>
        /// 读取上次完整构建时的资源包信息
        /// </summary>
        private void ReadCachedManifest(BundleBuildConfigSO config, Dictionary<string, string> cacheAssetToBundle)
        {
            string folder = EditorUtil.GetBundleCacheFolder(config.OutputRootDirectory, configParam.TargetPlatform);
            string path = RuntimeUtil.GetRegularPath(Path.Combine(folder, CatAssetManifest.ManifestJsonFileName));
            CatAssetManifest cachedManifest = CatAssetManifest.DeserializeFromJson(File.ReadAllText(path));
            foreach (var bundle in cachedManifest.Bundles)
            {
                foreach (var asset in bundle.Assets)
                {
                    cacheAssetToBundle.Add(asset.Name, bundle.BundleIdentifyName);
                }
            }
        }

        /// <summary>
        /// 读取上次完整构建时的资源缓存清单
        /// </summary>
        private static AssetCacheManifest ReadAssetCache(BundleBuildConfigSO config)
        {
            string folder = EditorUtil.GetAssetCacheManifestFolder(config.OutputRootDirectory);
            string path = RuntimeUtil.GetRegularPath(Path.Combine(folder, AssetCacheManifest.ManifestJsonFileName));
            string json = File.ReadAllText(path);
            AssetCacheManifest assetCacheManifest = JsonUtility.FromJson<AssetCacheManifest>(json);
            return assetCacheManifest;
        }

        /// <summary>
        /// 计算补丁资源
        /// </summary>
        private void CalPatchAsset(BundleBuildConfigSO config, BundleBuildConfigSO clonedConfig,
            Dictionary<string, List<string>> upStreamDict, Dictionary<string, List<string>> downStreamDict,
            Dictionary<string, string> assetToBundle, Dictionary<string, string> cacheAssetToBundle,
            Dictionary<string, AssetCacheManifest.AssetCacheInfo> assetCacheDict,
            Dictionary<string, AssetCacheManifest.AssetCacheInfo> curAssetCacheDict,
            Dictionary<string, bool> assetChangeStateDict)
        {
         
            int index = 0;
            for (int i = clonedConfig.Bundles.Count - 1; i >= 0; i--)
            {
                var bundle = clonedConfig.Bundles[i];

                //此资源包是否全部资源都变化了
                bool isAllChanged = true;

                for (int j = bundle.Assets.Count - 1; j >= 0; j--)
                {
                    var asset = bundle.Assets[j];
                    index++;
                    EditorUtility.DisplayProgressBar($"计算补丁资源", $"{asset.Name}", index / (config.AssetCount * 1.0f));

                    //1.自身是否变化
                    bool isChanged = IsChangedAsset(asset.Name, assetChangeStateDict, assetCacheDict, curAssetCacheDict,
                        assetToBundle, cacheAssetToBundle);

                    if (!isChanged)
                    {
                        //2.此资源依赖的资源是否变化
                        if (upStreamDict.TryGetValue(asset.Name, out var upStreamList))
                        {
                            foreach (string upStream in upStreamList)
                            {
                                isChanged = IsChangedAsset(upStream, assetChangeStateDict, assetCacheDict,
                                    curAssetCacheDict,
                                    assetToBundle, cacheAssetToBundle);
                                if (isChanged)
                                {
                                    break;
                                }
                            }
                        }
                    }

                    if (!isChanged)
                    {
                        //3.依赖此资源的资源是否变化
                        if (downStreamDict.TryGetValue(asset.Name, out var downStreamList))
                        {
                            foreach (string downStream in downStreamList)
                            {
                                isChanged = IsChangedAsset(downStream, assetChangeStateDict, assetCacheDict,
                                    curAssetCacheDict,
                                    assetToBundle, cacheAssetToBundle);
                                if (isChanged)
                                {
                                    break;
                                }
                            }
                        }
                    }

                    if (isChanged)
                    {
                        Debug.Log($"发现补丁资源:{asset.Name}");
                    }
                    else
                    {
                        //移除非补丁资源
                        bundle.Assets.RemoveAt(j);
                        isAllChanged = false;
                    }
                }

                if (bundle.Assets.Count > 0)
                {
                    //是补丁包
                    if (!isAllChanged)
                    {
                        //有部分资源不是补丁资源 需要改名 否则直接用本体包的名字转正
                        var part = bundle.BundleName.Split('.');
                        bundle.BundleName = $"{part[0]}_patch.{part[1]}";
                        bundle.BundleIdentifyName =
                            BundleBuildInfo.GetBundleIdentifyName(bundle.DirectoryName, bundle.BundleName);
                    }
                }
                else
                {
                    //不是补丁包 需要移除
                    clonedConfig.Bundles.RemoveAt(i);
                }
            }

            EditorUtility.ClearProgressBar();
        }

        /// <summary>
        /// 是否为已变化的资源
        /// </summary>
        private bool IsChangedAsset(string assetName, Dictionary<string, bool> assetChangeStateDict,
            Dictionary<string, AssetCacheManifest.AssetCacheInfo> assetCacheDict,
            Dictionary<string, AssetCacheManifest.AssetCacheInfo> curAssetCacheDict,
            Dictionary<string, string> assetToBundle, Dictionary<string, string> cacheAssetToBundle)

        {
            //0.已经计算过状态了
            if (assetChangeStateDict.TryGetValue(assetName, out bool isPatch))
            {
                return isPatch;
            }

            //1.新资源 
            if (!assetCacheDict.TryGetValue(assetName, out var assetCache))
            {
                assetChangeStateDict.Add(assetName, true);
                return true;
            }

            //2.变化的旧资源
            if (!curAssetCacheDict.TryGetValue(assetName, out var curAssetCache))
            {
                curAssetCache = AssetCacheManifest.AssetCacheInfo.Create(assetName);
                curAssetCacheDict.Add(assetName, curAssetCache);
            }

            if (curAssetCache != assetCache)
            {
                assetChangeStateDict.Add(assetName, true);
                return true;
            }

            //3.被移动到新包的旧资源
            if (assetToBundle[assetName] != cacheAssetToBundle[assetName])
            {
                assetChangeStateDict.Add(assetName, true);
                return true;
            }

            //未变化
            assetChangeStateDict.Add(assetName, false);
            return false;
        }
    }
}
```



上述代码会在完整构建的资源列表基础上，移除所有非补丁资源，以及不包含补丁资源的资源包，以此形成用于最终补丁构建的资源列表

另外值得一提的是，为了降低补丁包带来的复杂度与资源冗余，CatAsset规定了每个正式包最多只会有一个补丁包，同时如果某个补丁包包含了正式包的所有资源，则此补丁包会自动转正为正式包



## 合并资源清单

在构建补丁包完成后，需要将新的补丁资源包与上一次完整构建时输出的资源包进行合并，以得到最终输出的完整资源包与资源清单

```csharp
using System.Collections.Generic;
using System.IO;
using CatAsset.Runtime;
using UnityEditor;
using UnityEditor.Build.Pipeline;
using UnityEditor.Build.Pipeline.Injector;
using UnityEditor.Build.Pipeline.Interfaces;
using UnityEngine;

namespace CatAsset.Editor
{
    /// <summary>
    /// 合并补丁包资源清单
    /// </summary>
    public class MergePatchManifest : IBuildTask
    {
        [InjectContext(ContextUsage.In)]
        private IBundleBuildParameters buildParam;

        [InjectContext(ContextUsage.In)]
        private IBundleBuildConfigParam configParam;

        [InjectContext(ContextUsage.InOut)]
        private IManifestParam manifestParam;

        /// <inheritdoc />
        public int Version => 1;

        /// <inheritdoc />
        public ReturnCode Run()
        {
            var config = configParam.Config;
            
            var bundleCacheFolder = EditorUtil.GetBundleCacheFolder(config.OutputRootDirectory, configParam.TargetPlatform);

            //本次补丁包构建的资源
            HashSet<string> patchAssets = new HashSet<string>();
            foreach (BundleManifestInfo bundleManifestInfo in manifestParam.Manifest.Bundles)
            {
                foreach (AssetManifestInfo assetManifestInfo in bundleManifestInfo.Assets)
                {
                    patchAssets.Add(assetManifestInfo.Name);
                }
            }

            //修改资源清单 移除重复资源
            string path = RuntimeUtil.GetRegularPath(Path.Combine(bundleCacheFolder, CatAssetManifest.ManifestJsonFileName));
            CatAssetManifest cachedManifest = CatAssetManifest.DeserializeFromJson(File.ReadAllText(path));
            for (int i = cachedManifest.Bundles.Count - 1; i >= 0; i--)
            {
                BundleManifestInfo bundleManifestInfo = cachedManifest.Bundles[i];
                if (bundleManifestInfo.BundleName == RuntimeUtil.BuiltInShaderBundleName)
                {
                    //跳过内置Shader资源包
                    continue;
                }
                
                for (int j = bundleManifestInfo.Assets.Count - 1; j >= 0; j--)
                {
                    //删掉已经在补丁包中的资源信息
                    AssetManifestInfo assetManifestInfo = bundleManifestInfo.Assets[j];
                    if (patchAssets.Contains(assetManifestInfo.Name))
                    {
                        bundleManifestInfo.Assets.RemoveAt(j);
                    }
                }

                if (bundleManifestInfo.Assets.Count == 0)
                {
                    //删掉所有资源都在补丁包里的资源包
                    cachedManifest.Bundles.RemoveAt(i);
                }
            }

            //合并资源包
            string outputFolder = ((BundleBuildParameters)buildParam).OutputFolder;
            foreach (BundleManifestInfo bundleManifestInfo in cachedManifest.Bundles)
            {
                string sourcePath = RuntimeUtil.GetRegularPath(Path.Combine(bundleCacheFolder, bundleManifestInfo.RelativePath));
                string destPath = RuntimeUtil.GetRegularPath(Path.Combine(outputFolder,bundleManifestInfo.RelativePath));
                
                if (!string.IsNullOrEmpty(bundleManifestInfo.Directory))
                {
                    string fullDirectory = RuntimeUtil.GetRegularPath(Path.Combine(outputFolder,bundleManifestInfo.Directory));
                    if (!Directory.Exists(fullDirectory))
                    {
                        Directory.CreateDirectory(fullDirectory);
                    }
                }
                
                File.Copy(sourcePath,destPath);
            }
            
            //合并补丁包资源清单与缓存资源清单
            cachedManifest.Bundles.AddRange(manifestParam.Manifest.Bundles);
            manifestParam = new ManifestParam(cachedManifest, manifestParam.WriteFolder);
            
          
            
            return ReturnCode.Success;
        }


    }
}

```

此操作会将在原来被缓存的资源清单中存在的补丁资源， 以及所有资源都是补丁资源的资源包从原资源清单中移除，然后与补丁构建产生的资源清单合并，最终得到一份新的资源清单用于正确的初始化资源信息

以之前的例子来说，补丁构建产生的资源清单只会包含A_patch和资源b的信息，并且会将原资源清单中的资源b信息移除，然后将二者进行合并以得到新的资源清单
