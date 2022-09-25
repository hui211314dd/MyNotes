# 一键生成动画-育碧蒙特利尔关于Mocap的自动化技术

## 前言
在[MotionMatching 中的代码驱动移动和动画驱动移动](https://zhuanlan.zhihu.com/p/432663486)的Creative章节中，Daniel Holden提到了Dan Lowe的本次演讲，可以借助自动化工具对动捕数据进行调整以符合SimulationObject的要求，后来写完[UE5中的MotionMatching-实践(一)](https://zhuanlan.zhihu.com/p/557039076)后想着使用动捕数据作为Database, 如果想使用动捕数据那么必须面临数据一致性的问题，就涉及到动捕数据如何高效地编辑了，这时候觉得有必要研究下Dan Lowe的这篇演讲了。

阅读本文前建议会简单操作MotionBuilder

本次演讲的[视频链接](https://www.gdcvault.com/play/1023477/Animation-Bootcamp-The-Animate-Button)以及[PPT链接](https://www.gdcvault.com/play/1023114/Animation-Bootcamp-The-Animate-Button)

Dan Lowe目前在圣莫尼卡工作室，专注于《战神:诸神黄昏》的开发，今年他的职位也提升到了Principal Tech Animator，他经常活动于[推特](https://twitter.com/danlowlows)，并且经常探讨游戏动画相关的技术，他的[油管频道](https://www.youtube.com/user/danlowe1983)有一些3A游戏的动画技术分析视频，还有MotionBuilder相关的教程，他的[Github](https://github.com/Danlowlows)上有本次的脚本代码，大家可以关注学习。

## 正文
大家好，我是Dan Lowe，目前在育碧蒙特利尔担任动画TA，正如下图看到的，参与了很多游戏的开发。我知道不同的公司对于动画TA有不同的定义，基本上我每天大部分的时间跟其他动画师一样在参与动画的制作，但是我也会拿出一些时间做一些技术上的工作比如状态机，设计动画工具以及管线流程，确保每个人都在以最佳方式工作。我目前的工作是在为动画师做一些自动化工具，这也是接下来我们要讲的。本次演讲的内容主要是我和Zach Hall完成的。

![Dan参与的游戏](.\TheAnimateButtonPic/1.png)

那么什么是“一键生成动画”呢？我们通过一个视频来介绍下

{一键生成动画.mp4}

当然了视频里看到的仅仅是个玩笑，但我们想到是否可以将这种思想用来处理Mocap数据呢，我们是否可以让计算机帮助我做一些事情呢？这里可以先看下我们的自动化工具，放进去原始的动捕数据以及输入一些参数就可以生成可交付的动画。

![自动化工具](.\TheAnimateButtonPic/2.png)

这个工具我们称之为Automator, 在右上角你可以看到“Animate”的按钮，这里有一些戏谑的成分，在演讲的最后我会使用这个工具做个演示。

本次演讲主要分三个部分：

* Mocap for automation, 如果一开始我们的动捕数据就很糟糕，那么数据处理会更加困难，所以这里会讲一下育碧在动捕时的一些技巧从而帮助我们获得高质量的动捕数据

* Adjustment blending, 这里将会讲到一个核心技术“Adjustment Blending”, 正是因为这个技术让后面的自动化成为了可能

* Automation tools, 最后我会说下刚才提到的Automator工具，这个工具把所有的提到的融合到了一起
  
## Mocap for automation

![动捕各个阶段对动画质量的影响程度](.\TheAnimateButtonPic/3.png)

柱状图显示了在动捕各个阶段动画数据质量等级的变化，正常情况下，每一个处理步骤都会损失一点质量，然后 
我们的动画师试图通过动作编辑将这些质量找补回来。

![演员的舞台表演](.\TheAnimateButtonPic/4.png)

我们从舞台上的演员开始，就纯力学而言，这基本上是完美的, 因为我们处理的是一个真实的演员的表演以及真实世界的物理学。

![通过MarkerData还原数据](.\TheAnimateButtonPic/5.png)

然后，系统将其转化为标记数据，我们通常会在这里损失一些质量，因为标记数据需要被重建。从技术上讲，如果我们考虑肌肉和皮肤之类的话，我们确实在这里失去了大量的信息。但我们不用管这些，我们只谈论光学动捕以及重建光学标记数据。

![重定向到身体上](.\TheAnimateButtonPic/6.png)

然后，我们通常会将这些标记重定向到rigid bodies上，这个阶段我们同样会失去一些质量。

![再次重定向](.\TheAnimateButtonPic/7.png)

接下来我们需要再次的重定向，从rigid bodies到游戏的rig上，由于mocap actor与game character**比例差异**的缘故，我们依然会在这里丢失大量动画数据精度。

![动画编辑](.\TheAnimateButtonPic/3.png)

最后我们进行动作编辑，我在这里打了个大问号的原因是你在这里得到的质量完全取决于你的动画团队的能力。我们通常不得不以不同的方式约束数据，以适应我们的游戏系统，因此在这个过程中我们往往会失去原始动画数据的一些有趣的东西，但另一方面，在编辑时我们可以增加更有力的姿势，我们可以夸张一些以及尝试增加一些更具吸引力的东西，因此质量实际上可能在这里会得到提高。

但对于一些管线或者个别动画师来说，往往看到的是这样...

![管线处理过程中出现的糟糕情况](.\TheAnimateButtonPic/8.png)

可以看到，动画师做了一个非常基础且低质量的重定向，然后立即Bake数据并开始做动画，因为他们期望在动作编辑阶段来解决所有的重定向问题。我想说的是，这是一个很差的做法，因为用这种方式解决问题会费时更多。如果你在重定向阶段解决了一个问题，那么在处理其他动捕数据时这个问题已然不会再存在了，但如果你做了一个糟糕的重定向，然后进行Bake再编辑，你必须单独修复所有这些问题。通过自动化工具，我们希望事情看起来更像这样...

![通过自动化工具期望得到的结果](.\TheAnimateButtonPic/9.png)

我的意思是，对于任何管线，你都希望它看起来像这样，但它对自动化尤其重要，因为...

![动画编辑阶段很难得到提升](.\TheAnimateButtonPic/10.png)

夸张和有吸引力的Pose是很难自动化来做的...所以说很难在动画编辑阶段有质量的提升。

此外，对于重定向增加的每一个误差，如果我们是自动化处理，我们必须写一些脚本来清理这些。因此，一开始引入的误差越少，以后的工作就越容易。

![认真对待每一个细节](.\TheAnimateButtonPic/11.png)

因此，这没有什么诀窍，这真的只是长时间所学到的经验。并对每一步的质量进行严格的审查。这里再列举下一些重要的点，正如你看到的，相关人员应该很熟悉这些，但我觉得还是有必要再谈下，因为这些真的很重要

### Hire professionals
雇佣专业的演员和特技人员进行拍摄。这件事可以帮助我们获得高质量的动捕数据

### Plan and rehearse
你应该提前计划和排练好你的拍摄。拍摄Mocap本身是很昂贵的，所以如果你提前花时间进行了计划和排练，你就能在当天完成更多的拍摄。

而作为排练的一部分，我们有时会聘请动作教练。下面这个人是Terry Notary，他是一个动作指导，曾与我们合作过《Far Cry: Primal》，他还是《人猿星球》、《霍比特人》、《阿凡达》和其他一些电影的动作指导。

![Terry Notary](.\TheAnimateButtonPic/12.png)

### Capture face, body and voice at the same time
我们同时捕捉面部、身体和声音。在剧情方面使用动捕很好理解，但我们正试图在gameplay方面也越来越多地这样做。

### Refine your marker setup  
改善你的marker设置。你的marker对你的重定向结果有很大的影响，所以值得花一些时间尝试不同的设置，以确保你的设置是最佳的。

这里顺便提一件事情，就是我们采集身体数据的同时也采集了手指相关的数据，这其实并不是一个标准的做法。我们所做的是一个近似操作，我们没有做**完整**的手指捕捉。我们使用两个marker。一个在食指末端，一个在小指末端，正如下图看到的情况，我们创建三个预设的手部姿势：一个是完全张开的手，一个是放松的手，还有一个是握拳的手，然后我们用Mocap marker来驱动这些姿势

![Hand Marker](.\TheAnimateButtonPic/13.png)

正如下面视频看到的，这就是原始的动捕数据，效果虽称不上完美但却是个不错的开始！

{手部动捕的效果.mp4}

正如我之前所说的，重定向出现很多问题的主要原因之一是Mocap演员和游戏角色之间存在比例差异，所以我们尝试使角色的比例匹配Mocap演员的比例，因此，我们的做法是从拍摄现场的视频中截取一段，然后叠加上游戏角色，这使得我们很容易发现任何问题，然后我们与角色团队反复迭代角色的比例，直到得到我们满意的结果。当然随之可能还存在一些差异，不过这对于重定向后数据的质量有了很大的影响。

因此，结合所有这些Mocap技巧，我们得到的Mocap数据是非常高质量的，这减少了很多不必要的任务。我想重申一下，这对于获得可交付的高质量数据非常重要。

{角色等比例.mp4}

一旦我们有了这些数据，我们现在必须处理它，这就不得不提到下一个话题...

## Adjustment blending

正如视频里看到的，我们通过动捕获得了角色原地转身的动画数据，这对于动画师来讲特别熟悉，对数据的clean up是他们每天都会面对的工作, 经常做的就是，在MotionBuilder中创建一个layer, 在片段的开始和结尾处分别粘贴上consistent pose, 仔细看效果的话，可以看到产生了滑步。为什么会这样呢？点开脚部的Curve我们就可以看到，那是因为整个动画期间Pose一直在变化。如何解决呢？动画师往往会手动调整这些Curves, 以便当脚部移动时，我们的Pose才让改变，但这种手动调整的工作往往特别棘手，因为它必须对每个正在滑步的effector都要调整，这还仅仅只是一个简单的例子，在其他情况下可能会涉及到多次需要补偿的FootSteps.

{手动调整Curves避免滑步.mp4}

接下来演示下我们的Adjustment Blending tool是如何完成这些工作的...

{AdjustmentBlendingTool自动调整Curves.mp4}

可以看到只需点点点就自动完成了！

接下来我们看另外一个例子，一个角色从跑步状态到停下来...

{角色跑步到停下来存在滑步问题.mp4}

首先要在动画的开头Key一个Zero，这样就能保持我们的跑步姿势，然后在动画的结束部分粘贴一个consistent pose。随后游戏相关人员过来告诉我们希望停步距离保持在3m左右，我们切到动画开头，把位置拖动到相应位置并Key帧，播放后发现不仅仅脚在滑步，当角色站定时Hips也在滑动。

我们再试试Adjustment Blending tool...

{自动化工具自动调整停步动画.mp4}

它解决了所有的滑步问题，我们现在得到了一个很棒的停步动画！这其中发生了什么呢？

![停步时发生了什么](.\TheAnimateButtonPic/14.png)

**我们想的是当动画处于运动状态时我们悄无声息地进行adjustment, 当角色静止时不做任何adjustment**, 例如下图中脚站定时，无需adjustment

![脚站定时无需adjustment](.\TheAnimateButtonPic/15.png)

接下来的讨论会涉及到一些数学知识，不过放心，都是很简单的，做这些讨论是为了你们可以在自己的项目中独立实现。即使没有听懂也不用担心，后面我会把相关算法放到网上。

正如前面所说，我们需要找出角色是在哪些区间移动的，对于base layer中的每个curve,我们计算每帧之间的改变量(注: 绝对值)...

![每帧之间的改变量](.\TheAnimateButtonPic/16.png)

然后将这些改变量相加求和：

![改变量求和](.\TheAnimateButtonPic/17.png)

随之将每帧的改变量通过Total变换成百分比数据

![改变量变换为百分比数据](.\TheAnimateButtonPic/18.png)

然后我们把这些百分比数据映射到Curve上，这条Curve长下面这个样子...

![百分比数据映射成Curve](.\TheAnimateButtonPic/19.png)

我们称这条Curve为Motion Delta, 通过它我们可以轻易看出哪里角色在移动，程度如何，以及哪里在静止，比如陡峭的地方表明在运动，平坦的地方表明在静止。

(注：还记得最开始我们创建了一个单独的Layer并且在该Layer进行了Key值吗？下图就是新建Layer的Curve)

![修正前的Curve](.\TheAnimateButtonPic/20.png)

然后在新增的Layer上，我们进行修改，只需保证帧之间的百分比与上面求出的百分比相同即可，自动计算并调整后得到如下的结果:

![修正后的Curve](.\TheAnimateButtonPic/21.png)

这样就已经完成了！然后我们再对其他Curves重复同样的过程。我们通过比较图直观看下修正前后的比较，可以发现通过修正只有当角色运动时Curve数值才会有变化。

![修正前后的比较](.\TheAnimateButtonPic/22.png)

这个方法相当棒，因为只对Curves做一些数学的操作，所以可以很方便地移植到其他DCC工具中，同时也可以在game runtime中实现，我们稍后再聊这个话题。

![可以移植到其他DCCs中](.\TheAnimateButtonPic/23.png)

我刚才所描述的这种方法目前运行得很好，但肯定也有一些的问题和局限性...

### Hyper Extension

![腿部过度伸展](.\TheAnimateButtonPic/24.png)

任何过大的调整都会导致四肢过度伸展, 显然人类的步幅是有限制的，如何解决呢？ 首先我们必须查明在哪些帧上有过度拉伸的问题，这是很容易做到的，因为我们知道我们的四肢应该有多长

![腿部伸展长度](.\TheAnimateButtonPic/25.png)

然后我们在两个脚踝之间找到一个点，这个点的位置取决于我们每条腿有多少过度拉伸。因此如果我们一条腿有大量的拉伸，而另一条腿没有拉伸，那么这个点将更靠近有大量拉伸的那条腿的脚踝，我们将Hips向该点移动，直到我们得到满意的结果。

![调整Hips解决拉伸问题](.\TheAnimateButtonPic/26.png)

下面这张图展示了调整前后的比较:

![拉伸问题前后的比较](.\TheAnimateButtonPic/27.png)

### Apply to IK effectors for locked contacts



## Automation tools

## 我的总结

## Bonus!

脚本地址

https://zhuanlan.zhihu.com/p/34260822

https://zhuanlan.zhihu.com/p/34330297

DeltaCorrection

MotionWarping

## 不明白的点
* Layer原理
* Adjustment Blending核心算法
* MotionBuilder FullBodyIK在Adjustment Blending中充当的作用
* Best when adjusting existing motion什么意思，后面提到的比较amount怎么理解和操作？

# 单词
ridiculous  
suspense
tease
occlude
muscles
optical
exaggerate
appeal
throughout 
anally 
retentive
reiterating
in advance
rehearsal
a bunch of
proportion
disparities
reiterate
compensated
definitely
be aware of
Hyper
detect
be supposed to be