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

enum class ECmSignificanceType : uint8
{
	// 
	BigWorld_SimObjectDefault,

	// 对应大世界中有实体Actor的NPCs
	BigWorld_NPCDefault,

	// 对应大世界的所有城市
	BigWorld_CityDefault,

	// 非战斗场景的NPCs
	NonBattleScene_NPCDefault,

	// 战斗场景步兵
	BattleScene_InfantryDefault,

	// 战斗场景骑兵
	BattleScene_DragoonDefault,

	Max,
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
	struct FBigWorldSimObjectDefaultSignificanceTypeData: public FSignificanceTypeData
	{
		float MyData;

		FBigWorldSimObjectDefaultSignificanceTypeData(): MyData(1.0f){}
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

	struct FBigWorldSimObjectInfo : public USignificanceManager::FManagedObjectInfo
	{
	public:
		FBigWorldSimObjectInfo(UObject* InObject, FName InTag): USignificanceManager::FManagedObjectInfo(InObject, InTag, &BigWorldSimObjectDefaultSignificance,
			USignificanceManager::EPostSignificanceType::Sequential, &BigWorldSimObjectDefaultPostSignificance), SignificanceLOD(0)
		{}

		int32 GetSignificanceLOD() const { return SignificanceLOD; }

		static float BigWorldSimObjectDefaultSignificance(const FManagedObjectInfo* ObjectInfo, const FTransform& Viewpoint);
		static void BigWorldSimObjectDefaultPostSignificance(const FManagedObjectInfo* ObjectInfo, float OldSignificance, float NewSignificance, bool bFinal);

		//add-on handling for pre-post update operations
		static void BigWorldSimObjectDefaultPreSignificanceUpdate(UCMSignificanceManager* SignificanceManager);
		static void BigWorldSimObjectDefaultPostSignificanceUpdate(UCMSignificanceManager* SignificanceManager);
	public:
		int32 SignificanceLOD;
	};
	
	// Define Object Data End
public:
	// Overridable function to update the managed objects' significance
	virtual void Update(TArrayView<const FTransform> Viewpoints) override;

	//This is the function that users call as a replacement for the virtual RegisterObject which exposes only the significance type so it can be looked up
	void RegisterSignificanceObject(UObject* Object, ECmSignificanceType SignificanceType);

	FSignificanceTypeInfo* GetSignificanceTypeInfo(ECmSignificanceType SignificanceType);
private:
	float ElapsedTime;

	//static storage of all the significance handling data
	UCMSignificanceManager::FSignificanceTypeInfo SignificanceTypeInfo[(uint8)ECmSignificanceType::Max];
};


```

```C++

CMSignificanceManager.cpp

// Fill out your copyright notice in the Description page of Project Settings.


#include "GameManager/CMSignificanceManager.h"

static float GcmSignificanceManagerTickIntervalTime = 0;
static FAutoConsoleVariableRef CVarSignificanceManagerTickIntervalTime(
	TEXT("CMSigMan.TickIntervalSeconds"),
	GcmSignificanceManagerTickIntervalTime,
	TEXT("0 means call update function per frame.\n"),
	ECVF_Default
	);

UCMSignificanceManager::UCMSignificanceManager()
	:Super()
{
	ElapsedTime = 0;

	if (IsTemplate() /*Only populate static struct in the CDO*/)
	{
		FSignificanceTypeInfo& SimObjectDefaultInfo = SignificanceTypeInfo[(uint8)ECmSignificanceType::BigWorld_SimObjectDefault];
		SimObjectDefaultInfo.Tag = "BigWorld.SimObjectDefault";
		//can leave significance functions unmodified if they aren't used...
		SimObjectDefaultInfo.PostSignificanceType = EPostSignificanceType::Sequential;
		SimObjectDefaultInfo.Data = MakeUnique<FBigWorldSimObjectDefaultSignificanceTypeData>();
		//Also register update functions, if they exist for this tag
		SimObjectDefaultInfo.PreUpdateFunction = &FBigWorldSimObjectInfo::BigWorldSimObjectDefaultPreSignificanceUpdate;
		SimObjectDefaultInfo.PostUpdateFunction = &FBigWorldSimObjectInfo::BigWorldSimObjectDefaultPostSignificanceUpdate;
	}
}

// Calculate significance based on distance and other factors
float UCMSignificanceManager::FBigWorldSimObjectInfo::BigWorldSimObjectDefaultSignificance(
	const FManagedObjectInfo* ObjectInfo, const FTransform& Viewpoint)
{
	/*
	* FPlayerPawnObjectInfo* PlayerInfo = static_cast<FPlayerPawnObjectInfo*>(ObjectInfo);
		
		// Grab any data from the actor we may need here
		const APlayerPawn* PlayerPawn = CastChecked<const APlayerPawn>(GetObject());
 
		// You can have conditions here to force max significance based on the pawn properties (eg. they're the local player).
 
		// Calculate significance based on things like distance or whatever makes sense for your project.
		return CalculateSignificanceForDistance(...);
	 */
	return 0;
}

// This is called per-object by USignificanceManager::Update after the significance values have been calculated.
// This has the option to be run in a parallel for loop if you have thread safe operations you can run to apply the new significance values.
// Or you can run it sequentially, see EPostSignificanceType.
// 
// If the order of the actors doesn't matter then it's better to use this than the PostUpdate above as it removes a level of indirection.
void UCMSignificanceManager::FBigWorldSimObjectInfo::BigWorldSimObjectDefaultPostSignificance(
	const FManagedObjectInfo* ObjectInfo, float OldSignificance, float NewSignificance, bool bFinal)
{
}

// Called once in UMySignificanceManager::Update before USignificanceManager::Update runs.
void UCMSignificanceManager::FBigWorldSimObjectInfo::BigWorldSimObjectDefaultPreSignificanceUpdate(
	UCMSignificanceManager* SignificanceManager)
{
	// This is a good place to pre-calcuate any common data you may need during the per-object significance calculations so you can avoid doing them for every object. You likely won't need this though.
}

// Called once in UMySignificanceManager::Update after USignificanceManager::Update runs.
// The key difference between this function and the PostSignificance function you give to FManagedObject is this is called
// after USignificanceManager has sorted the list by significance.
void UCMSignificanceManager::FBigWorldSimObjectInfo::BigWorldSimObjectDefaultPostSignificanceUpdate(
	UCMSignificanceManager* SignificanceManager)
{
	// This is where we apply the Significance LOD to each player.
	// We define these as basically a large list of ints where there is 1 entry per player, then we use the player's index in the sorted list that USignificanceManager::Update created to lookup their Significance LOD in that list.
/*
	const UCMSignificanceManager::FSignificanceTypeInfo* PlayerTypeInfo = UCMSignificanceManager::GetSignificanceTypeInfo(EMyProjectSignificanceType::Player);
	const TArray<USignificanceManager::FManagedObjectInfo*>& ManagedPlayers = SignificanceManager->GetManagedObjects(PlayerTypeInfo->Tag);
 
	// Loop over the list of players, sorted by significance.
	for (int32 SignificanceIndex = 0; SignificanceIndex < ManagedPlayers.Num(); ++SignificanceIndex)
	{
		// Budget is an FOrderedBudget which is an engine type.
		const int32 SignificanceLOD = PlayerTypeInfo->Budget.GetBudgetForIndex(SignificanceIndex);
 
		FPlayerPawnObjectInfo* PlayerInfo = Cast<FPlayerPawnObjetInfo>(ManagedPlayers[SignificanceIndex]);
			
		// 6. Notify the pawn about the new significance LOD value.
		APlayerPawn* PlayerPawn = static_cast<APlayerPawn*>(GetObject());
		if (NewSignificanceLOD != PlayerInfo->SignificanceLOD)
		{
			PlayerPawn->ApplySignificanceLOD(this);
			PlayerInfo->SignificanceLOD = NewSignificanceLOD;
		}
	}
*/
}

void UCMSignificanceManager::RegisterSignificanceObject(UObject* Object, ECmSignificanceType SignificanceType)
{
	check(SignificanceType != ECmSignificanceType::Max);

	// TODO 处理没有注册的情况；
	
	const FSignificanceTypeInfo& Info = SignificanceTypeInfo[(uint8)SignificanceType];

	if (SignificanceType == ECmSignificanceType::BigWorld_SimObjectDefault)
	{
		// UMyTag1Object* MyTag1Object = CastChecked<UMyTag1Object>(Object);
		// if (!MyTag1Object->ShouldManageSignificance() /* || some other checks */)
		// {
		// 	return;
		// }
		
		RegisterManagedObject(new FBigWorldSimObjectInfo(Object, Info.Tag));
	}
}



UCMSignificanceManager::FSignificanceTypeInfo* UCMSignificanceManager::GetSignificanceTypeInfo(
	ECmSignificanceType SignificanceType)
{
	return &SignificanceTypeInfo[(uint8)SignificanceType];
}

void UCMSignificanceManager::Update(TArrayView<const FTransform> Viewpoints)
{
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