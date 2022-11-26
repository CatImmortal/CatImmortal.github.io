---
title: UnitySBP打包SpriteAtlas图集时的纹理冗余Bug
date: 2022-11-26 15:11:56
tags: 资源管理
categories: 
  - 程序
  - 踩坑记录
---

# 前言

在使用SpriteAtlas时，通常会将include in build勾选，然后将SpriteAtlas的所有散图打包进同一个AssetBundle中（但图集本身不打包），最终打包出的AssetBundle会包含图集Texture+所有散图的Sprite



假设有图片G0、G1、G2、G3被打包进了同一个AssetBundle中，并且归属于同一个SpriteAtlas，那么此AssetBundle所包含的资源便如下：

![](	https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/SBPSpriteAtlasBug/SBPSpriteAtlasBug_01.png)



而在没有SpriteAtlas的情况下，则是这样的：

![](	https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/SBPSpriteAtlasBug/SBPSpriteAtlasBug_02.png)

可以看出2个AssetBundle的文件体积差距并不大，其主要差异在其中包含的资源上



那么是什么导致这样的差异呢？

这里就需要提及在反复打包测试后得出的， Unity处理SpriteAtlas时的2个重要规则了：

1. 在图片被打包时，如果此图片属于某个SpriteAtlas，那么Unity就**只会打包此图片的Sprite，而不会打包Texture**，因为在加载此图片时，得实际从SpriteAtlas的Texture中进行加载
2. 如果**勾选了SpriteAtlas的include in build，那么会建立起散图对SpriteAtlas的隐式依赖**，后续在打包图片时就会按照Unity的自动依赖收集机制决定SpriteAtlas是进入安装包还是进入AssetBundle，这样Unity就可以在需要加载散图时自动找到对应的SpriteAtlas（否则就需要使用者自行做late binding处理）。而且Unity会自动**修正因为通过隐式依赖进入AssetBundle中的SpriteAtlas冗余**（但是在老版本中因为Bug导致没有这个修正，从而造成SpriteAtlas在AssetBundle中的多份冗余）



# SBP处理SpriteAtlas的Bug

然而可惜的是，在Unity的SBP中（可编程构建管线），第1个规则并没有生效，于是就形成了SBP与Builtin构建管线的结果不一致现象

通过SBP对上面相同资源的打包结果如下：

![](	https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/SBPSpriteAtlasBug/SBPSpriteAtlasBug_03.png)

可以看到散图的Texture也一并被打包了，最终造成**纹理的双倍冗余**



在反复测试后，发现只有单独打包SpriteAtlas而不打包散图的情况下，才能避免纹理冗余：

![](	https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/SBPSpriteAtlasBug/SBPSpriteAtlasBug_04.png)

但这么做的弊端就是上层使用者无法直接从AssetBundle中手动加载散图，必须先加载SpriteAtlas，然后再从SpriteAtlas中手动加载散图才行，最终破坏了原本上层使用者对于SpriteAtlas的**“无感知”体验优势**，而想恢复这个优势就必须自行在资源框架层进行额外处理



# 解决方案

在发现这个Bug后，便先试图反馈到Unity官方，但是即使反馈成功也不知道何时会进行修复，所以只能先自己动手解决Bug了

思路也比较简单，得益于SBP高度可定制的构建流程，**只需要在散图AssetBundle的内容生成后——文件写入前这个时机，通过插入自定义的BuildTask，将冗余出来的散图纹理从中删掉即可**，具体代码如下：

```csharp
 	/// <summary>
    /// 修复SBP构建包含图集散图的资源包时，发生散图纹理冗余的Bug
    /// </summary>
    public class FixSpriteAtlasBug : IBuildTask
    {
        /// <inheritdoc />
        public int Version => 1;
        
        [InjectContext]
        IBundleWriteData writeDataParam;
        
        /// <inheritdoc />
        public ReturnCode Run()
        {
            BundleWriteData writeData = (BundleWriteData)writeDataParam;

            //所有图集散图的guid集合
            HashSet<GUID> spriteGuids = new  HashSet<GUID>();

            //遍历资源包里的资源 记录其中图集的散图guid
            foreach (var pair in writeData.FileToObjects)
            {
                foreach (ObjectIdentifier objectIdentifier in pair.Value)
                {
                    string path = AssetDatabase.GUIDToAssetPath(objectIdentifier.guid);
                    Object asset = AssetDatabase.LoadAssetAtPath<Object>(path);
                    if (asset is SpriteAtlas)
                    {
                        List<string> spritePaths = EditorUtil.GetDependencies(path, false);
                        foreach (string spritePath in spritePaths)
                        {
                            GUID spriteGuild = AssetDatabase.GUIDFromAssetPath(spritePath);
                            spriteGuids.Add(spriteGuild);
                        }
                    }
                }
            }

            //将writeData.FileToObjects包含的图集散图的texture删掉 避免冗余
            foreach (var pair in writeData.FileToObjects)
            {
                List<ObjectIdentifier> objectIdentifiers = pair.Value;
                for (int i = objectIdentifiers.Count - 1; i >= 0; i--)
                {
                    ObjectIdentifier objectIdentifier = objectIdentifiers[i];
                    if (spriteGuids.Contains(objectIdentifier.guid))
                    {
                        if (objectIdentifier.localIdentifierInFile == 2800000)
                        {
                            //删除图集散图的冗余texture
                            objectIdentifiers.RemoveAt(i);
                        }
                    }
                }
            }
            
            return ReturnCode.Success;
        }
    }
```

```csharp
 		/// <summary>
        /// 获取SBP内置的构建任务
        /// </summary>
        private static List<IBuildTask> GetSBPInternalBuildTask()
        {
            //...省略

            // Packing
            buildTasks.Add(new GenerateBundlePacking());
            buildTasks.Add(new FixSpriteAtlasBug());  //这里插入一个修复SBP图集Bug的任务
            buildTasks.Add(new UpdateBundleObjectLayout());
            buildTasks.Add(new GenerateBundleCommands());
            buildTasks.Add(new GenerateSubAssetPathMaps());
            buildTasks.Add(new GenerateBundleMaps());
            buildTasks.Add(new PostPackingCallback());

            //...省略

            return buildTasks;
        }
```



**注意：此修复的限制在于，同一个SpriteAtlas的的所有散图必须被打包进同一个AssetBundle，并且如果SpriteAtlas没有勾选include in build，也必须和它所包含的散图在同一个AssetBundle中**
