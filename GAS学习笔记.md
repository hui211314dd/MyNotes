## Attribute

即刻(Instant)GameplayEffect可以永久性的修改BaseValue, 而持续(Duration)和无限(Infinite)GameplayEffect可以修改CurrentValue. 周期性(Periodic)GameplayEffect被视为即刻(Instant)GameplayEffect并且可以修改BaseValue.

### PreAttributeChange
PreAttributeChange(const FGameplayAttribute& Attribute, float& NewValue)是AttributeSet中的主要函数之一, 其在修改发生前响应Attribute的CurrentValue变化, 其是通过引用参数NewValue限制(Clamp)CurrentValue即将进行的修改的理想位置.

### PostGameplayEffectExecute

PostGameplayEffectExecute(const FGameplayEffectModCallbackData & Data)仅在即刻(Instant)GameplayEffect对Attribute的BaseValue修改之后触发, 当GameplayEffect对其修改时, 这就是一个处理更多Attribute操作的有效位置.

例如, 在样例项目中, 我们在这里从生命值Attribute中减去了最终的伤害值Meta Attribute, 如果有护盾值Attribute的话, 我们也会在减除生命值之前从护盾值中减除伤害值. 样例项目也在这里应用了被击打反应动画, 显示浮动的伤害数值和为击杀者分配经验值和赏金. 通过设计, 伤害值Meta Attribute总是会传递给即刻(Instant)GameplayEffect而不是Attribute Setter.

## GameEffect



TODO

1. BaseHealth为1，应用了10s的增血buff, 伤害发生时会修改BaseHealth, 导致BaseHealth为0？这一刻血量显示正确吗(由于Clamp的原因，BaseHealth只能设置为0)？10s之后会死亡吗？还是说10s的buff会修改BaseHealth?

1. GAS各个模块在网络游戏中的流程，ROLE_SimulatedProxy，ROLE_AutonomousProxy，ROLE_Authority都执行哪些内容

2. 什么是预测以及如何预测

3. RootMotion的技能如何实现和同步


参考资料：

[GASDocumentation](https://github.com/BillEliot/GASDocumentation_Chinese)
[GameplayAbilities and You中文版](https://zhuanlan.zhihu.com/p/338036335)
[虚幻四Gameplay Ability System入门系列](https://zhuanlan.zhihu.com/p/366500615)