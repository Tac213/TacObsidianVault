行为树的task节点可以通过C++或Blueprint自定义，Blueprint自定义task节点可以参考[offical quick start guide](https://docs.unrealengine.com/5.0/en-US/behavior-tree-in-unreal-engine---quick-start-guide/)。

C++需要继承UBTTaskNode，如果需要在节点中配置黑板key的话，可以直接继承UBTTask_BlackboardBase。当然不继承也是可以的，把直接继承UBTTaskNode然后把UBTTask_BlackboardBase的逻辑抄过来就可以了，参考UBTTask_WaitBlackboardTime。

## 显示相关配置

在constructor里面可以自定义节点名。

header:

```cpp
public:  
   UBTTask_TryToAttackEnemy(const FObjectInitializer& ObjectInitializer);
```

cpp file:

```cpp
UBTTask_TryToAttackEnemy::UBTTask_TryToAttackEnemy(const FObjectInitializer& ObjectInitializer)  
   : Super(ObjectInitializer)  
{  
   NodeName = "Try To Attack Enemy";  
}
```

重写GetStaticDescription可以自定义节点描述。

header:

```cpp
public:
    virtual FString GetStaticDescription() const override;
```

cpp file:

```cpp
FString UBTTask_TryToAttackEnemy::GetStaticDescription() const  
{  
   return FString::Printf(TEXT("Try to attack: %s"), *BlackboardKey.SelectedKeyName.ToString());  
}
```

## 节点执行逻辑

实现ExecuteTask即可。

header:

```cpp
public:
    virtual EBTNodeResult::Type ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory) override;
```

cpp file:

```cpp
EBTNodeResult::Type UBTTask_TryToAttackEnemy::ExecuteTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory)  
{  
    const UBlackboardComponent* OwnerBlackboard = OwnerComp.GetBlackboardComponent();  
    if (OwnerBlackboard && BlackboardKey.SelectedKeyType == UBlackboardKeyType_Object::StaticClass())  
    {
        if (ACombatUnit* TargetUnit = Cast<ACombatUnit>(OwnerBlackboard->GetValue<UBlackboardKeyType_Object>(BlackboardKey.GetSelectedKeyID())))  
        {
            if (ACombatUnit* SelfUnit = Cast<ACombatUnit>(OwnerComp.GetOwner()))  
            {
                if (SelfUnit->GetDistanceTo(TargetUnit) > SelfUnit->AttackSightRadius)  
                {
                    return EBTNodeResult::Type::Failed;  
                }
                FGameplayTagContainer AttackTagContainer;   
                AttackTagContainer.AddTag(FGameplayTag::RequestGameplayTag("Ability.Spell.Attack"));  
                if (SelfUnit->AbilitySystemComponent->TryActivateAbilitiesByTag(AttackTagContainer))  
                {
                    SelfUnit->AbilitySystemComponent->OnAbilityEnded.AddUObject(this, &UBTTask_TryToAttackEnemy::OnAttackEnded);  
                    return EBTNodeResult::Type::InProgress;  
                }
                return EBTNodeResult::Type::Failed;  
            }
        }
        else  
        {  
            return EBTNodeResult::Type::Failed;  
        }
    }
    return EBTNodeResult::Type::Failed;  
}
```

返回值是个枚举:

```cpp
UENUM(BlueprintType)  
namespace EBTNodeResult  
{  
   // keep in sync with DescribeNodeResult()  
   enum Type  
   {  
      // finished as success  
      Succeeded,  
      // finished as failure  
      Failed,  
      // finished aborting = failure  
      Aborted,  
      // not finished yet  
      InProgress,  
   };
}
```

主要就是使用Succeeded Failed InProgress这3种返回值。

Succeeded和Failed都会把执行权交还给行为树，由行为树根据返回值决定下一步要做什么，InProgess则会在当前节点卡住，直到节点调用下面这个函数:

```cpp
FinishLatentTask(OwnerComp, bWasAttackCancelled ? EBTNodeResult::Type::Failed : EBTNodeResult::Type::Succeeded);
```

需要把UBehaviorTreeComponent传回去，并告知当前节点的最终执行结果。

通常来说会返回InProgress的节点都需要使用Tick函数来判断任务结束没有。如果节点需要tick，需要在contructor里配置:

```cpp
UBTTask_TryToAttackEnemy::UBTTask_TryToAttackEnemy(const FObjectInitializer& ObjectInitializer)  
   : Super(ObjectInitializer)  
{   
   bNotifyTick = true;  
}
```

同时实现TickTask方法。

header:

```cpp
virtual void TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds) override;
```

cpp file:

```cpp
void UBTTask_TryToAttackEnemy::TickTask(UBehaviorTreeComponent& OwnerComp, uint8* NodeMemory, float DeltaSeconds)  
{  
   if (bAttackEnded)  
   {      FinishLatentTask(OwnerComp, bWasAttackCancelled ? EBTNodeResult::Type::Failed : EBTNodeResult::Type::Succeeded);  
   }
}
```

## Read Blackboard

如果继承UBTTask_BlackboardBase，可以通过BlackBoardKey这个成员变量来读取黑板。

比如下面的代码可以判断行为树配置的黑板键类型:

```cpp
if (BlackboardKey.SelectedKeyType == UBlackboardKeyType_Object::StaticClass())
{
    // do something
}
else if (BlackboardKey.SelectedKeyType == UBlackboardKeyType_Int::StaticClass())
{
    // do something
}
```

下面的代码可以获取行为树配置的黑板键对应的值:

```cpp
const UBlackboardComponent* OwnerBlackboard = OwnerComp.GetBlackboardComponent();
UObject* ObjectValue = OwnerBlackboard->GetValue<UBlackboardKeyType_Object>(BlackboardKey.GetSelectedKeyID());
```

下面的代码可以获取行为树配置的黑板键的名字:

```cpp
FName KeyName = BlackboardKey.SelectedKeyName;
```
