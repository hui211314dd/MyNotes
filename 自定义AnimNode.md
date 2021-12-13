## 自定义AnimNode

/* 节点初始化使用，可能被调用多次 */  
virtual void Initialize_AnyThread(const FAnimationInitializeContext& Context);
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>

/* 更新weight信息 */  
void FAnimNode_SequencePlayer::UpdateAssetPlayer(const FAnimationUpdateContext& Context)
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>

/* 打开动画蓝图BP时，SequencePlayerNode节点的Debug显示，去掉的话，可以看到SequencePlayerNode下面的Frame滑块不再更新了，仅编辑器使用。 */  
DebugData->RecordSequencePlayer(Context.GetCurrentNodeId(), GetAccumulatedTime(), Sequence != nullptr ? Sequence->SequenceLength : 0.0f, Sequence != nullptr ? Sequence->GetNumberOfFrames() : 0);
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>

TRACE_ANIM_SEQUENCE_PLAYER和TRACE_ANIM_NODE_VALUE系列函数供UnrealInsights使用。
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>

/* 供ShowDebug Animation使用 */  
void FAnimNode_SequencePlayer::GatherDebugData(FNodeDebugData& DebugData)
<br/>
<br/>
<br/>
<br/>
<br/>
<br/>

### 疑问
GetEvaluateGraphExposedInputs().Execute(Context)执行什么。  
答：AnimGraph中Node有些输入引脚需要实时计算，Execute是进行计算的。