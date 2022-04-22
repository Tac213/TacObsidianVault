[参考文档](https://github.com/tranek/GASDocumentation)

## 接入Gameplay Ability System

首先要在项目里启用GameplayAbilitySystem插件，在插件面板里可以完成。

然后在需要使用GAS功能的游戏模块里，在Build.cs里面增加`"GameplayAbilities", "GameplayTags", "GameplayTasks"`到`PrivateDependencyModuleNames`里。

如果版本高于4.24，还要实现一个AssetManager启用GAS的GlobalData。

头文件：

```cpp

#pragma once

#include "CoreMinimal.h"
#include "Engine/AssetManager.h"
#include "ClashOnmyojiAssetManager.generated.h"

/**
 * 
 */
UCLASS()
class CLASHONMYOJI_API UClashOnmyojiAssetManager : public UAssetManager
{
	GENERATED_BODY()

public:
	static UClashOnmyojiAssetManager& Get();

	/** Starts initial load, gets called from InitializeObjectReferences */
	virtual void StartInitialLoading() override;
	
};

```

实现文件：

```cpp


#include "ClashOnmyojiAssetManager.h"
#include "AbilitySystemGlobals.h"


UClashOnmyojiAssetManager& UClashOnmyojiAssetManager::Get()
{
	UClashOnmyojiAssetManager* This = Cast<UClashOnmyojiAssetManager>(GEngine->AssetManager);

	if (This)
	{
		return *This;
	}
	else
	{
		UE_LOG(LogTemp, Fatal, TEXT("Invalid AssetManager in DefaultEngine.ini, must be UClashOnmyojiAssetManager!"))
		return *NewObject<UClashOnmyojiAssetManager>();
	}
}


void UClashOnmyojiAssetManager::StartInitialLoading()
{
	Super::StartInitialLoading();
	// This function should be called starting with 4.24
	UAbilitySystemGlobals::Get().InitGlobalData();
}


```

最后，重新生成项目sln，编译即可。

## 接入AbilitySystemComponent

根据需要实现自己的AbilitySystemComponent，并加到对应的Actor上。

目标Actor分2种类型：

- OwnerActor: 真正持有AbilitySystemComponent实例的Actor
- AvatarActor: 只持有AbilitySystemComponent的弱引用

OwnerActor和AvatarActor可以是同一个，此时不需要使用弱引用，比如AI控制的可以使用技能的Actor可以这么处理。如果是玩家控制的Actor，而且Actor会复活，复活后的战斗属性保持不变(比如MOBA里面玩家操控的英雄)，那么OwnerActor和AvatarActor就会不同，通常OwnerActor是PlayerState而AvatarActor是玩家所操控的那个Character。

> 如果OwnerActor是PlayerState，则需要提升PlayerState的NetUpdateFrequency
>
> 同时如果使用NetUpdateFrequency的话，net.UseAdaptiveNetUpdateFrequency必须设为1
> 
> 在`Config/DefaultEngine.ini`文件中做下面的设置即可完成
> 
> 如果还是不行，则需要在引擎的`Engine/Config/ConsoleVariables.ini`文件中像[这样](https://github.com/EpicGames/UnrealEngine/commit/002a1bebb69660c16b4f2f0f8b0f43520e818cd1#diff-7f69b5e66f7365484672aeb4e44691f46a89674b05df4587a02ad49369360ec6)把`net.UseAdaptiveNetUpdateFrequency=0`删掉

```ini
[SystemSettings]
net.UseAdaptiveNetUpdateFrequency=1
```

可以在Actor的构造函数里面像下面这样构造一个AbilitySystemComponent:

```cpp
AbilitySystemComponent = CreateDefaultSubobject<UCombatUnitAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
AbilitySystemComponent->SetIsReplicated(true);
AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);
if (GetWorld())
{
	AbilitySystemComponent->RegisterComponent();
}
```

代码里有个ReplicationMode，解释如下：


Replication Mode | 什么时候使用 | 描述
---|---|---
Full | 单个玩家 | 每个GameplayEffect都会replicate到每个客户端
Mixed | 多个玩家, 玩家控制的Actor | GameplayEffect只会replicate到owning client，而GameplayTags和GameplayCues则依然会replicate到每个客户端
Minimal | 多个玩家, AI控制的Actor | GameplayEffect不会replicate，而GameplayTags和GameplayCues则依然会replicate到每个客户端


> 使用Mixed模式时，OnwerActor的Owner应当是PlayerController。PlayerState的Owner时PlayerController，而Character的Onwer不是，所以当玩家使用ASC时，最好由PlayerState持有ASC并使用Mixed模式。如果使用Mixed模式但是ASC的OwnerActor不是PlayerState，那么应当在OwnerActor的实现代码里调SetOwner()，把OwnerActor的Owner设成一个合适的PlayerController


AbilitySystemComponent需要初始化，告诉ASC你的OwnerActor和AvatarActor分别是哪个。初始化调用的是ASC的IntAbilityActorInfo函数，第一个参数传的是OwnerActor的实例，第二个参数传的是AvatarActor的实例。

**服务端和客户端都需要初始化**。对于OwnerActor和AvatarActor是同一个的Actor，而且不用Mixed模式，则在BeginPlayer里初始化就可以了：

```cpp
// Called when the game starts or when spawned
void ACombatUnit::BeginPlay()
{
	Super::BeginPlay();

	AbilitySystemComponent->InitAbilityActorInfo(this, this);
}
```

对于OwnerActor和AvatarActor是同一个的Actor，但使用Mixed模式，则相对复杂一些，服务端可以在PossedBy函数里面初始化**并SetOwner**:

```cpp
void APACharacterBase::PossessedBy(AController * NewController)
{
	Super::PossessedBy(NewController);

	if (AbilitySystemComponent)
	{
		AbilitySystemComponent->InitAbilityActorInfo(this, this);
	}

	// ASC MixedMode replication requires that the ASC Owner's Owner be the Controller.
	SetOwner(NewController);
}
```

客户端则是在PlayerController的AcknowledgePossession函数里初始化：

```cpp
void APAPlayerControllerBase::AcknowledgePossession(APawn* P)
{
	Super::AcknowledgePossession(P);

	APACharacterBase* CharacterBase = Cast<APACharacterBase>(P);
	if (CharacterBase)
	{
		CharacterBase->GetAbilitySystemComponent()->InitAbilityActorInfo(CharacterBase, CharacterBase);
	}

	//...
}
```

对于OwnerActor和AvatarActor不是同一个的Actor，则**服务端**可以在AvatarActor的PossessedBy里面初始化(仍然需要注意Mixed模式的问题，下面的代码OwnerActor是PlayerState所以不需要SetOwner)：

```cpp
// Server only
void AGDHeroCharacter::PossessedBy(AController * NewController)
{
	Super::PossessedBy(NewController);

	AGDPlayerState* PS = GetPlayerState<AGDPlayerState>();
	if (PS)
	{
		// Set the ASC on the Server. Clients do this in OnRep_PlayerState()
		// AvatarActor的AbilitySystemComponent是一个TWeakObjectPtr
		AbilitySystemComponent = Cast<UGDAbilitySystemComponent>(PS->GetAbilitySystemComponent());

		// AI won't have PlayerControllers so we can init again here just to be sure. No harm in initing twice for heroes that have PlayerControllers.
		PS->GetAbilitySystemComponent()->InitAbilityActorInfo(PS, this);
	}
	
	//...
}
```

客户端则在AvatarActor的OnRep_PlayerState里初始化：

```cpp
// Client only
void AGDHeroCharacter::OnRep_PlayerState()
{
	Super::OnRep_PlayerState();

	AGDPlayerState* PS = GetPlayerState<AGDPlayerState>();
	if (PS)
	{
		// Set the ASC for clients. Server does this in PossessedBy.
		AbilitySystemComponent = Cast<UGDAbilitySystemComponent>(PS->GetAbilitySystemComponent());

		// Init ASC Actor Info for clients. Server will init its ASC when it possesses a new Actor.
		AbilitySystemComponent->InitAbilityActorInfo(PS, this);
	}

	// ...
}
```

如果遇到报错信息：`LogAbilitySystem: Warning: Can't activate LocalOnly or LocalPredicted ability %s when not local!`，说明ASC没有在客户端初始化。

有AbilitySystemComponent的Actor(无论OwnerActor或AvatarActor)还要实现IAbilitySystemInterface接口，主要是为了实现下面这个函数给GAS系统调用：

```cpp
virtual UAbilitySystemComponent* GetAbilitySystemComponent() const override;
```

### AbilitySystemComponent上的GameplayEffects和GameplayAbilities

ASC有个属性是`FActiveGameplayEffectsContainer ActiveGameplayEffects`，里面存着所有当前激活的GameplayEffects。

ASC有个属性是`FGameplayAbilitySpecContainer ActivatableAbilities`，里面存着所有当以已被授予的GameplayAbilities。可以通过`ActivableAbilities.Items`遍历ActivableAbilities，但是必须增加`ABILITYLIST_SCOPE_LOCK();`在for循环前面，示例代码：

```cpp
ABILITYLIST_SCOPE_LOCK();
for (FGameplayAbilitySpec& Spec : ActivatableAbilities.Items)
{
	if (Spec.Ability)
	{
		Spec.Ability->OnAvatarSet(AbilityActorInfo.Get(), Spec);
	}
}
```