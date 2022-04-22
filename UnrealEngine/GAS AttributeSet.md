AttributeSet用于定义战斗数值，应当继承UAttributeSet。

在定义自己的AttributeSet之前，需要准备好一个宏：

```cpp
// Uses macros from AttributeSet.h
#define ATTRIBUTE_ACCESSORS(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
	GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName)

```

后面会解释这个宏的作用，然后继承UAttributeSet定义自己的AttributeSet。

```cpp
UCLASS()
class CLASHONMYOJI_API UCombatUnitAttributeSetBase : public UAttributeSet
{
	GENERATED_BODY()
}
```

接着就可以再这个上面定义战斗数值，比如：

```cpp
UPROPERTY(VisibleAnywhere, BlueprintReadOnly, Category = "Health", ReplicatedUsing = OnRep_MaxHealth)
FGameplayAttributeData MaxHealth;
ATTRIBUTE_ACCESSORS(UCombatUnitAttributeSetBase, MaxHealth)
```

`ATTRIBUTE_ACCESSORS`宏通过调用其他4个宏，给我们提供了以下这4个函数，用来获取属性对象、获取&设置&初始化属性的值，如下面的源代码所示：

```cpp
#define GAMEPLAYATTRIBUTE_PROPERTY_GETTER(ClassName, PropertyName) \
	static FGameplayAttribute Get##PropertyName##Attribute() \
	{ \
		static FProperty* Prop = FindFieldChecked<FProperty>(ClassName::StaticClass(), GET_MEMBER_NAME_CHECKED(ClassName, PropertyName)); \
		return Prop; \
	}

#define GAMEPLAYATTRIBUTE_VALUE_GETTER(PropertyName) \
	FORCEINLINE float Get##PropertyName() const \
	{ \
		return PropertyName.GetCurrentValue(); \
	}

#define GAMEPLAYATTRIBUTE_VALUE_SETTER(PropertyName) \
	FORCEINLINE void Set##PropertyName(float NewVal) \
	{ \
		UAbilitySystemComponent* AbilityComp = GetOwningAbilitySystemComponent(); \
		if (ensure(AbilityComp)) \
		{ \
			AbilityComp->SetNumericAttributeBase(Get##PropertyName##Attribute(), NewVal); \
		}; \
	}

#define GAMEPLAYATTRIBUTE_VALUE_INITTER(PropertyName) \
	FORCEINLINE void Init##PropertyName(float NewVal) \
	{ \
		PropertyName.SetBaseValue(NewVal); \
		PropertyName.SetCurrentValue(NewVal); \
	}

```

RepNotify函数也是调一个宏，是一个固定写法，比如：

```cpp
void UCombatUnitAttributeSetBase::OnRep_Health(const FGameplayAttributeData& PreviousValue)
{
	GAMEPLAYATTRIBUTE_REPNOTIFY(UCombatUnitAttributeSetBase, Health, PreviousValue);
}
```

当然还要把战斗数值加到GetLifetimeReplicatedProps，这是属性同步的基础。不过REPNOTIFY的类型最好用Always，这样即使服务端下发的属性的值和客户端相同，客户端的RepNotify也会被调用，参考下面的代码：

```cpp
void UCombatUnitAttributeSetBase::GetLifetimeReplicatedProps(TArray<FLifetimeProperty>& OutLifetimeProps) const
{
	Super::GetLifetimeReplicatedProps(OutLifetimeProps);

	DOREPLIFETIME_CONDITION_NOTIFY(UCombatUnitAttributeSetBase, Health, COND_None, REPNOTIFY_Always);
	DOREPLIFETIME_CONDITION_NOTIFY(UCombatUnitAttributeSetBase, MaxHealth, COND_None, REPNOTIFY_Always);
}
```

AttributeSet在ASC的OwnerActor的构造函数上实例化，这样的话AttributeSet就会自动注册到ASC上，只能在C++中完成这一步骤。

```cpp
AbilitySystemComponent = CreateDefaultSubobject<UCombatUnitAbilitySystemComponent>(TEXT("AbilitySystemComponent"));
AbilitySystemComponent->SetIsReplicated(true);
AbilitySystemComponent->SetReplicationMode(EGameplayEffectReplicationMode::Minimal);
if (GetWorld())
{
	AbilitySystemComponent->RegisterComponent();
}

AttributeSet = CreateDefaultSubobject<UCombatUnitAttributeSetBase>(TEXT("AttributeSet"));
```

然后需要初始化AttributeSet的基础属性。和ASC的初始化一样，同样需要在双端初始化。初始化有很多种方法，官方推荐使用一个GameplayEffect来初始化属性，代码类似下面这样(下面代码中`DefaultAttributes`是`TSubClassOf<UGameplayEffect>`)

```cpp
void AGDCharacterBase::InitializeAttributes()
{
	if (!AbilitySystemComponent.IsValid())
	{
		return;
	}

	if (!DefaultAttributes)
	{
		UE_LOG(LogTemp, Error, TEXT("%s() Missing DefaultAttributes for %s. Please fill in the character's Blueprint."), *FString(__FUNCTION__), *GetName());
		return;
	}

	// Can run on Server and Client
	FGameplayEffectContextHandle EffectContext = AbilitySystemComponent->MakeEffectContext();
	EffectContext.AddSourceObject(this);

	FGameplayEffectSpecHandle NewHandle = AbilitySystemComponent->MakeOutgoingSpec(DefaultAttributes, GetCharacterLevel(), EffectContext);
	if (NewHandle.IsValid())
	{
		FActiveGameplayEffectHandle ActiveGEHandle = AbilitySystemComponent->ApplyGameplayEffectSpecToTarget(*NewHandle.Data.Get(), AbilitySystemComponent.Get());
	}
}
```

或者参考`AttributeSet.h`里面的方法，也就是官方文档的方法，用一个DataCurve。

我的方法是参考`AttributeSet.h`里面的方法，调AbilitySystemComponent->SetNumericAttributeBase(AttributeToModify, FieldValue)来初始化，不过数据来源是一个DataTable:

```cpp
void FCombatUnitAttributeSetInitializer::InitializeCombatUnitAttribute(UAbilitySystemComponent* AbilitySystemComponent, const FString& UnitId) const
{
	if (!IsValid(AbilitySystemComponent))
	{
		return;
	}
	const UDataTable* CombatUnitDataTable = LoadObject<UDataTable>(nullptr, DataTablePath);
	if (CombatUnitDataTable == nullptr)
	{
		return;
	}
	TSubclassOf<UAttributeSet> AttributeSetClass = UCombatUnitAttributeSetBase::StaticClass();
	FCombatUnitData* CombatUnitData = CombatUnitDataTable->FindRow<FCombatUnitData>(FName(UnitId), "");
	for (TFieldIterator<FProperty> FieldIterator(CombatUnitData->StaticStruct()); FieldIterator; ++FieldIterator)
	{
		FProperty* Field = *FieldIterator;
		FString AttributeName = Field->GetName();
		FProperty* Property = FindFProperty<FProperty>(*AttributeSetClass, *AttributeName);
		if (!IsSupportedProperty(Property))
		{
			continue;
		}
		if (uint8* ValuePtr = Field->ContainerPtrToValuePtr<uint8>(CombatUnitData))
		{
			if (FNumericProperty* NumProp = CastField<FNumericProperty>(Field))
			{
				if (NumProp->IsFloatingPoint())
				{
					float FieldValue = NumProp->GetFloatingPointPropertyValue(ValuePtr);
					FGameplayAttribute AttributeToModify(Property);
					AbilitySystemComponent->SetNumericAttributeBase(AttributeToModify, FieldValue);
				}
			}
		}
	}
	AbilitySystemComponent->ForceReplication();
}
```