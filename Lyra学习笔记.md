# 学习目标

1. Lyra动画框架，制作规范，C/S端执行流程，新特性使用及其原理等
   * Lyra动画框架，包括LinkAnimBlueprint，如何拆分，如何配置，如何生效等；
   * LinkAnimBlueprint在动态切换时，两个不同的LinkAnim是如何过渡的；
   * 在这个框架下，如果添加新武器需要配置哪些内容，跟ALS相比有哪些优势；
   * DS端一般不执行动画蓝图的逻辑，Curves有时没数据，如何解决？
   * DS端不执行动画蓝图的情况下，如何正确的伤害判定？
   * 新特性PropertyAccess，UE5新的Update, Evaluation原理
  
2. C/S模型下3C的执行流程，特别是Movement, 比如Movement如何写
   * CharacterMovement如何重写，以及CharacterMovement原理解析

3. 其他框架，比如GAS, GF, SubSystem, 进副本流程，GameMode与Experience等








4. 杂项
   CommonLoadingScreen解析
   PropertyAccessSystem原理
   ModularGameplay解析，包括GameFrameworkComponentManager，UGameFrameworkComponent等，是什么，解决什么问题，怎么用等等
   CommonUser解析



   ### UpperBody/LowerBody Split

```
├─root                         0
├─ik-foot-root                 0
├─ik-hand-root                 1
├─pelvis                       0.1
│  ├─spline-01                 0.2
│  │  └─spline-02              0.3
│  │      └─spline-03          0.4
│  │        └─spline-04        0.5
│  │          └─spline-05      0.6
│  │            ├─clavicle-l   0.75
│  │            │ └─*          1
│  │            ├─clavicle-r   0.75
│  │            │ └─*          1
│  │            ├─neck01       0.75
│  │            │ └─*          1
│  └─thigh-l                   0
│  └─thigh-r                   0
```