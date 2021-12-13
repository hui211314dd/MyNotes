## [【UE4】给Animation加一点料（一）- Motion Matching篇](https://zhuanlan.zhihu.com/p/315908910)

关于AnimNode的代码以及Trajectory很有参考价值。  
关于Cost的计算是用矩阵进行计算而且并没有做加速结构的处理，Tag的使用并没有过多介绍。

后面的学习过程中需要关注的问题:  
* Cost的计算过程，以及Trajectory如何参与到Cost计算中去。  
* 为什么会有RootMotion From Everything这样的问题呢？  
* 最最重要的是工作流。  
* Tag的应用场景。
* [Simon Clavet的演讲仍然是最主要的资料](https://www.gdcvault.com/play/1023280/Motion-Matching-and-The-Road)


## Motion Symphony
目前来看，Motion Symphony的完整程度应该是最高的，编辑器，加速结构，参数调整。在我看来不足的就是RootMotion触发运动的(但是可以自己改造下)，还有就是可以调节的参数太多了，对于新手来讲，有点费劲。(主要原因是我先看代码后看文档)，也是主要的学习资料吧。  
[官方文档](https://www.wikiful.com/@AnimationUprising/motion-symphony)

* ### Trajectory

* ### Trajectory Error Warping

* ### Motion Data

* ### Motion Match Config

* ### MMOpimisation MultiClustering
UMMOptimisation_MultiClustering


* ### Motion Calibration
weight的配置，用于如何计算cost。  

* ### Mirroring Profile

* ### Tag  
    跟AnimNotify类似，Tag也分为Action和Section，Section常用。  
    * DotNotUse 对于不喜欢的或者重复的或者脏的动画片段给与标注，Runtime不考虑这些片段。
    * Trait Tags  不仅可以对整个动画进行标注也可以对动画片段进行标注，Trait可以对动画进行分类(可以按照状态，比如走跑跳，受伤战斗等)，在MotionMatching阶段可以传入想查询的TraitSet，这样MotionMatching在Runtime查询时只查询标注有TraitSet的动画片段。
    * Cost Multiplier Tags Cost系数，如果你希望某个片段倾向于被选中，可以设置一个小于0的系数，否则可以设置一个大于0的系数(因为MotionMatching是选中Cost最小的那个)。
    * Custom Tag根据业务需要自定义Tag。


## todo
需要在荣耀战魂里面试下在墙，边缘处，墙角处的反应，尤其是在墙角处是停下还是直接转向呢？


