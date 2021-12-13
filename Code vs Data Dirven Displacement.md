# Motion Matching Code vs Data Driven Displacement

## 前言
这是育碧研究员Daniel Holden九月份的一篇文章，讲述了Motion Matching中Code Driven Displacement 和Data Driven Displacement的使用以及权衡，思路清晰，抽丝剥茧，视频，图片，代码应有尽有，我单方面宣布这是我见过所有介绍Motion Matching最好的一篇文章了！

Daniel研究方向是机器学习和角色动画，有兴趣的可以去关注[他的博客](https://theorangeduck.com/page/code-vs-data-driven-displacement)。

如果你对Motion Matching感兴趣，可以看下我的其他文章。

[《荣耀战魂》中的Motion Matching](https://zhuanlan.zhihu.com/p/401890149)  
[《最后生还者2》中的Motion Matching](https://zhuanlan.zhihu.com/p/403923793)  
[《Control》中的Motion Matching](https://zhuanlan.zhihu.com/p/405873194)  
[游戏开发中的Pose Matching](https://zhuanlan.zhihu.com/p/424382326)  

## 正文
Code VS Data Driven Displacement在游戏开发中一直是一个争论的问题，动画师和Gameplay设计师都有各自不同的意见。在我看来，理解其中的细节问题比单纯使用某种特定办法要重要的多，这样你可以跟动画师和Gameplay设计师正确地去沟通交流这些选择能带来什么，有哪些优缺点。  
那么Code VS Data Driven Displacement具体指什么呢？其实这个问题可以换一种问法，就是如何在游戏世界内移动一个角色并且同时保证真实和即时反馈？    

本文的目录如下:   
[Definitions](#definitions)  
[Simulation Bone](#simulation-bone)  
[Character Controller](#character-controller)  
[Synchronization](#synchronization)  
[Adjustment](#adjustment)  
[Clamping](#clamping)  
[Be Creative](#be-creative)  
[Sliding and Foot Locking](#sliding-and-foot-locking)  
[Move the Camera](#move-the-camera)  
[Source Code](#source-code)    

### Definitions   
开始时我们先解释一些相关的专有名词，方便后面的理解。  
#### The Simulation Object
Simulation Object是一个physical object(UE4对应的就是CapsuleComponent，其他解决方案有的则是Circle), 这个是Character的一个运动代理，用来负责跟物理世界进行场景交互，常见的就是墙壁阻挡运动，挤着墙壁滑动等，这都是Simulation Object来负责的。  
一般来说，Simulation Object的运动原理是这样的，通过操作手柄，根据手柄的方向和力度映射成一个期望的速度向量，把速度向量设置给角色。对于这些离散的速度向量，我们使用临界阻尼进行平滑，临界阻尼的效果既能满足运动的及时反馈又能满足我们期望的关于Character的速度变化。(注:Daniel在[另外一篇文章](https://theorangeduck.com/page/spring-roll-call)详细介绍过临界阻尼以及推导，《荣耀战魂》Motion Matching的演讲人Simon Clavet在[油管上也分两个视频讲过](https://www.youtube.com/watch?v=RGOUPe-GfCk)，知乎上大佬sincezxl[也完整推导过](https://zhuanlan.zhihu.com/p/412291354))  

(视频1 SimulationObject)  

上面的视频可以看到我们使用手柄控制了simulation object,我们可以通过调整面板上的参数来设置反馈灵敏度和角色速度，正如上面所说的，我们是通过临界阻尼弹簧进行速度模拟的，当然你也可以按照你的想法进行修改(可以看到运动是跟动画资源无关的，这完全是由代码控制的，这就是Code Driven Displacement!)下面的代码展示了从摇杆输入到目标速度的映射过程，同时支持前后运动，侧向运动等。  
```
vec3 desired_velocity_update(
    const vec3 gamepadstick_left,
    const float camera_azimuth,
    const quat simulation_rotation,
    const float fwrd_speed,
    const float side_speed,
    const float back_speed)
{
    // Find stick position in world space by rotating using camera azimuth
    vec3 global_stick_direction = quat_mul_vec3(
        quat_from_angle_axis(camera_azimuth, vec3(0, 1, 0)), gamepadstick_left);
    
    // Find stick position local to current facing direction
    vec3 local_stick_direction = quat_inv_mul_vec3(
        simulation_rotation, global_stick_direction);
    
    // Scale stick by forward, sideways and backwards speeds
    vec3 local_desired_velocity = local_stick_direction.z > 0.0 ?
        vec3(side_speed, 0.0f, fwrd_speed) * local_stick_direction :
        vec3(side_speed, 0.0f, back_speed) * local_stick_direction;
    
    // Re-orientate into the world space
    return quat_mul_vec3(simulation_rotation, local_desired_velocity);
}
```

#### The Character Entity
Character Entity就是我们玩家看到的那个角色的实体，包括位置和旋转信息。  
不同于上面所说的利用阻尼弹簧去控制运动的Logic, 这里更倾向于用动画里的运动数据去控制移动，前者会造成滑步(注:总所周知，滑步主要就是运动跟动画不匹配导致的)，后者则表现更为真实。通过动画数据驱动运动的方式就是Data Driven Displacement。  

(视频2 MotionCaptureData)

上面的动画数据来自[育碧开放的动画库里](https://github.com/ubisoft/ubisoft-laforge-animation-dataset)，地面的红点就是Character Entity的位置和旋转信息。  
你给我等下...这个动画库仅仅提供了角色身上骨骼的位置和旋转，Character Entity的位置和旋转来自哪里呢？这就引出了我们要讲的第一个很重要的概念: the Simulation Bone。

### Simulation Bone
Simulation Bone是我们添加的一个额外的骨骼，用来存储角色在动画中的位移和旋转信息。Simulation Bone可以通过人工，或者完成程序生成，或者两者的混合来制作([《最后生还者2》中有视频展示了在Maya调整的过程](https://www.gdcvault.com/play/1027378/Motion-Matching-in-The-Last))，确定方案后，我们通常会把它做成RootBone。

这里一定要牢记的是，Character Entity的运动和Simulation Object的运动应该尽可能地一致，包括速度，加速度，减速度以及转向速度等。(注：常说的数据一致性)  
基于这点，这里说下我对于程序生成Simulation Bone的心得：取一个upper spine骨骼的位置信息垂直投射到地面上，它的position就是Simulation Bone的position信息。我们使用[Savitzky-Golay filter](https://en.wikipedia.org/wiki/Savitzky%E2%80%93Golay_filter)进行smooth out;将hip bone的forward direction投射到地面上，使用相同的filter进行smooth out,将这个forward direction和一个vertical axis合并生成一个rotation,这个rotation就是Simulation Bone的rotation信息。下面视频展示了我们按照此方法生成的Simulation Bone信息。(注：关于更多Savitzky-Golay filter的学习，可以[参考图形学大佬Bart Wronski这篇文章](https://bartwronski.com/2021/11/03/study-of-smoothing-filters-savitzky-golay-filters/))  

{视频3 MotionCaptureDataWalking}  

这种方法背后的思想是通过平滑移除一些Hips随意抖动带来的噪声点，而且这种方法更匹配临界阻尼的运动风格，不过需要注意的是这种方法对于Loop Animation或者已经被拆成段的短动画效果不是很好，所以这种办法比较适合Start和Stop都是标准Pose的长动画。我还有一个建议就是：程序生成Simulation Bone后，一定要有“手动二次调整”的选项。  
Simulation Bone对于后面Character Entity和Simulation Object的同步至关重要，我们后面再说。现在我们可以给这些生成好的动画数据做一些有意思的数据分析，来看下这些数据跟Simulation Object的运动匹配的如何。  
举例，我们用上面视频中提到的walk动画和按照dance card动捕的run动画做一些数据分析，我们选取Simulation Bone 1s的trajectory长度，站在原点，朝向脸部方向：  

![mm](.\CodeVSDataPic/trajectory_distribution_data.png)

每条线段代表了动画数据内一条不同的trajectory，这些数据涵盖了不同的运动形态包括turns，strafes，backward以及这些形态对应的速度等。  
可以看到，运动数据表明运动反馈并没有那么及时，running的最大速度值为4m/s, sideways strafing的速度不到3m/s, backward running速度在2m/s左右，同样的，walk的速度在1.75m/s左右，sideways速度在1.5m/s, backward walk速度在1.25m/s。  
我们把Simulation Bone的velocity, acceleration, angular velocity做成柱状图用来分析更有意思。通过看maximum acceleration 和 angular velocity的分析可以看出动画数据关于“及时反馈”的程度，看velocities可以看出动画内速度的分布情况。  

![mm](.\CodeVSDataPic/velocity_distribution_data.png)  

如果我们想看下Simulation Object的运动情况，可以录制一些Simulation Object随意运动的动画进行分析，通过图表进行分析他们是否相似。  

{视频4 SimulationDanceCardFast} 

通过上面的录制，我们可以一块比较下: 

![mm](.\CodeVSDataPic/trajectory_distribution.png)  

![mm](.\CodeVSDataPic/velocity_distribution.png)  

可以看到上面大部分区域都重叠了，说明Simulation Object Behavior和 the Simulation Bone of Character Entity匹配的相当不错，这就意味着，如果设置合理的话，两者匹配后的视觉效果将会非常不错。问题是，如何在游戏内实现呢，我们需要先构建一个Character Controller...  

### Character Controller
不论你创建的是何种CharacterC ontroller，ode-vs-data-driven的问题依然是无法避免的，我们基于[这篇paper](https://theorangeduck.com/page/learned-motion-matching)创建一个character controller来展示不同驱动方式下character不同的表现。  
当然了，这里讨论的技术是通用的，你可以应用到任何风格的character controllers。我们的目标其实是"solving this disconnect between the character entity and the simulation object"，我们希望Character Entity和Simulation Object能够建立起一个隐形的连接，保持行为一致。不过刚开始我们可以尝试下什么都不管-Simulation Object按照自己的Code进行驱动，Character Entity按照动画数据进行驱动。或者可以多一点点精准同步的工作，那就是Trajectory的数据是通过Simulation Object的future trajectory相对于Character Entity的偏移计算出来的，这样有个好处，就是MotionMatching能够保证Character Entity找到的动画会一直向Simulation Object的目标方向上靠拢，不至于偏离越来越大。  
```
// Compute the query feature vector for the current 
// trajectory controlled by the gamepad.
void query_compute_trajectory_position_feature(
    slice1d<float> query, 
    int& offset, 
    const vec3 root_position, 
    const quat root_rotation, 
    const slice1d<vec3> trajectory_positions)
{
    vec3 traj0 = quat_inv_mul_vec3(root_rotation, trajectory_positions(1) - root_position);
    vec3 traj1 = quat_inv_mul_vec3(root_rotation, trajectory_positions(2) - root_position);
    vec3 traj2 = quat_inv_mul_vec3(root_rotation, trajectory_positions(3) - root_position);
    
    query(offset + 0) = traj0.x;
    query(offset + 1) = traj0.z;
    query(offset + 2) = traj1.x;
    query(offset + 3) = traj1.z;
    query(offset + 4) = traj2.x;
    query(offset + 5) = traj2.z;
    
    offset += 6;
}
```
下面的视频可以看到，红点标记的trajectory就是被MotionMatching选中动画的trajectory。  

{视频5 DoNothing}  

上面视频可以观察到，当Character跟Sumulation Object偏差太大时，Character会匹配到一个faster或者slower的动画去追Simulation Object。  
尽管上面示例效果上很难令人满意，但却是一个很好的例子，它很好地展示出了当我们什么都不管的时候，Simulation Object和Character Entity表现的样子。  
不管怎么说，上面的效果肯定达不到我们游戏标准的，因为玩家不能**精准地控制角色**，有的时候你会发现character会偏离位置(可能会走到不曾设想的道路上)，最后导致玩家失去了对游戏的信心。  
如果想要效果变的更好，我们必须再多做一些事情！  
### Synchronization
一个常见的做法是，每帧直接把Simulation Object的translate和rotate设置给Character Entity，这样的话两者完全同步了，我们看下效果。  

{视频6 CodeDriven}  

这种做法的优势是可以完全直接地控制角色(a very direct, one-to-one feeling of control over the character), 很多游戏也采用了这种做法，但这种做法的劣势在于，除非动画跟运动匹配的相当完美，否则就会出现角色滑步，漂浮感，转身不协调等问题。  
(注: 下面视频是我测试用的CodeDriven的效果，可以很直观感受下上面说的这些问题)  

{视频7 OurCodeDriven}  

可能我们为了能让运动更加平滑选择去调整Simulation Object，但这样可能会适得其反，因为我们一开始同步Simulation Object和Character Entity的目的就是为了保证that feeling of responsiveness，如果调整Simulation Object的话，会导致less responsive, 还不如去修正下动画数据呢。 (注：数据一致性讲究的也是动画数据匹配Simulation运动，而不是反过来)  
还有一点需要特别注意的是，Simulation Object的Responsiveness主要涉及到Desired Velocity和Desired Rotation，而且这两者都单独控制的，所以这两项的调整都会影响到动画的选择。很多游戏常犯的错误是Desired Rotation反馈太过灵敏，导致旋转效果不真实并且玩家想精确走到某个位置都很难。  
当然了，主要还是看游戏，很可能一些游戏产品对于上面的效果已经很满意了。有的时候某些游戏会认为精确的控制比动画的真实感要重要的多。(注：比如《只狼》这种，角色的移动的快速反馈比脚步是否滑步这些重要的多)  


另外一种做法就是完全相反，为了尽可能地追求真实感，我们每帧使用Character Entity的数据去设置Simulation Object。  

{视频8 DataDriven}  

(注: 下面视频是我测试用的DataDriven的测试效果, 商城插件自带效果)

{视频9 OurDataDriven}  

可以看到动画表现的特别真实，而且几乎没有什么滑步，但缺点也很明显，除非你的动画数据特别敏捷精炼，否则玩家控制角色时会感觉特别迟缓且反馈慢。(注：想象一下2K手感玩只狼...)  
理所当然地，我们会尝试可不可以设置一个blend weight参数，去混合Simulation Object 和 Character Entity呢？ 将blend后的数据同时设置给Simulaton Object和Character Entity，我们看下效果：  

{视频10 CodeDataBlend}  

效果确实不错，但我个人并不是很满意， 因为它本质上的做法是两个限制部分都做了退让而不是去同时满足它们。(注：最坏结果是反馈一般，真实性也一般，两边都不讨好。但我觉得还得看你的游戏，如果这种方式的手感和表现可以接受的话，也是完全OK的！)  
而且会经常出现这种退让情况，因为玩家的控制是无法预测的，游戏内的角色跟正常人也不一样，比如运动速度，还有就是正常人走路或者停步的时候脑袋会有一点点时间去准备，游戏则不行，当玩家滑动摇杆的时候，角色应该立马有反馈以及响应。  
除了上面让两种限制都退让的办法，还有其他的办法吗？  
### Adjustment
不像上面说的那样，每帧用Simulation Object的数据设置给Character Entity(反之亦然),我们换一种思路，Simulation Object和Character Entity还是各走各的，不过Character Entity随着时间慢慢向Simulation Object靠拢，这种方法允许一些误差但好处是，不用在Simulation Object上妥协了。  
比如，我们使用[Damper](https://theorangeduck.com/page/spring-roll-call#damper)将Character Entity推向Simulation Object: 
```
vec3 adjust_character_position(
    const vec3 character_position,
    const vec3 simulation_position,
    const float halflife,
    const float dt)
{
    // Find the difference in positioning
    vec3 difference_position = simulation_position - character_position;
    
    // Damp that difference using the given halflife and dt
    vec3 adjustment_position = damp_adjustment_implicit(
        difference_position,
        halflife,
        dt);
    
    // Add the damped difference to move the character toward the sim
    return adjustment_position + character_position;
}
```
对于旋转也是使用同样的方法:  
```
quat adjust_character_rotation(
    const quat character_rotation,
    const quat simulation_rotation,
    const float halflife,
    const float dt)
{
    // Find the difference in rotation (from character to simulation).
    // Here `quat_abs` forces the quaternion to take the shortest 
    // path and normalization is required as sometimes taking 
    // the difference between two very similar rotations can 
    // introduce numerical instability
    quat difference_rotation = quat_abs(quat_normalize(
        quat_mul_inv(simulation_rotation, character_rotation)));
    
    // Damp that difference using the given halflife and dt
    quat adjustment_rotation = damp_adjustment_implicit(
        difference_rotation,
        halflife,
        dt);
    
    // Apply the damped adjustment to the character
    return quat_mul(adjustment_rotation, character_rotation);
}
```
damp_adjustment_implicit是这里[damper code](https://theorangeduck.com/page/spring-roll-call#implicitspringdamper)的变种版本(damps a point starting at zero (or the identity rotation) moving toward the desired difference using a given halflife and dt)。 
```
#define LN2f 0.69314718056f

vec3 damp_adjustment_implicit(vec3 g, float halflife, float dt, float eps=1e-8f)
{
    return g * (1.0f - fast_negexpf((LN2f * dt) / (halflife + eps)));
}

quat damp_adjustment_implicit(quat g, float halflife, float dt, float eps=1e-8f)
{
    return quat_slerp_shortest_approx(quat(), g,
        1.0f - fast_negexpf((LN2f * dt) / (halflife + eps)));
}
```
quat_slerp_shortest_approx的推导和实现来源于[Arseny Kapoulkine的这篇文章](https://zeux.io/2015/07/23/approximating-slerp/)

应用Adjustment后的表现如下:  

{视频11 AdjustmentBasic}  

正如前面所说的，rotational adjustment和positional adjustment可以单独控制变化速度，我们经常把Rotational Adjustment调整的慢一点，这样的话一方面对于Responsiveness影响较小，另一方面如果设置太过激进的话表现会很糟糕。  
目前表现大致上还不错，但仔细看的话，角色在stand或者plant-and-turn的时候然后会有一些脚步滑动的问题。  
我们尝试使用这样一种办法，通过给total character velocity设置一个比例系数并计算一个最大max_length从而限制移动的距离。(看代码啥都懂了)  

```
vec3 adjust_character_position_by_velocity(
    const vec3 character_position,
    const vec3 character_velocity,
    const vec3 simulation_position,
    const float max_adjustment_ratio,
    const float halflife,
    const float dt)
{
    // Find and damp the desired adjustment
    vec3 adjustment_position = damp_adjustment_implicit(
        simulation_position - character_position,
        halflife,
        dt);
    
    // If the length of the adjustment is greater than the character velocity 
    // multiplied by the ratio then we need to clamp it to that length
    float max_length = max_adjustment_ratio * length(character_velocity) * dt;
    
    if (length(adjustment_position) > max_length)
    {
        adjustment_position = max_length * normalize(adjustment_position);
    }
    
    // Apply the adjustment
    return adjustment_position + character_position;
}

quat adjust_character_rotation_by_velocity(
    const quat character_rotation,
    const vec3 character_angular_velocity,
    const quat simulation_rotation,
    const float max_adjustment_ratio,
    const float halflife,
    const float dt)
{
    // Find and damp the desired rotational adjustment
    quat adjustment_rotation = damp_adjustment_implicit(
        quat_abs(quat_normalize(quat_mul_inv(
            simulation_rotation, character_rotation))),
        halflife,
        dt);
    
    // If the length of the adjustment is greater than the angular velocity 
    // multiplied by the ratio then we need to clamp this adjustment
    float max_length = max_adjustment_ratio *
        length(character_angular_velocity) * dt;
    
    if (length(quat_to_scaled_angle_axis(adjustment_rotation)) > max_length)
    {
        // To clamp convert to scaled angle axis, rescale, and convert back
        adjustment_rotation = quat_from_scaled_angle_axis(max_length * 
            normalize(quat_to_scaled_angle_axis(adjustment_rotation)));
    }
    
    // Apply the adjustment
    return quat_mul(adjustment_rotation, character_rotation);
}
```

我们把max_adjustment_ratio设置成0.5，看下效果： 

{视频12 AdjustmentMaxRatio}  

可以看到，在plant-and-turns和start-and-stop时，滑步没有那么明显了。  
即使Character看起来有些沉重感，我仍然喜欢这个方案，因为它在本质上是data-driven的方案上增加了很多控制感。 
即使我们做了这么多，在某些情况下偏离的问题依然存在，所以我们还需要一些额外的同步行为，我们还可以做哪些工作呢？(注：当你读到这里，你知道为什么还存在偏离吗，上面哪个步骤做的越好这里偏离会越小呢？)   
### Clamping
为了解决Character Entity偏离Simulation Object太远的问题，我们可以对distance和angle做一个clamp操作，代码如下：  
```
vec3 clamp_character_position(
    const vec3 character_position,
    const vec3 simulation_position,
    const float max_distance)
{
    // If the character deviates too far from the simulation 
    // position we need to clamp it to within the max distance
    if (length(character_position - simulation_position) > max_distance)
    {
        return max_distance * 
            normalize(character_position - simulation_position) + 
            simulation_position;
    }
    else
    {
        return character_position;
    }
}

quat clamp_character_rotation(
    const quat character_rotation,
    const quat simulation_rotation,
    const float max_angle)
{
    // If the angle between the character rotation and simulation 
    // rotation exceeds the threshold we need to clamp it back
    if (quat_angle_between(character_rotation, simulation_rotation) > max_angle)
    {
        // First, find the rotational difference between the two
        quat diff = quat_abs(quat_mul_inv(
            character_rotation, simulation_rotation));
        
        // We can then decompose it into angle and axis
        float diff_angle; vec3 diff_axis;
        quat_to_angle_axis(diff, diff_angle, diff_axis);
        
        // We then clamp the angle to within our bounds
        diff_angle = clampf(diff_angle, -max_angle, max_angle);
        
        // And apply back the clamped rotation
        return quat_mul(
          quat_from_angle_axis(diff_angle, diff_axis), simulation_rotation);
    }
    else
    {
        return character_rotation;
    }
}
```
其中quat_angle_between函数如下，如果你不知道代码为什么这么写，可以参考[这篇文章](https://theorangeduck.com/page/exponential-map-angle-axis-angular-velocity)
```
float quat_angle_between(quat q, quat p)
{   
    quat diff = quat_abs(quat_mul_inv(q, p));
    return 2.0f * acosf(clampf(diff.w, -1.0f, 1.0f));
}
```
clamp定义了最大的偏移是多少，MotionMatching在网络同步以及碰撞检测时相当重要。clamp和adjustment结合后的效果如下：

{视频13 Clamped}  

我们可以看到clamp限制了Character偏离距离同时也保证Character在圈圈里移动的自然，同时有的时候可以看到Character被拉拽的痕迹，这种问题需要设计师和动画师去权衡了。  
### Be Creative
这里没有绝对完美的方案，所以需要你把上面讲到的一个多个技术跟你们自身的技术进行结合，我们估计设计师和动画师去探索一些潜在的技术方面，能让你们深刻到哪些东西是我们关心的，哪些是不重要的。  
我们文章中说过Simulation Object要尽可能贴合动画数据，但是正常情况下，我们要做的是，让动画数据去贴合Simulation Object, Dan Lowe的[这篇视频](https://www.youtube.com/watch?v=eeWBlMJHR14)讲解给了我们很多tricks，比如在我们这个例子中，我们可以把max-speed最高提高到动画数据速度的10%左右来增强responsiveness,还有就是mirroring去增大数据范围。  
还有就是[V社《Half-Life: Alyx》的研究](https://www.youtube.com/watch?v=RCu-NzH4zrs)可以解决一些特别基础的疑难问题，使动画适配的更加自然。  
还有一些特别细的tricks可以改善responsiveness的体验，比如在转向时添加程序化的lookat，使head始终面向desired direction等。  
在调配参数上，我倾向于创建一个数据面板的UI展示给设计师和动画师，他们可以根据自己的喜好随意配置，这种调配比那种参数隐藏在状态机，逻辑脚本等要好一些。  
Previously, motion matching and other data heavy or machine learning methods have been framed as fundamentally opposed to the fast and responsive code-driven control that is required by many games, but I don't see it like this.

By default motion matching does not blend animation data which means often it's difficult to achieve an exact desired velocity or turning angle without some correction. But at the same time, having a fluid, interruptible-at-any-point animation system which uses a lot of data can also mean you have a greater chance of finding a realistic animation that can achieve the desired movement of the character with minimal adjustment, visual weirdness, and data preparation.(注：这一段我觉得要讲的应该是MotionMatching中应该有选项支持Data-Driven或者混合模式，比如《荣耀战魂》中的Repoision以及《Control》中的MotionWarping功能等，等后面再详细补充吧)

### Sliding and Foot Locking
滑步是老生常谈的问题了，我们尝试运行时修正动画来解决下。  
一般情况下分两步来解决：一是发现脚步接触地板时锁住该脚直到接触结束。二是[使用IK调整Pose](https://theorangeduck.com/page/simple-two-joint)。  
For the locking I like to use inertialization again. Here the logic is relatively simple. We take as input the position and velocity of the toe joint in the world space, as well as a flag saying if a contact is active or not. When the foot enters a contact state we use an inertializer to transition to a state where we feed as input just that fixed contact point (with zero velocity), then, when we release the contact we transition the inertializer back to just feeding it the animation data. We can also add an additional little heuristic which unlocks the foot until the next contact if the distance between the locked contact point and the animation data grows too large.  
Once we have this sliding-free toe location we can then compute the desired heel location and use something similar to this method of inverse kinematics to pin the foot in place.  
In the video below I also do some additional little procedural fix-ups like preventing the foot from penetrating the ground plane and bending the toe when the foot comes in contact with the floor. Check the source code for the full details.  
(注：等我看完再详细解释吧，这里面还是有不少细节问题的 -。-！！)  
```
void contact_update(
    bool& contact_state,
    bool& contact_lock,
    vec3& contact_position,
    vec3& contact_velocity,
    vec3& contact_point,
    vec3& contact_target,
    vec3& contact_offset_position,
    vec3& contact_offset_velocity,
    const vec3 input_contact_position,
    const bool input_contact_state,
    const float unlock_radius,
    const float foot_height,
    const float halflife,
    const float dt,
    const float eps=1e-8)
{
    // First compute the input contact position velocity via finite difference
    vec3 input_contact_velocity = 
        (input_contact_position - contact_target) / (dt + eps);    
    contact_target = input_contact_position;
    
    // Update the inertializer to tick forward in time
    inertialize_update(
        contact_position,
        contact_velocity,
        contact_offset_position,
        contact_offset_velocity,
        // If locked we feed the contact point and zero velocity, 
        // otherwise we feed the input from the animation
        contact_lock ? contact_point : input_contact_position,
        contact_lock ?        vec3() : input_contact_velocity,
        halflife,
        dt);
    
    // If the contact point is too far from the current input position 
    // then we need to unlock the contact
    bool unlock_contact = contact_lock && (
        length(contact_point - input_contact_position) > unlock_radius);
    
    // If the contact was previously inactive but is now active we 
    // need to transition to the locked contact state
    if (!contact_state && input_contact_state)
    {
        // Contact point is given by the current position of 
        // the foot projected onto the ground plus foot height
        contact_lock = true;
        contact_point = contact_position;
        contact_point.y = foot_height;
        
        inertialize_transition(
            contact_offset_position,
            contact_offset_velocity,
            input_contact_position,
            input_contact_velocity,
            contact_point,
            vec3());
    }
    
    // Otherwise if we need to unlock or we were previously in 
    // contact but are no longer we transition to just taking 
    // the input position as-is
    else if ((contact_lock && contact_state && !input_contact_state) 
         || unlock_contact)
    {
        contact_lock = false;
        
        inertialize_transition(
            contact_offset_position,
            contact_offset_velocity,
            contact_point,
            vec3(),
            input_contact_position,
            input_contact_velocity);
    }
    
    // Update contact state
    contact_state = input_contact_state;
}
```

{视频14 IK} 

我们上面使用的IK办法在测试环境里还好，但是放到游戏内可能会出现各种问题，很多边缘条件我们都没有考虑到，所以这属于一个长期迭代的问题，需要团队找到自己合适的解决方案。  
然而，foot locking这个问题是仁者见仁的事，允许一点点滑步有的时候反而效果更好，foot locking会破坏动画pose的连续性，看起来有些怪异，所以可以把时间优先放到其他技术上。  
有的时候，当玩家向墙的方向行走时，滑步反而是最优解，哈哈。  

{视频15 bring GTA to life}
### Move the Camera
当然了，我也发现了很多游戏都在用的另外一种技术，比如deadline已经很临近了，我们可以将Camera移到肩膀上，也可以完美解决滑步！  

{视频16 MoveCamera}

看吧，确实没有滑步了！

### Source Code

https://github.com/orangeduck/Motion-Matching

## 我的总结
* Data Driven Displacement意味着角色动画的真实性更强，但同时控制反馈比较差，表现就是手感很硬，Code Driven Displacement相反，所以需要根据游戏类型，设计师喜好来进行权衡，没有银弹！
* 临界阻尼可以考虑作为常规武器加入到游戏开发的工具箱里，不要一想到平滑就线性插值。后面计划做一个SpringDamper测试关卡！
* 可以利用Savitzky-Golay filter在Maya上写一个程序化生成RootBone的脚本。(参考generate_database.py)
* Trajectory数据相对于Character Entity而并非Simulation Object。
* 关于Synchronization的讲解也特别透彻，刚接触MotionMatching的时候一直有一个疑惑就是直接CodeDriven不就可以了吗，动画仅仅是在模型上播放就可以了，其实自己实现看到效果后才知道Adjustment和Clamping的必要性，因为效果太差了，滑步，漂浮感，转向等问题都出来了，很难应用到游戏里。商城里的插件也仅仅是做到了DataDriven。值得注意的是，《最后生还者2》中主角是没有Clamping的，所以就更加要求数据一致性。
* 自己的基础太薄弱了，比如看些Quat相关的还是不熟练，后面计划做一个数学测试关卡练练！
* MotionMatching有选项支持Data-Driven以及混合模式，需要根据《荣耀战魂》和《Control》的例子实操下。
* 项目代码中大量细节，后面计划分几篇文章一一讲解！
* raylib和raygui是优秀的工具集，多平台支持，后面做小游戏或者小工具的话可以考虑使用。
