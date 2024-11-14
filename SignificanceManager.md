```C++

CMGameViewportClient.cpp

// Fill out your copyright notice in the Description page of Project Settings.


#include "GameViewportClient/CMGameViewportClient.h"
#include "SignificanceManager.h"
#include "Kismet/GameplayStatics.h"
#include "Camera/PlayerCameraManager.h"

void UCMGameViewportClient::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

	// Ensure that we have a valid World and Significance Manager instance.
	if (const UWorld* World = GetWorld())
	{
		if (USignificanceManager* SignificanceManager = USignificanceManager::Get<USignificanceManager>(World))
		{
			// Update once per frame, using only Player 0's world transform.
			if (const APlayerCameraManager *LocalCameraManager = UGameplayStatics::GetPlayerCameraManager(World, 0))
			{
				// The Significance Manager uses an ArrayView. Construct a one-element Array to hold the Transform.
				FVector CameraLocation;
				FRotator CameraRotation;
				LocalCameraManager->GetCameraViewPoint(CameraLocation, CameraRotation);
				const FTransform CameraViewPoint(CameraRotation, CameraLocation);
				TArray<FTransform> TransformArray;
				TransformArray.Add(CameraViewPoint);
				// Update the Significance Manager with our one-element Array passed in through an ArrayView.
				SignificanceManager->Update(TArrayView<FTransform>(TransformArray));
			}
		}
	}
}
```


```C++
CMSignificanceManager.h

// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "OrderedBudget.h"
#include "SignificanceManager.h"
#include "CMSignificanceManager.generated.h"

/**
 * 创建新类型：
 * Step 0: 创建ECmSignificanceType新类型；
 * Step 1: 创建FSignificanceTypeData的子类(可选)；
 * Step 2: 创建FManagedObjectInfo的子类；
 * Step 3: 实现DefaultSignificance(计算得分)和DefaultPostSignificanceUpdate(根据得分排序结果按高到低分配LOD)函数；
 * Step 4: 在UCMSignificanceManager构造函数里注册；
 * Step5:  在RegisterSignificanceObject中处理;
 */


UENUM(BlueprintType)
enum class ECmSignificanceType : uint8
{
	// 对应大世界中所有的Pawns
	BigWorld_PawnDefault,

	// 对应大世界的所有静态物体，包括City, Mine等；
	BigWorld_StaticDefault,
	
	// 非战斗场景的NPCs
	NonBattleScene_PawnDefault,

	// 战斗场景步兵
	BattleScene_InfantryDefault,

	// 战斗场景骑兵
	BattleScene_DragoonDefault,

	Max    UMETA(DisplayName="Invalid"),
};

/**
 *  SourceCode from https://udn.unrealengine.com/s/question/0D52L00004lug9TSAQ/significance-manager-crashes-with-more-than-one-object-tag
 *                                https://udn.unrealengine.com/s/question/0D54z00008kSTCtCAO/significance-manager-setup
 */
UCLASS()
class UCMSignificanceManager : public USignificanceManager
{
	GENERATED_BODY()
public:
	UCMSignificanceManager();

public:
	struct FSignificanceTypeData
	{};

	// Define Type Data Begin
	struct FBigWorldPawnDefaultSignificanceTypeData: public FSignificanceTypeData
	{
		float MyData;

		FBigWorldPawnDefaultSignificanceTypeData(): MyData(1.0f){}
	};
	
	typedef TFunction<void(UCMSignificanceManager*)> FPreUpdateFunction;
	typedef TFunction<void(UCMSignificanceManager*)> FPostUpdateFunction;

	// 每个有效的ECmSignificanceType会对应一个FSignificanceTypeInfo
	struct FSignificanceTypeInfo
	{
		// 使用Tag建立外部Actors和FSignificanceTypeInfo的对应关系
		FName Tag;
		
		EPostSignificanceType PostSignificanceType = EPostSignificanceType::None;
		TUniquePtr<FSignificanceTypeData> Data;

		struct FOrderedBudget Budget;
		//add-on update functions (not part of base significance manager functionality)
		FPreUpdateFunction PreUpdateFunction;
		FPostUpdateFunction PostUpdateFunction;
	};

	// Define Type Data End

	// Define Object Data Begin

	struct FBigWorldPawnInfo : public USignificanceManager::FManagedObjectInfo
	{
	public:
		FBigWorldPawnInfo(UObject* InObject, FName InTag): USignificanceManager::FManagedObjectInfo(InObject, InTag, &BigWorldPawnDefaultSignificance,
			USignificanceManager::EPostSignificanceType::Sequential, &BigWorldPawnDefaultPostSignificance), SignificanceLOD( -1 )
		{}

		int32 GetSignificanceLOD() const { return SignificanceLOD; }

		static float BigWorldPawnDefaultSignificance(const FManagedObjectInfo* ObjectInfo, const FTransform& Viewpoint);
		static void BigWorldPawnDefaultPostSignificance(const FManagedObjectInfo* ObjectInfo, float OldSignificance, float NewSignificance, bool bFinal);

		//add-on handling for pre-post update operations
		static void BigWorldPawnDefaultPreSignificanceUpdate(UCMSignificanceManager* SignificanceManager);
		static void BigWorldPawnDefaultPostSignificanceUpdate(UCMSignificanceManager* SignificanceManager);
	public:
		int32 SignificanceLOD;
	};

	struct FBigWorldStaticInfo : public USignificanceManager::FManagedObjectInfo
	{
	public:
		FBigWorldStaticInfo(UObject* InObject, FName InTag): USignificanceManager::FManagedObjectInfo(InObject, InTag, &BigWorldStaticDefaultSignificance,
			USignificanceManager::EPostSignificanceType::Sequential, &BigWorldStaticDefaultPostSignificance), SignificanceLOD( -1 )
		{}

		int32 GetSignificanceLOD() const { return SignificanceLOD; }

		static float BigWorldStaticDefaultSignificance(const FManagedObjectInfo* ObjectInfo, const FTransform& Viewpoint);
		static void BigWorldStaticDefaultPostSignificance(const FManagedObjectInfo* ObjectInfo, float OldSignificance, float NewSignificance, bool bFinal){};

		//add-on handling for pre-post update operations
		static void BigWorldStaticDefaultPreSignificanceUpdate(UCMSignificanceManager* SignificanceManager){};
		static void BigWorldStaticDefaultPostSignificanceUpdate(UCMSignificanceManager* SignificanceManager);
	public:
		int32 SignificanceLOD;
	};
	
	// Define Object Data End
public:
	// Overridable function to update the managed objects' significance
	virtual void Update(TArrayView<const FTransform> Viewpoints) override;

	//This is the function that users call as a replacement for the virtual RegisterObject which exposes only the significance type so it can be looked up
	void RegisterSignificanceObject(UObject* Object, ECmSignificanceType SignificanceType);

	static FSignificanceTypeInfo* GetSignificanceTypeInfo(ECmSignificanceType SignificanceType);
private:
	float ElapsedTime;
};

```

```C++

CMSignificanceManager.cpp

// Fill out your copyright notice in the Description page of Project Settings.


#include "GameManager/CMSignificanceManager.h"
#include "Simulator/Actors/MBSimBaseMovableActor.h"
#include "Simulator/Actors/MBSimEditorActor.h"

static float GcmSignificanceManagerTickIntervalTime = 1.0f;
static FAutoConsoleVariableRef CVarSignificanceManagerTickIntervalTime(
	TEXT("CMSigMan.TickIntervalSeconds"),
	GcmSignificanceManagerTickIntervalTime,
	TEXT("0 means call update function per frame.\n"),
	ECVF_Default
	);

static TAutoConsoleVariable<FString> CVarBigWorldPawnBudgetSpecification(
	TEXT("CMSigMan.BigWorldPawnBudgetSpecification"),
	TEXT("1,8,15"),
	TEXT("Level0Num,Level1Num,Level2Num,Level3Num..."),
	ECVF_Default);

static TAutoConsoleVariable<FString> CVarBigWorldStaticBudgetSpecification(
	TEXT("CMSigMan.BigWorldStaticBudgetSpecification"),
	TEXT("5,5,50"),
	TEXT("Level0Num,Level1Num,Level2Num,Level3Num..."),
	ECVF_Default);

//static storage of all the significance handling data
static UCMSignificanceManager::FSignificanceTypeInfo SignificanceTypeInfo[(uint8)ECmSignificanceType::Max];

UCMSignificanceManager::UCMSignificanceManager()
	:Super()
{
	ElapsedTime = 0;

	if (IsTemplate() /*Only populate static struct in the CDO*/)
	{
		FSignificanceTypeInfo& PawnDefaultInfo = SignificanceTypeInfo[(uint8)ECmSignificanceType::BigWorld_PawnDefault];
		PawnDefaultInfo.Tag = "BigWorld.PawnDefault";
		//can leave significance functions unmodified if they aren't used...
		PawnDefaultInfo.PostSignificanceType = EPostSignificanceType::Sequential;
		PawnDefaultInfo.Data = MakeUnique<FBigWorldPawnDefaultSignificanceTypeData>();
		// 支持不同手机型号不同设置
		PawnDefaultInfo.Budget.RecreateBudget(CVarBigWorldPawnBudgetSpecification.GetValueOnAnyThread());
		//Also register update functions, if they exist for this tag
		PawnDefaultInfo.PreUpdateFunction = &FBigWorldPawnInfo::BigWorldPawnDefaultPreSignificanceUpdate;
		PawnDefaultInfo.PostUpdateFunction = &FBigWorldPawnInfo::BigWorldPawnDefaultPostSignificanceUpdate;

		FSignificanceTypeInfo& StaticDefaultInfo = SignificanceTypeInfo[(uint8)ECmSignificanceType::BigWorld_StaticDefault];
		StaticDefaultInfo.Tag = "BigWorld.StaticDefault";
		//can leave significance functions unmodified if they aren't used...
		StaticDefaultInfo.PostSignificanceType = EPostSignificanceType::Sequential;
		// 支持不同手机型号不同设置
		StaticDefaultInfo.Budget.RecreateBudget(CVarBigWorldStaticBudgetSpecification.GetValueOnAnyThread());
		//Also register update functions, if they exist for this tag
		StaticDefaultInfo.PreUpdateFunction = &FBigWorldStaticInfo::BigWorldStaticDefaultPreSignificanceUpdate;
		StaticDefaultInfo.PostUpdateFunction = &FBigWorldStaticInfo::BigWorldStaticDefaultPostSignificanceUpdate;
	}
}

// Calculate significance based on distance and other factors
float UCMSignificanceManager::FBigWorldPawnInfo::BigWorldPawnDefaultSignificance(
	const FManagedObjectInfo* ObjectInfo, const FTransform& Viewpoint)
{
	const UCMSignificanceManager::FBigWorldPawnInfo* PlayerInfo = static_cast<const UCMSignificanceManager::FBigWorldPawnInfo*>(ObjectInfo);

	// Grab any data from the actor we may need here
	const AMBSimBaseMovableActor* PlayerPawn = CastChecked<const AMBSimBaseMovableActor>(ObjectInfo->GetObject());
 
	// You can have conditions here to force max significance based on the pawn properties (eg. they're the local player).
	if (PlayerPawn->IsLocalPlayer())
	{
		return MAX_FLT;
	}
	
	// Calculate significance based on things like distance or whatever makes sense for your project.
	// 判断标准：IsVisible(第一考虑) + 距离(第二考虑)
	float FinalScore = 0;
	if(PlayerPawn->WasRecentlyRenderedOnScreen(0.2f) || PlayerPawn->IsVisible() )
	{
		FinalScore += 1000000000;
	}

	const float DistanceSquared2D = FVector::DistSquared2D(PlayerPawn->GetActorLocation(), Viewpoint.GetLocation());
	// 分数与距离成反比
	FinalScore += FMath::GetMappedRangeValueClamped(FVector2f(0, 2500000000.0f), FVector2f(500000000, 0), DistanceSquared2D);
	
	return FinalScore;
}

// This is called per-object by USignificanceManager::Update after the significance values have been calculated.
// This has the option to be run in a parallel for loop if you have thread safe operations you can run to apply the new significance values.
// Or you can run it sequentially, see EPostSignificanceType.
// 
// If the order of the actors doesn't matter then it's better to use this than the PostUpdate above as it removes a level of indirection.
void UCMSignificanceManager::FBigWorldPawnInfo::BigWorldPawnDefaultPostSignificance(
	const FManagedObjectInfo* ObjectInfo, float OldSignificance, float NewSignificance, bool bFinal)
{
}

// Called once in UMySignificanceManager::Update before USignificanceManager::Update runs.
void UCMSignificanceManager::FBigWorldPawnInfo::BigWorldPawnDefaultPreSignificanceUpdate(
	UCMSignificanceManager* SignificanceManager)
{
	// This is a good place to pre-calcuate any common data you may need during the per-object significance calculations so you can avoid doing them for every object. You likely won't need this though.
}

// Called once in UMySignificanceManager::Update after USignificanceManager::Update runs.
// The key difference between this function and the PostSignificance function you give to FManagedObject is this is called
// after USignificanceManager has sorted the list by significance.
void UCMSignificanceManager::FBigWorldPawnInfo::BigWorldPawnDefaultPostSignificanceUpdate(
	UCMSignificanceManager* SignificanceManager)
{
	// This is where we apply the Significance LOD to each player.
	// We define these as basically a large list of ints where there is 1 entry per player, then we use the player's index in the sorted list that USignificanceManager::Update created to lookup their Significance LOD in that list.
	const UCMSignificanceManager::FSignificanceTypeInfo* PlayerTypeInfo = UCMSignificanceManager::GetSignificanceTypeInfo(ECmSignificanceType::BigWorld_PawnDefault);
	const TArray<USignificanceManager::FManagedObjectInfo*>& ManagedPlayers = SignificanceManager->GetManagedObjects(PlayerTypeInfo->Tag);
 
	// Loop over the list of players, sorted by significance.
	for (int32 SignificanceIndex = 0; SignificanceIndex < ManagedPlayers.Num(); ++SignificanceIndex)
	{
		// Budget is an FOrderedBudget which is an engine type.
		// SignificanceLOD为逻辑LOD, 以0开始
		const int32 NewSignificanceLOD = PlayerTypeInfo->Budget.GetBudgetForIndex(SignificanceIndex);
 
		const FBigWorldPawnInfo* ConstPlayerInfo = static_cast<const FBigWorldPawnInfo*>(ManagedPlayers[SignificanceIndex]);
		FBigWorldPawnInfo* PlayerInfo = const_cast<FBigWorldPawnInfo*>(ConstPlayerInfo);
		
		// 6. Notify the pawn about the new significance LOD value.
		AMBSimBaseMovableActor* PlayerPawn = static_cast<AMBSimBaseMovableActor*>(PlayerInfo->GetObject());
		if (NewSignificanceLOD != PlayerInfo->SignificanceLOD)
		{
			PlayerInfo->SignificanceLOD = NewSignificanceLOD;
			// TODO ApplySignificanceLOD可以提到一个接口里
			PlayerPawn->ApplySignificanceLOD(NewSignificanceLOD);
		}
	}
}

float UCMSignificanceManager::FBigWorldStaticInfo::BigWorldStaticDefaultSignificance(
	const FManagedObjectInfo* ObjectInfo, const FTransform& Viewpoint)
{
	// Grab any data from the actor we may need here
	const AActor* CurActor = CastChecked<const AActor>(ObjectInfo->GetObject());
	
	// Calculate significance based on things like distance or whatever makes sense for your project.
	// 判断标准：距离
	float FinalScore = 0;
	
	const float DistanceSquared2D = FVector::DistSquared2D(CurActor->GetActorLocation(), Viewpoint.GetLocation());
	// 分数与距离成反比
	FinalScore += FMath::GetMappedRangeValueClamped(FVector2f(0, 2500000000.0f), FVector2f(500000000, 0), DistanceSquared2D);
	
	return FinalScore;
}

void UCMSignificanceManager::FBigWorldStaticInfo::BigWorldStaticDefaultPostSignificanceUpdate(
	UCMSignificanceManager* SignificanceManager)
{
	// This is where we apply the Significance LOD to each player.
	// We define these as basically a large list of ints where there is 1 entry per player, then we use the player's index in the sorted list that USignificanceManager::Update created to lookup their Significance LOD in that list.
	const UCMSignificanceManager::FSignificanceTypeInfo* StaticTypeInfo = UCMSignificanceManager::GetSignificanceTypeInfo(ECmSignificanceType::BigWorld_StaticDefault);
	const TArray<USignificanceManager::FManagedObjectInfo*>& ManagedPlayers = SignificanceManager->GetManagedObjects(StaticTypeInfo->Tag);
 
	// Loop over the list of players, sorted by significance.
	for (int32 SignificanceIndex = 0; SignificanceIndex < ManagedPlayers.Num(); ++SignificanceIndex)
	{
		// Budget is an FOrderedBudget which is an engine type.
		// SignificanceLOD为逻辑LOD, 以0开始
		const int32 NewSignificanceLOD = StaticTypeInfo->Budget.GetBudgetForIndex(SignificanceIndex);
 
		const FBigWorldStaticInfo* ConstStaticInfo = static_cast<const FBigWorldStaticInfo*>(ManagedPlayers[SignificanceIndex]);
		FBigWorldStaticInfo* StaticInfo = const_cast<FBigWorldStaticInfo*>(ConstStaticInfo);
		
		// 6. Notify the pawn about the new significance LOD value.
		AMBSimEditorActor* CurStaticActor = static_cast<AMBSimEditorActor*>(StaticInfo->GetObject());
		if (NewSignificanceLOD != StaticInfo->SignificanceLOD)
		{
			StaticInfo->SignificanceLOD = NewSignificanceLOD;
			// TODO ApplySignificanceLOD可以提到一个接口里
			CurStaticActor->ApplySignificanceLOD(NewSignificanceLOD);
		}
	}
}

void UCMSignificanceManager::RegisterSignificanceObject(UObject* Object, ECmSignificanceType SignificanceType)
{
	check(SignificanceType != ECmSignificanceType::Max);

	const FSignificanceTypeInfo& Info = SignificanceTypeInfo[(uint8)SignificanceType];

	if (SignificanceType == ECmSignificanceType::BigWorld_PawnDefault)
	{
		// UMyTag1Object* MyTag1Object = CastChecked<UMyTag1Object>(Object);
		// if (!MyTag1Object->ShouldManageSignificance() /* || some other checks */)
		// {
		// 	return;
		// }
		
		RegisterManagedObject(new FBigWorldPawnInfo(Object, Info.Tag));
	}
	else if (SignificanceType == ECmSignificanceType::BigWorld_StaticDefault)
	{
		RegisterManagedObject(new FBigWorldStaticInfo(Object, Info.Tag));
	}
	else
	{
		check(0);
	}
}

//---------------------------------新增新类型时下面的代码不用动----------------------------------------------

UCMSignificanceManager::FSignificanceTypeInfo* UCMSignificanceManager::GetSignificanceTypeInfo(
	ECmSignificanceType SignificanceType)
{
	return &SignificanceTypeInfo[(uint8)SignificanceType];
}

void UCMSignificanceManager::Update(TArrayView<const FTransform> Viewpoints)
{
	TRACE_CPUPROFILER_EVENT_SCOPE(UCMSignificanceManager::Update)
	
	if (!FMath::IsNearlyZero(GcmSignificanceManagerTickIntervalTime))
	{
		ElapsedTime += GetWorld()->GetDeltaSeconds();
		if (ElapsedTime < GcmSignificanceManagerTickIntervalTime)
		{
			return;
		}
		ElapsedTime -= GcmSignificanceManagerTickIntervalTime;
	}

	for (int32 Idx = 0; Idx < (int32)ECmSignificanceType::Max; Idx++)
	{
		if (SignificanceTypeInfo[Idx].PreUpdateFunction)
		{
			SignificanceTypeInfo[Idx].PreUpdateFunction(this);
		}
	}
	
	Super::Update(Viewpoints);

	for (int32 Idx = 0; Idx < (int32)ECmSignificanceType::Max; Idx++)
	{
		if (SignificanceTypeInfo[Idx].PostUpdateFunction)
		{
			SignificanceTypeInfo[Idx].PostUpdateFunction(this);
		}
	}
	// UE_LOG(LogTemp, Log, TEXT("UCMSignificanceManager::Update: %f, %f, %f"), Viewpoints[0].GetLocation().X, Viewpoints[0].GetLocation().Y, Viewpoints[0].GetLocation().Z);
}

/*
// ---------------------------------------------------------------
void APlayerPawn::BeginPlay()
{
	Super::BeginPlay();
 
	// 1. Register ourselves with the Significance system
	UMySignificanceManager::Get()->RegisterObject(this, EMyProjectSignificanceType::Player);
}
 
void APlayerPawn::ApplySignificanceLOD(const FPlayerPawnObjectInfo* SignificanceObject)
{
	// 7. Update the pawn LOD settings based on the new significance LOD.
	// How you map significance LOD to things like skeletal mesh LOD is something you'll need to setup in your project.
	const SignificanceLOD = SignificanceTypeInfo->GetSignificanceLOD();
}
// ---------------------------------------------------------------
 */

```