# Child classes from `FAssetData`

Once I needed a utility that would be able to delete all the child classes of a particular class. 
In this tutorial, let's look at an example of how to get its child classes from `FAssetData`.

## Selected Asset

Selected Assets in **Context Browser** can be received as follows:

```C++
const TArray<FAssetData> SelectedAssetData{UEditorUtilityLibrary::GetSelectedAssetData()};
```

## Generated Class

We need the names `GeneratedClass`. I know two ways to get them:

### Method one

```C++
for (const auto& AssetData : SelectedAssetData)  
{
	if (AssetData.IsValid())  
	{  
		if(const UBlueprint* Asset = Cast<UBlueprint>(AssetData.GetAsset()))    
		{    
		   const UClass* GeneratedClass = Asset->GeneratedClass;  
		   const FName GeneratedClassName = IsValid(GeneratedClass) ? GeneratedClass->GetFName() : NAME_None;  
		}
	}
}
```

Due to the `GetAsset()` function loading the asset, this method will take a long time on a large number of assets.


### Method two

In this method we get the name we want via a variable `FAssetData::TagsAndValues`.

```C++
for (const auto& AssetData : SelectedAssetData)  
{
	if (AssetData.IsValid())  
	{  
	   const FAssetDataTagMapSharedView::FFindTagResult Result = AssetData.TagsAndValues.FindTag(  
		  FBlueprintTags::GeneratedClassPath);  
	   if (Result.IsSet())  
	   {  
		  const FString ClassObjectPath = FPackageName::ExportTextPathToObjectPath(Result.GetValue());  
		  const FName GeneratedClassName = *FPackageName::ObjectPathToObjectName(ClassObjectPath);  
	   }  
	}
}
```

Next we will use the code from the second method. Let's wrap it in the function: `GetGeneratedClassName()`

```C++
FORCEINLINE FName GetGeneratedClassName(const FAssetData& AssetData) const
{
	if (AssetData.IsValid())
	{
		const FAssetDataTagMapSharedView::FFindTagResult Result = AssetData.TagsAndValues.FindTag(
			FBlueprintTags::GeneratedClassPath);
		if (Result.IsSet())
		{
			const FString ClassObjectPath = FPackageName::ExportTextPathToObjectPath(Result.GetValue());
			return *FPackageName::ObjectPathToObjectName(ClassObjectPath);
		}
	}
	return NAME_None;
}
```

## Derived Class Names

Using `AssetRegistry` we get a collection containing `DerivedClassNames`.

```C++
const FAssetRegistryModule& AssetRegistryModule = FModuleManager::LoadModuleChecked<FAssetRegistryModule>("AssetRegistry");  
const IAssetRegistry& AssetRegistry = AssetRegistryModule.Get();

TSet<FName> OutDerivedClassNames;  
AssetRegistry.GetDerivedClassNames(ClassNames.Array(), TSet<FName>{}, OutDerivedClassNames);  
```

## Get Derived

Convert the `OutDerivedClassNames` from the previous paragraph, into an array of `FAssetData`.
To do this, get all the assets from the project:

```C++
TArray<FAssetData> AssetsData;  
{  
   FARFilter Filter;  
   Filter.ClassNames = TArray<FName>{UBlueprint::StaticClass()->GetFName()};  
   Filter.bRecursiveClasses = true;  
   AssetRegistry.GetAssets(Filter, AssetsData);  
}
```

We will allocate memory for the final array in advance :

```C++
TArray<FAssetData> OutDerivedAssetData;
OutDerivedAssetData.SetNumZeroed(OutDerivedClassNames.Num());
```

In a parallel loop, going through all the assets, we will find the ones we need:

```C++
/** Get FAssetData using OutDerivedClassNames  */  
TAtomic<int32> AtomicIdx{-1};  
ParallelFor(AssetsData.Num(), [&](const int32 Idx)
	{
		FAssetData AssetData = AssetsData.IsValidIndex(Idx) ? AssetsData[Idx] : FAssetData{};
		if (AssetData.IsValid())
		{
			const FName GeneratedClassName{GetGeneratedClassName(AssetData)};
			if (!GeneratedClassName.IsNone() && OutDerivedClassNames.Contains(GeneratedClassName))
			{
				OutDerivedAssetData[++AtomicIdx] = MoveTemp(AssetData);
			}
		}
	});
	OutDerivedAssetData.RemoveAll([](const FAssetData& Item)
	{
		return !Item.IsValid();
	});
```

## Code

There is a code of the utility below that removes the children of a specific asset.

### .h

```c++
UCLASS(Blueprintable)
class EDITORASSOCIATES_API UDeleteAssestUtility : public UAssetActionUtility
{
	GENERATED_BODY()

public:
	/**
	 * @brief Deletes all children for selected assets
	 */
	UFUNCTION(CallInEditor, BlueprintCallable)
	void DeleteAllChildren() const;

public:
	UDeleteAssestUtility(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	IAssetRegistry& GetAssetRegistry() const;

	/**
	* @brief Get generated class names from asset data
	*/
	template <typename InContainer, typename OutContainer>
	decltype(std::declval<InContainer>().begin(), std::declval<OutContainer>().begin(), std::declval<InContainer>().end(),
	         std::declval<OutContainer>().end(), void())
	GetGeneratedClassNames(const InContainer& AssetsData,	OutContainer& ClassNames) const;

	UFUNCTION(BlueprintCallable)
	FName GetGeneratedClassName(const FAssetData& AssetData) const;

protected:
	/**
	 * @brief  Get the FAssetData array of all classes derived by the supplied class names
	 */
	UFUNCTION(BlueprintCallable)
	void GetDerivedClasses(const TArray<FAssetData>& SelectedAssetData, TArray<FAssetData>& OutDerivedAssetData) const;

private:
	static void DeleteAssets(TArrayView<const FAssetData> AssetDataArray);
};
```


### .cpp

```c++
UDeleteAssestUtility::UDeleteAssestUtility(const FObjectInitializer& ObjectInitializer)
   : Super(ObjectInitializer)
{
}

FORCEINLINE IAssetRegistry& UDeleteAssestUtility::GetAssetRegistry() const
{
   const FAssetRegistryModule& AssetRegistryModule = FModuleManager::LoadModuleChecked<FAssetRegistryModule>("AssetRegistry");
   return AssetRegistryModule.Get();
}

template <typename InContainer, typename OutContainer>
decltype(std::declval<InContainer>().begin(), std::declval<OutContainer>().begin(), std::declval<InContainer>().end(),
         std::declval<OutContainer>().end(), void())
UDeleteAssestUtility::GetGeneratedClassNames(const InContainer& AssetsData, OUT OutContainer& ClassNames) const
{
   ClassNames.Reserve(AssetsData.Num());
   for (const auto& AssetData : AssetsData)
   {
      FName ClassName{GetGeneratedClassName(AssetData)};
      if (!ClassName.IsNone())
      {
         ClassNames.Add(MoveTemp(ClassName));
      }
   }
}

FORCEINLINE FName UDeleteAssestUtility::GetGeneratedClassName(const FAssetData& AssetData) const
{
   if (AssetData.IsValid())
   {
      const FAssetDataTagMapSharedView::FFindTagResult Result = AssetData.TagsAndValues.FindTag(
         FBlueprintTags::GeneratedClassPath);
      if (Result.IsSet())
      {
         const FString ClassObjectPath = FPackageName::ExportTextPathToObjectPath(Result.GetValue());
         return *FPackageName::ObjectPathToObjectName(ClassObjectPath);
      }
   }
   return NAME_None;
}

void UDeleteAssestUtility::GetDerivedClasses(const TArray<FAssetData>& SelectedAssetData,
   TArray<FAssetData>& OutDerivedAssetData) const
{
   /** Get the classes of the selected Assets */
   TSet<FName> ClassNames;
   GetGeneratedClassNames(SelectedAssetData, ClassNames);
   if (ClassNames.Num() <= 0)
   {
      return;
   }

   const IAssetRegistry& AssetRegistry = GetAssetRegistry();

   /** Get the names of all classes derived by the supplied class names, excluding any classes matching the excluded class names */
   TSet<FName> OutDerivedClassNames;
   AssetRegistry.GetDerivedClassNames(ClassNames.Array(), TSet<FName>{}, OutDerivedClassNames);
   if (OutDerivedClassNames.Num() <= 0)
   {
      return;
   }

   /** Delete parent classes */
   OutDerivedClassNames = OutDerivedClassNames.Difference(ClassNames);

   TArray<FAssetData> AssetsData;
   {
      FARFilter Filter;
      Filter.ClassNames = TArray<FName>{UBlueprint::StaticClass()->GetFName()};
      Filter.bRecursiveClasses = true;
      AssetRegistry.GetAssets(Filter, AssetsData);
   }

   OutDerivedAssetData.SetNumZeroed(OutDerivedClassNames.Num());
   /** Get FAssetData using OutDerivedClassNames  */
   TAtomic<int32> AtomicIdx{-1};
   ParallelFor(AssetsData.Num(), [&](const int32 Idx)
   {
      FAssetData AssetData = AssetsData.IsValidIndex(Idx) ? AssetsData[Idx] : FAssetData{};
      if (AssetData.IsValid())
      {
         const FName GeneratedClassName{GetGeneratedClassName(AssetData)};
         if (!GeneratedClassName.IsNone() && OutDerivedClassNames.Contains(GeneratedClassName))
         {
            OutDerivedAssetData[++AtomicIdx] = MoveTemp(AssetData);
         }
      }
   });
   OutDerivedAssetData.RemoveAll([](const FAssetData& Item)
   {
      return !Item.IsValid();
   });
}

void UDeleteAssestUtility::DeleteAllChildren() const
{
   const TArray<FAssetData> SelectedAssetData{UEditorUtilityLibrary::GetSelectedAssetData()};
   if (SelectedAssetData.Num() <= 0)
   {
      return;
   }

   TArray<FAssetData> DerivedAssetData;
   GetDerivedClasses(SelectedAssetData, DerivedAssetData);
   DeleteAssets(DerivedAssetData);
}


FORCEINLINE void UDeleteAssestUtility::DeleteAssets(TArrayView<const FAssetData> AssetDataArray)
{
   if (AssetDataArray.Num() <= 0)
   {
      return;
   }

   ObjectTools::DeleteAssets(TArray<FAssetData>{AssetDataArray});
}
```
