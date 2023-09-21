---
layout: post
title: UE实现圆锥体ConeTrace
published: true
typora-root-url: ..
---

有时候实现Gameplay逻辑的时候，会要求判断屏幕的圆是否与场景的碰撞体相交（如瞄准辅助）。屏幕的圆与场景碰撞相交本质是视锥体与场景物体是否相交。此时ConeTrace能解决此问题，然而UE并不支持，Physx貌似也没有此类基本几何体。本文意在实现快速的ConeTrace。

<img src="/assets/postasset/2023-7-27-UE实现ConeTrace/image-20230802204512708.png" alt="ConeTrace" style="zoom:50%;"/>

<center>ConeTrace在摄像机内的横截面图示</center>



## 构造凸包碰撞体方案

动态生成凸包是最万金油的做法，基本上大部分三维形状的Sweep都能做到。不过由于圆锥有弧度，生成的凸包会有一定的顶点数，最终也没有SphereTrace（内部实现是CapsuleTrace）的性能好。不过好处是能实现各种通用形状的Trace，如扇形体，平截头体等。由于暂时多考虑性能而不是通用性，暂不采用此方案。



## 基本几何体的组合方案

使用SphereTrace或者BoxTrace通过迭代多次，组合成近似的圆锥体是一种简单有效的实现方案。基本几何的Trace性能可控，做法调试也比较简单。

而组合Trace的方法又有两个：网格划分和切面划分。

网格划分指的是将屏幕圆划分成数个圆或者矩形，以圆或矩形发射SphereTrace或者BoxTrace。BoxTrace比SphereTrace的空隙更少，Trace长度长的比短的空隙更少，Trace分割（迭代）次数多的空隙比分割次数少的空隙更少。

<img src="/assets/postasset/2023-7-27-UE实现ConeTrace/image-20230803162823639.png" alt="image-20230803162823639" style="zoom:50%;" />



切面划分的方法指的是把圆锥垂直切分成数分，用SphereTrace捕获对应的Actor。此方案不会有空隙的情况，但是需要排除几何体以外的碰撞。考虑到点在圆锥内的判断算法消耗不高，用此方案更为合适。

<img src="/assets/postasset/2023-7-27-UE实现ConeTrace/image-20230803162745861.png" alt="image-20230803162745861" style="zoom:50%;" />



## 切面划分

对于敌人高密度分布均匀的场景，切面划分次数越多，性能相对来说会越好。否则，一般三五个划分足矣。这个取决于游戏类型。另外对于UE物理引擎，默认Trace会遇到Block会立刻停止，如果此碰撞位置位于圆锥外面，需要忽略掉此物理体重新发出Trace，这样会增加物理消耗。此处可以利用碰撞通道反馈的特点，把Trace的FCollisionResponseParams.CollisionResponse降级为Overlap，此时Trace始终不会停止。具体可参考官方博客

[Collision Filtering]: https://www.unrealengine.com/en-US/blog/collision-filtering

<img src="https://cdn2.unrealengine.com/blog/FilterTable-900x490-756106034.jpg" alt="Filter Table" style="zoom: 50%;" />

<center>图来自官方博客，通道会根据最小通道反馈自动降级</center>



经过切面划分的方式，结果可能有不在圆锥内的结果，因此还需要排除。排除方法也很简单，判断Trace方向与结果方向的夹角，是否小于圆锥半角即可。可以概括为下面公式：
$$
\arccos(\hat{V_t}\cdot\hat{V_d}) \leq \frac{\theta_c}{2}
$$



其中$\hat{V_t}$是圆锥顶点到搜索目标的单位向量，$\hat{V_d}$是Trace方向的单位向量，$\theta_c$为圆锥角大小。

实现代码如下

```c++
void ConeTraceSingle(const UObject* WorldContextObject, const FVector Start, const FVector End, 
                     float ConeAngle, ETraceTypeQuery TraceChannel, bool bTraceComplex,
                     const TArray<AActor*>& ActorsToIgnore, EDrawDebugTrace::Type 		
                     DrawDebugType, FHitResult& OutHit, bool bIgnoreSelf,
                     int IterateNum, FLinearColor TraceColor, FLinearColor TraceHitColor, float DrawTime)
{
	UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull);
	FVector Direction = (End - Start).GetSafeNormal();
	float Length = (End - Start).Size();
	float StepLength = Length / IterateNum;
	float ConeHalfAngle = ConeAngle / 2;
	FHitResult FinalResult;

	if (IterateNum <= 1)
	{
		UKismetSystemLibrary::SphereTraceSingle(WorldContextObject, Start, End, 0, TraceChannel, bTraceComplex,
			ActorsToIgnore, DrawDebugType, OutHit, bIgnoreSelf, TraceColor, TraceHitColor, DrawTime);
	}
	else
	{
		for (int i = 0; i < IterateNum; i++)
		{
			float ConeBottomRadius = (i + 1) * StepLength * FMath::Tan(FMath::DegreesToRadians(ConeHalfAngle));
			FVector StepStart = Start + Direction * i * StepLength;
			FVector StepEnd = Start + Direction * (i + 1) * StepLength;

			TArray<FHitResult> StepHit;
			SphereTraceMultiWithoutBlock(WorldContextObject, StepStart, StepEnd, ConeBottomRadius, TraceChannel, 				bTraceComplex, ActorsToIgnore, DrawDebugType, StepHit, bIgnoreSelf, TraceColor,FLinearColor::Black, 				DrawTime);

			if (StepHit.Num() > 0)
			{
				StepHit.Sort([StepStart](const FHitResult& A, const FHitResult& B)
				{
					return (A.ImpactPoint - StepStart).SizeSquared() < (B.ImpactPoint - StepStart).SizeSquared(); 
				});

				for (auto Hit : StepHit)
				{
					FVector ImpactVec = Hit.ImpactPoint - Start;
					float HitAngle = AngleBetweenVectors(ImpactVec, Direction);
					if (Hit.Actor != nullptr && HitAngle <= ConeHalfAngle)
					{
						FinalResult = Hit;
						break;
					}
				}
			}

			if (FinalResult.Actor != nullptr)
				break;
		}
	}

	OutHit = FinalResult;
}
```



其中`SphereTraceMultiWithoutBlock`函数是把Block降级为Overlap的方法：

```c++
void SphereTraceMultiWithoutBlock(const UObject* WorldContextObject, const FVector Start, const FVector End, float Radius, ETraceTypeQuery TraceChannel, bool bTraceComplex, const TArray<AActor*>& ActorsToIgnore, EDrawDebugTrace::Type DrawDebugType, TArray<FHitResult>& OutHits, bool bIgnoreSelf, FLinearColor TraceColor, FLinearColor TraceHitColor, float DrawTime)
{
	SCOPE_CYCLE_COUNTER(STAT_SMGSphereTrace);
	
	ECollisionChannel CollisionChannel = UEngineTypes::ConvertToCollisionChannel(TraceChannel);

	static const FName SphereTraceMultiName(TEXT("SphereTraceMulti"));
	FCollisionQueryParams Params = ConfigureCollisionParams(SphereTraceMultiName, bTraceComplex, ActorsToIgnore, bIgnoreSelf, WorldContextObject);

	FCollisionResponseParams ResponseParams;
	ResponseParams.CollisionResponse.SetAllChannels(ECR_Overlap);
	
	UWorld* World = GEngine->GetWorldFromContextObject(WorldContextObject, EGetWorldErrorMode::LogAndReturnNull);
	bool const bHit = World ? World->SweepMultiByChannel(OutHits, Start, End, FQuat::Identity, CollisionChannel, FCollisionShape::MakeSphere(Radius), Params, ResponseParams) : false;
}
```
