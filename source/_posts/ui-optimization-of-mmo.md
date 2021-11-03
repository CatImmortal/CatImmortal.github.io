---
title: 手游MMO项目的UI优化实践
date: 2021-11-02 21:18:02
tags: 性能优化
categories: 程序
description: 关于UnityUI性能优化的项目实践
top_img: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/UIOptimizatioOfMMO/UIOptimizatioOfMMO_4.png
cover: https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/UIOptimizatioOfMMO/UIOptimizatioOfMMO_4.png
---

# 前言

本文主要记录笔者去年在一个处于开发后期的UnityMMO项目中，根据UWA性能分析报告进行UI优化的实践经验

对UI优化的实践主要分为以下几方面：

- Drawcall
- Rebuild
- Rebatch
- Overdraw
- Instance

本文将从这几个方面出发，回顾在项目中进行UI优化的实践经验



# Drawcall

Drawcall，即绘制调用命令

CPU在准备好渲染数据后会通过Drawcall命令通知GPU进行渲染

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/UIOptimizatioOfMMO/UIOptimizatioOfMMO_1.png)

CPU每次调用Drawcall前准备渲染数据时都会产生不少的性能消耗，而GPU的渲染速度是非常快的，常常出现CPU喂不饱GPU的情况，因此可以通过合批的形式一次性发送大量渲染数据到GPU中，以减少Drawcall调用次数



UI的合批规则主要根据UI网格覆盖范围计算：

1. 如果一个UI底下没有其他UI，那么此UI深度为0
2. 如果一个UI底下有其他UI，并且此UI可以与底下深度最大的UI合批（材质相同），那么此UI深度和那个UI相同
3. 如果一个UI底下有其他UI，并且不能与底下深度最大的UI合批，那么此UI深度为底下最大深度+1
4. 计算出深度后，Unity会根据深度排序然后将深度相同的UI用1个Drawcall画出来



UI的合批主要依靠图集来进行，项目中当然也做了这样的处理，而笔者所做的工作主要在于通过FrameDebugger来进行逐步调试，以查出是什么地方应该合批而没有合批，无法合批的原因是什么



## 网格穿插

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/UIOptimizatioOfMMO/UIOptimizatioOfMMO_2.png)

MMO项目中主界面右上角一般有大量的活动Icon，这些Icon的合批失败会造成许多的额外Drawcall

项目中活动Icon合批失败的原因，在于红点的网格穿插到了旁边活动Icon的背景图，以致于无法按照期望的那样第1个Drawcall画完所有背景图，第2个Drwacall画完所有红点



举个简单的例子，在下图中，经过UI深度规则计算，所有白图的深度为0，红图深度为1，所以只需要2个Drawcall

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/UIOptimizatioOfMMO/UIOptimizatioOfMMO_3.png)



而像下图这样，因为红图位置偏了点，导致第二个白图被覆盖到了红图上，最终深度计算结果就是：白图1的深度为0，红图1的深度为1，白图2的深度为2，红图2的深度为3，只能使用4个Drawcall才能画完

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/UIOptimizatioOfMMO/UIOptimizatioOfMMO_4.png)



对于这种情况解决方案当然就是调整活动Icon预制，使红点处于正确位置



## Mask组件

在使用UI的滚动列表时，通常默认的遮罩组件是Mask

Mask遮罩的实现原理是模板缓存，其坑点在于自身就会造成额外Drawcall，并且被Mask隐藏掉的物体并不是真正的被裁剪了

通过将Scene界面的Shading Mode设置为Wireframe，会看到这些物体的网格信息仍然存在，仍然被视为渲染中的物体，这也就意味着如果这些item是无法合批的，那么会造成大量的额外Drawcall，事实上笔者就曾经遇到过这样一个Drawcall高达数百的item列表

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/UIOptimizatioOfMMO/UIOptimizatioOfMMO_5.png)



解决方案主要是将Mask替换为RectMask2D，其实现原理为调用了Shader中的Clip函数，实现了对遮罩隐藏物体的真正裁剪，并且该组件本身也不会造成额外Drawcall

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/UIOptimizatioOfMMO/UIOptimizatioOfMMO_6.png)



## Z值不为0

曾经有一次拿FrameDebugger死活没看出来为啥UI合批失败，最后还是主程提醒了一下才发现是有UI的PosZ值不为0导致的



# Rebuild

Rebuild是指当UI自身发生变化时，从C#层发出的UI重建（比如缩放UI，切换图片，开关UI），一般都会造成不小的性能开销

其在Profiler中为Canvas.SendWillRenderCanvases项，其中CanvasUpdateRegistry.LayoutRebuild为重建布局信息，CanvasUpdateRegistry.GraphicRebuild为重建网格与材质信息

这一项主要依靠通过脚本反射两个Rebuild队列进行监控，找出高频触发Rebuild的UI然后进行针对性优化

脚本代码如下：

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;
using UnityEngine.UI;
using System;
using System.Reflection;
using System.Text;

public class UIRebuildLogger : MonoBehaviour {

    /// <summary>
    /// 是否开启帧模式，
    /// 帧模式下关心每一帧有哪些UI元素触发了Rebuild，
    /// 非帧模式下则关心UI元素触发了多少次Rebuild（建议开启Console的Collapse）
    /// </summary>
    [Tooltip("是否开启帧模式，\n帧模式下关心每一帧有哪些UI元素触发了Rebuild，\n非帧模式下则关心UI元素触发了多少次Rebuild（建议开启Console的Collapse）")]
    [SerializeField]
    bool m_FrameMode;

    IList<ICanvasElement> m_LayoutRebuildQueue;
    IList<ICanvasElement> m_GraphicRebuildQueue;
    StringBuilder sb = new StringBuilder();
    private void Awake()
    {
        Type type = typeof(CanvasUpdateRegistry);
        FieldInfo field = type.GetField("m_LayoutRebuildQueue", BindingFlags.NonPublic | BindingFlags.Instance);
        m_LayoutRebuildQueue = (IList<ICanvasElement>)field.GetValue(CanvasUpdateRegistry.instance);
        field = type.GetField("m_GraphicRebuildQueue", BindingFlags.NonPublic | BindingFlags.Instance);
        m_GraphicRebuildQueue = (IList<ICanvasElement>)field.GetValue(CanvasUpdateRegistry.instance);
    }

    private void Update()
    {
       

        for (int j = 0; j < m_LayoutRebuildQueue.Count; j++)
        {
            ICanvasElement element = m_LayoutRebuildQueue[j];
            if (!ObjectValidForUpdata(element))
            {
                continue;
            }

            Graphic graphic = element.transform.GetComponent<Graphic>();
            if (graphic == null)
            {
                continue;
            }

            Canvas canvas = graphic.canvas;
            if (canvas == null)
            {
                continue;
            }

            string str = $"<color=#ff0000>{element.transform.name}</color>的LayoutRebuild引起<color=#ff0000>{canvas.name}</color>网格重建";

            if (m_FrameMode)
            {
                sb.AppendLine(str);
            }
            else
            {
                Debug.LogError(str);
            }
            
        }

        for (int j = 0; j < m_GraphicRebuildQueue.Count; j++)
        {
            ICanvasElement element = m_GraphicRebuildQueue[j];

            if (!ObjectValidForUpdata(element))
            {
                continue;
            }

            Graphic graphic = element.transform.GetComponent<Graphic>();
            if (graphic == null)
            {
                continue;
            }

            Canvas canvas = graphic.canvas;
            if (canvas == null)
            {
                continue;
            }


            string canvansName = canvas.name;
            if (canvansName == "DebugFps")
            {
                continue;
            }

            string str = $"<color=#ff0000>{element.transform.name}</color>的LayoutRebuild引起<color=#ff0000>{canvas.name}</color>网格重建";

            if (m_FrameMode)
            {
                sb.AppendLine(str);
            }
            else
            {
                Debug.LogError(str);
            }
        }

        if (sb.Length != 0)
        {
            Debug.LogError($"当前帧<color=#66ccff>{Time.frameCount}</color>:\n{sb.ToString()}");
            sb.Clear();
        }
    }

    private bool ObjectValidForUpdata(ICanvasElement element)
    {
        bool valid = element != null;
        bool isUnityObject = element is UnityEngine.Object;

        if (isUnityObject)
        {
            valid  = (element as UnityEngine.Object) != null;
        }

        return valid;
    }
}

```



# Rebatch

Rebatch指以Canvas为单位的UI网格合并，其在Profiler中体现为Canvas.BuildBatch项，Rebuild通常都会引发Rebatch，UI的位置移动也会触发Rebatch

因此对于需要频繁改变的UI，就需要通过子Canvas与不变的UI分开，以降低改变每次改变UI导致的Rebatch开销，这也被称之为动静分离

但是不同Canvas下的UI是不能合批的，按照Unity官方的说法，在开启了**多线程渲染**的情况下，已经不建议再做动静分离了

# Overdraw

在渲染管线中，对于不透明物体，其绘制顺序都是从前往后绘制，这样被挡住的物体就不会再被绘制，继而节省了GPU渲染开销

而半透明物体则是从后往前渲染，因为半透明物体需要进行颜色混合，从前往后渲染会导致在一个半透明物体遮挡另一个半透明物体时，颜色混合结果不正确

Overdraw即过度绘制，意为一个像素被反复绘制了多次，这种情况通常发生在半透明物体的渲染中

在实际开发中，为了实现一个点击弹窗空白区域关闭弹窗UI的效果，往往会使用一个完全透明的Image放在弹窗底下，而由于Unity中所有UI都是作为半透明物体进行渲染的，这样做就会造成Overdraw现象

![](https://cathole-1307936347.cos.ap-guangzhou.myqcloud.com/UIOptimizatioOfMMO/UIOptimizatioOfMMO_7.png)

解决方案是通过继承Image组件重写OnPopulateMesh方法，将顶点信息清空

```
using UnityEngine;
using System.Collections;

namespace UnityEngine.UI
{
    public class Empty4Raycast : MaskableGraphic
    {
        protected Empty4Raycast()
        {
            useLegacyMeshGeneration = false;
        }

        protected override void OnPopulateMesh(VertexHelper toFill)
        {
            toFill.Clear();
        }
    }
}
```

# Instance

在打开一个诸如背包这样有很多Item的界面时，通常会感觉到明显的卡顿

其原因主要在于一次性实例化了过多GameObject，而Unity中GameObject的实例化只能在主线程进行，一次性实例化太多就会卡住主线程的执行

解决方案就是通过定时器实现分帧加载与实例化，这样可以做到对背包之类的多Item界面打开时玩家会有很丝滑的体验
