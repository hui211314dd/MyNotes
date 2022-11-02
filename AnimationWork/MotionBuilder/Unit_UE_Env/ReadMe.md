# 简介
Unit_UE4_Env提供了Kinematica_Demo中的动画fbx转为UEMannequin的环境

# 如何使用呢？

1. MotionBuilder导入Unit_UE4_Env，Character设置为UnitCharacter, Source设置为None
2. 点击File/Motion File Import... 导入需要的动画资源，比如Kinematica_Demo\Assets\Animations\Biped\Circles_Jog_1.fbx, 弹出Import Options设置框，设置为Merge, 点击Import
3. 点击Bake(Plot) to ControlRig，将动画设置给UnitCharacter ControlRig了
4. Character设置为UECharacter, Source设置为UnitCharacter
5. 点击Bake(Plot) to ControlRig，将动画设置给UECharacter ControlRig了
6. 选中Scene中的root(不是Root, root是UE骨骼的根骨骼)，点击Bake(Plot) Selection
7. 再点击Bake(Plot) to Skeleton, 这时候所有动画信息就Bake到UE骨骼上了
8. 选中Scene中的root，右键选择Select Branches, 选择File/Save Selection, 导出名字为UE_Circles_Jog_1.FBX，弹出的Save Selection Options只勾选Take 2即可，然后点击Save就可以了
9. 将UE_Circles_Jog_1拖入引擎，Skeletion选择UE_Mannequin_Skeletion，点击Import All即可
