---
layout: post
title:  "[UE5,UMG] 중앙에서 가장 가까운 위젯을 고정해주는 ScrollBox"
tags: UE5, UScrollBox
---
예전에 만들어 뒀던걸 정리하는 포스트 

![image](/assets/img/RefAnimation.gif)*이런 형태의 지정한 곳에서 가장 가까운 자식 위젯을 끌어당기며 강조하는 위젯이 필요하다.*

1. ### 스크롤의 0번 자식 위젯과 (Num-1)번 자식 위젯이 최상단 혹은 최하단에 위치하지 않도록 공간 만들기
```cpp
void SAutoSelectScrollBox::FillEmptyArea() const
{
	const FVector2D EmptySize2D = CachedGeometry.GetLocalSize() * FVector2D(0.5f, 0.5f);
	for (int32 i = 0; i < 2; i++)
	{
	    auto NewSlotArguments = SScrollBox::Slot();
	    SScrollBox::FSlot& NewSlot = *NewSlotArguments.GetSlot();
	    const TSharedRef<SSpacer> Spacer = SNew(SSpacer).Size(EmptySize2D);
	    NewSlot
	    [
		  Spacer
	    ];
	    ScrollPanel->Children.InsertSlot(MoveTemp(NewSlotArguments), i == 0 ? 0 : ScrollPanel->Children.Num());
	}	
}
```
 해당 함수는 ```tick```에서 ```SAutoSelectScrollBox```의```CachedGeometry```가 채워질 때 까지 기다렸다가 호출된다. (위젯의 Geometry가 계산되기 전에 위젯의 크기를 가져오려 하면 0을 반환하기 때문)




2. ### 스크롤이 끝나면 현재 스크롤 상태에서 중앙과 가장 가까운 위젯을 찾기
```cpp
UWidget* SAutoSelectScrollBox::FindTargetWidget() const
{
	const TArray<UWidget*>& WidgetList = Parent->GetAllChildren();
	int32 idx = 0;
	TOptional<float> WidgetSize;
	TOptional<float> ScrollOffset;
	const bool bIsVertical = Orientation == Orient_Vertical;
	UWidget* Widget = nullptr;
	do
	{
		if (!WidgetList.IsValidIndex(idx))
			break;
		UWidget* elem = WidgetList[idx];
		
		TSet<TSharedRef<SWidget>> WidgetsToFind;
		{
			WidgetsToFind.Add(elem->TakeWidget());
		}
		TMap<TSharedRef<SWidget>, FArrangedWidget> Result;
		FindChildGeometries(CachedGeometry, WidgetsToFind, Result);
		if (const FArrangedWidget* WidgetGeometry = Result.Find(elem->TakeWidget()))
		{
			auto LocalSize = WidgetGeometry->Geometry.GetLocalSize();
			auto FindWidgetVector = CachedGeometry.AbsoluteToLocal(WidgetGeometry->Geometry.GetAbsolutePosition()) + (LocalSize / 2);
			auto MyVector = CachedGeometry.GetLocalSize() * FVector2D(0.5f, 0.5f);
			const float WidgetPosition = bIsVertical ? FindWidgetVector.Y : FindWidgetVector.X;
			const float MyPosition = bIsVertical ? MyVector.Y : MyVector.X;
			if (!ScrollOffset.IsSet() || FMath::Abs(ScrollOffset.GetValue()) > FMath::Abs(WidgetPosition - MyPosition))
			{
				Widget = elem;
				ScrollOffset = WidgetPosition - MyPosition;
			}
			if (!WidgetSize.IsSet())
			{
				WidgetSize = bIsVertical ? LocalSize.Y : LocalSize.X;
			}
		}
		idx++;
	}
	while (idx < WidgetList.Num() && WidgetSize.IsSet() ? FMath::Abs(ScrollOffset.IsSet() ? ScrollOffset.GetValue() : 0) >= WidgetSize.GetValue() / 2 : true);
	return Widget;
}
```
##### 각 자식 위젯에 대해 다음을 수행한다.
* 위젯 위치 계산
* 위젯의 위치와 스크롤박스의 중앙 위치 사이 거리 계산
* 이 거리가 위젯의 절반 크기보다 작으면 해당 위젯을 반환




3. ### 찾은 위젯을 중앙으로 옮기기
```cpp
void SAutoSelectScrollBox::ScrollModify()
{
	if (!IsOk())
		return;
		
	if (WidgetToFind)
	{
		ScrollDescendantIntoView(WidgetToFind->GetCachedWidget(), true, EDescendantScrollDestination::Center);
		WidgetToFind = nullptr;
		return;
	}


	if (!IsCenter(TargetWidget))
	{
		const auto FindWidget = FindTargetWidget();
		if (TargetWidget != FindWidget)
		{
			OnAutoUnSelectedScrollBoxItem.Broadcast(TargetWidget);
			TargetWidget = FindWidget;
			OnAutoSelectedScrollBoxItem.Broadcast(TargetWidget);
		}
		if (TargetWidget)
			ScrollDescendantIntoView(TargetWidget->GetCachedWidget(), true, EDescendantScrollDestination::Center);
	}
}
```
--
* 스크롤 박스 위치를 수정할 수 있는지 확인```(!bTouchPanningCaputre&&!bIsScrolling)```
* ```WidgetToFind```가 설정되어 있는지 확인 (설정되어 있다면 ```ScrollDecendantIntoView``` 함수를 호출하여 이동)
* 현재 ```TargetWidget```이 중앙에 있는지 확인하고 중앙에 위치하지 않다면 ```FindTargetWidget``` 함수를 호출하여 새 타겟을 찾고 새 타겟이 현재 타겟과 다르면 ```OnAutoUnSelectedScrollBoxItem``` 이벤트를 발생시켜 현재 타겟 위젯의 선택이 해제되었음을 알리고, ```OnAutoSelectedScrollBoxItem```이벤트를 발생시켜 새로운 타겟 위젯이 선택되었음을 알린다.
* 그리고 ```ScrollDecendantIntoView``` 를 호출하여 다시 중앙 위치로 조정한다.


4. ### 나머지
이제 따로 구현한 스크롤 아이템 위젯에 위젯과 스크롤 센터의 갭을 계산해서 각 위젯으로 보내 애니메이션을 재생하거나 위젯의 사이즈를 수정하면 된다.






![image](https://github.com/piraxis2/AutoSelectScroll_UE5/raw/master/Animation.gif)*완성*

[깃헙링크](https://github.com/piraxis2/AutoSelectScroll_UE5)

