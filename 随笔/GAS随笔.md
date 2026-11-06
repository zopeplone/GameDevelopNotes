
# AttributeData
每个AttributeData，都由BaseValue和CurrentValue组成，BaseValue为基础值，CurrentValue由BaseValue计算得来，且此时保存的CurrentValue是一个Cache（避免每次Get时都进行所有Modifiers的计算），真正的CurrentValue要从BaseValue得出，需要聚合所有的Modifiers来计算
```cpp
struct FGameplayAttributeData{
	float BaseValue;
	float CurrentValue;
};
```
`FGameplayAttribute`是一个“地址”，是可以持久化，且可以指向真正的AttributeData，也是外部获取AttributeData的唯一途径
```cpp
struct FGameplayAttribute{
	TFieldPath<FProperty> Attribute;
	TObjectPtr<UStruct> AttributeOwner;
}
```
