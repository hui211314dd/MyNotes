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

接下来我们看另外一个例子，一个角色从跑步状态到停下来。首先要在动画的开头Key一个Zero，这样就能保持我们的跑步姿势，然后在动画的结束部分粘贴一个consistent pose。随后游戏相关人员过来告诉我们希望停步距离保持在3m左右，我们切到动画开头，把位置拖动到相应位置并Key帧，播放后发现不仅仅脚在滑步，当角色站定时Hips也在滑动。

我们再试试Adjustment Blending tool...

{角色跑步到停下来存在滑步问题并且自动化工具自动调整停步动画.mp4}

它解决了所有的滑步问题，我们现在得到了一个很棒的停步动画！这其中发生了什么呢？

![停步时发生了什么](.\TheAnimateButtonPic/14.png)

**我们想的是当动画处于运动状态时我们悄无声息地进行adjustment, 当角色静止时不做任何adjustment**, 例如下图中脚站定时，无需adjustment

![脚站定时无需adjustment](.\TheAnimateButtonPic/15.png)

接下来的讨论会涉及到一些数学知识，不过放心，都是很简单的，做这些讨论是为了你们可以在自己的项目中独立实现。即使没有听懂也不用担心，后面我会把相关算法放到网上。

正如前面所说，我们需要找出角色是在哪些区间移动的，对于BaseLayer中的每个curve,我们计算每帧之间的改变量(注: 绝对值)...

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

Adjustment Blending需要注意的第二个问题是为了获得锁定的触点，你需要将这个过程应用于你的IK Effectors。因为我们在MotionBuilder中工作并且其提供了FullBodyIK Rig, 所以我们不用关心这一点，但如果你想在其他DCC或者Runtime上实现，怎必须考虑这一点。

![MotionBuilder中的FullBodyIK Rig](.\TheAnimateButtonPic/28.png)

### Best when adjusting existing motion

最后一点要注意的是: Adjustment Blending最好是调整BaseLayer中已经存在的动作而不是在动画中添加大量的额外动作, 举例来说我们有一个站立Idle的动画，现在想在AdditiveLayer上Key一个点击按钮的动画

![Idle动画修改成PushButton动画](.\TheAnimateButtonPic/29.png)

由于我们AdditiveLayer上的按钮动作比我们BaseLayer上的Idle大得多，Adjustment Blending将很难找到陡峭的曲线来隐藏按钮的运动。它最终给出了这个非常糟糕，抽搐且断断续续的运动，如下面视频演示的那样：

{PushButton断断续续.mp4}

解决方法也很简单，还记得我们步骤中曾经计算过这个值吗？

![BaseLayer中的Total](.\TheAnimateButtonPic/30.png)

如果AdditiveLayer中的这个值远远大于上面的值，说明不适合使用Adjustment Blending，这时候默认的Linear Curve就能获得更好的结果

(由于视频数量限制，下篇再介绍Adjustment Blending以及Automation Tools)

## Adjustmeng Blending At Runtime

我之前提到过，我们同样可以在Runtime时做这个，现在还处在比较早期的阶段，所以我想谈谈它潜在的用途。

我们现在假设游戏中的角色跑步然后停下来，那么角色经历的动画分别是Run, Stop以及Idle, 就是这么简单的逻辑就会碰到许多挑战...

![Run-Stop-Idle](.\TheAnimateButtonPic/31.png)

首先我们必须保证动画接缝处poses match...

![接缝处动画帧要一致](.\TheAnimateButtonPic/32.png)

我们也不知道当角色得到停止的信号时我们正将处于什么姿势，所以我们通常会在这个时候看到一些混合的问题

![没有人能知道玩家想在什么时候停下来](.\TheAnimateButtonPic/33.png)

再有，如果角色是AI，我们经常希望我们的AI能够准确停在寻路路径的末端点上，因此我们通常要做一些距离或转弯的修正，这就造成了滑步，这是我们不希望的。

![为了精准控制位置在Stop时不得不实时修正](.\TheAnimateButtonPic/34.png)

Adjustment Blending如何处理这个问题呢？我们知道角色打算停步时那一帧的动画，把该帧动画提取出来.

![从Run动画中提取帧](.\TheAnimateButtonPic/35.png)

我们再看一下我们将要融入的Idle Pose，我们同样也提取这个姿势

![从Idle动画中也提取帧](.\TheAnimateButtonPic/36.png)

然后在Runtime时，我们使用Adjustment Blending算法生成一个Additive Layered动画，然后在Stop上播放，它可以修正姿势...特别棒的主意！

![Runtime使用Adjustment Blending的原理](.\TheAnimateButtonPic/37.png)

使用Adjustment Blending算法的好处是Idle Pose是被提取出来的，所以我们不再需要在所有的基础数据中都有一个一致的Idle Pose

![不需要专门的Core/Consistent Idle Pose了](.\TheAnimateButtonPic/38.png)

因此我们可以有一堆不同的Idle Animations，当准备停步时可以从中随机一个，一旦选择好StartTime后，Adjustment Blending会帮我们处理好后面的工作

![可以很自然地停到不同的Idle动画上](.\TheAnimateButtonPic/39.png)

正如下面的视频里看到的，5个角色停步时分别过渡到不同的Idle Pose上，看起来特别自然，没有任何滑步。所有的角色使用了相同的Stop动画，Adjustment Blending将他们推向了不同的Idle上。

{5个角色停步到不同的IdlePose上.mp4}

我们甚至可以在Blend之前改变IdlePose在世界坐标的位置和旋转，我们可以得到一定程度的距离修正和旋转修正

![可以重新改变IdlePose在世界坐标的位置](.\TheAnimateButtonPic/40.png)

下面这个视频展示了5个角色停步到了不同朝向的位置，表现很自然，同样没有滑步。所有角色使用了相同的Stop动画

{5个角色停步到不同朝向位置的IdlePose上.mp4}

这让我想到通常情况下，我们的角色是通过播放这些短动画来移动的，动画片段之所以尽可能短是因为我们的系统需要动态变化。但是如果你有非常好的修正比如Adjustment Blending给予的那样，那么对于AI来说，如果我们提前知道AI角色要去哪个地方，实际上可以始终保持在同一个动画中，然后使用Adjustment Blending来动态修正

所以我的意思是，我们可以使用长动画来代替短动画，Adjustment Blending修正使之成为可能

![使用长动画来代替短动画](.\TheAnimateButtonPic/41.png)

我们可以看下面这个视频，视频中的角色从一个掩体移动到另一个掩体附近，这是个几乎没有修改过的长动画，而且整个运动特别流畅，合理的加减速以及跟环境正确的交互

{从一个掩体移动到另一个掩体的动捕长动画.mp4}

但是，如果要放入游戏内的话，通常的做法是将这个长动画切成若干个短动画，比如掩护退出动画，然后进入标准的locomotion，最后还有一个掩护进入(Entry)动画

![将长动画切成3个短动画-42](.\TheAnimateButtonPic/42.png)

我们通过下面的视频看下实际的效果，可以看到效果有些差强人意，标准的locomotion放在这里并不合适，而且在融合时不仅有些生硬，而且在进入掩护状态时为了修正位置出现了滑步的问题

{标准做法导致生硬且滑步的问题.mp4}

我们再回过头来看我们的动捕动画，这明显才是我们想要的效果呀...只不过当目标掩护位置发生改变的时候，动画就不准确了...

![实际期望位置经常与动画实际位置不同-43](.\TheAnimateButtonPic/43.png)

就像之前我们做的那样，把目标位置拉到期望位置后，运行Adjustment Blending。通过下面的视频可以看到，动画质量几乎跟源数据一样不过位置已经修正到了我们期望的位置，效果自然且没有滑步。因此在一定的阈值范围内，我们可以纠正目标位置的这种差异，这使得我们的掩护系统能够在不同的情况下发挥作用。

{通过修正角色跑到了我们期望的位置.mp4}

我们也可以在一定程度上绕过障碍物, 我们从路径上采样并且指定目标位置后，运行Adjustment Blending

{修正后可以绕过障碍物.mp4}

你必须建立一个网络，每个格子代表一个动画，比如你想通过掩护到下面这个位置，那么就必须找到离这个位置最近的动画，然后通过修正达到我们的目的。这可能看起来是一个很大的覆盖面，但这种东西在mocap阶段是超级容易拍摄的，因为我不必担心脚步匹配或任何通常的技术限制，我只需设置好掩护物体，然后告诉演员，我需要你从这里走到这里，我不关心演员怎么做，我只需专注于表演即可。

![找出最近的动画-44](.\TheAnimateButtonPic/44.png)

你仍然需要一个常规的locomotion system作为保底方案，因为角色总是会有中断或者必须做很多转弯的时候，比如说如果你正在制作一个有很多走廊的游戏或者一个障碍物非常密集的环境，那么Adjustment Blending或许不是最好的选择。但是在一个有相对开放空间的游戏中，我认为Adjustment Blending会有一个真正的质量提升。

这就是Adjustment Blending。

我们在《Far Cry: Primal》中大量使用了它的离线版本，它确实有助于提高我们动画团队的工作效率。问题是Adjustment Blending并没有什么特别聪明的地方，我们的理念是："做我们的动画师已经在做的事情，只不过写一个脚本来自动完成"。于是我们开始思考，如果我们采用同样的理念并将其应用于整个动画编辑pipeline，会怎么样？

## Automation tools

我们大概估计Adjustment Blending为我们所有的动画师节省了大约15%的时间，这听起来可能不是很多，但要知道育碧有很多动画师在做清理大量mocap的工作，所以这15%代表了大量的时间和精力被节省。随后我花了时间来复盘整个mocap pipeline，试图更好地了解我们作为mocap动画师的效率有多低，然后研究我们如何通过自动化任务来提高生产力。

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
subtle
steep
appropriate
estimate
proposed this mandate
inefficient
investigate