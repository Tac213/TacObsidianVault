AAIController是AController的派生类，使用AI控制Pawn时需要在Pawn的constructor中配置:

```cpp
AIControllerClass = ACombatUnitAIController::StaticClass();
AutoPossessAI = EAutoPossessAI::Spawned;
bUseControllerRotationYaw = true;
```

如果要在possess时运行行为树，可以在AIController的OnPossess函数中完成。

header:

```cpp
public:
    virtual void OnPossess(APawn* InPawn) override;
```

cpp file:

```cpp
void ACombatUnitAIController::OnPossess(APawn* InPawn)  
{  
   Super::OnPossess(InPawn);  
   if (ACombatUnit* CombatUnit = Cast<ACombatUnit>(InPawn))  
   {
       RunBehaviorTree(CombatUnit->BehaviorTree);  
   }
}
```

## 检测Actor进入视野范围

需要通过UAIPerceptionComponent完成。

header:

```cpp
public:
    UPROPERTY(VisibleAnywhere, BlueprintReadOnly)  
    TObjectPtr<UAIPerceptionComponent> AIPerceptionComponent;
```

cpp file:

```cpp
ACombatUnitAIController::ACombatUnitAIController()  
{  
   AIPerceptionComponent = CreateDefaultSubobject<UAIPerceptionComponent>(TEXT("AIPerceptionComponent"));  
   UAISenseConfig_Sight* AISightConfig = NewObject<UAISenseConfig_Sight>(this, UAISenseConfig_Sight::StaticClass(), TEXT("Combat Unit AI Sight Config"));  
   AISightConfig->DetectionByAffiliation.bDetectNeutrals = 1;  
   AIPerceptionComponent->ConfigureSense(*AISightConfig);  
}
```

上面这一步也可以在蓝图里完成，具体可以参考[offical quick start guide](https://docs.unrealengine.com/5.0/en-US/behavior-tree-in-unreal-engine---quick-start-guide/)。

有了AIPerceptionComponent就可以监听Actor进入视野范围。首先声明一个Delegate:

```cpp
protected:  
   void OnTargetPerceptionUpdated(AActor* Target, FAIStimulus Stimulus);
```

接着构造函数里监听OnTargetPerceptionUpdated事件:

```cpp
AIPerceptionComponent->OnTargetPerceptionUpdated.AddDynamic(this, &ACombatUnitAIController::OnTargetPerceptionUpdated);
```

Delegate实现如下:

```cpp
void ACombatUnitAIController::OnTargetPerceptionUpdated(AActor* Target, FAIStimulus Stimulus)  
{  
    if (Stimulus.WasSuccessfullySensed())  
    {
        // do something
    }
}
```
