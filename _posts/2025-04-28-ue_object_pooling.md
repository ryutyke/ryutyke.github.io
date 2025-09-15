---
title: "Object Pooling의 효과"
excerpt: "Unreal Insights로 분석, Incremental GC인데 풀링이 필요할까?"

categories:
  - UnrealEngine
tags:
  - [UnrealEngine, Object, Pooling]

permalink: /unrealengine/object_pooling/

toc: true
toc_sticky: true

date: 2025-04-28 17:30:00
last_modified_at: 2025-04-28
---
<br>

# 오브젝트 풀링

## 미리 보는 결론

오브젝트 풀링하면, UObject 개수가 줄어서 메모리 최대 사용량이 적고, 사용 메모리 양을 어느정도 예측 가능하다는 장점이 있었다.  

오브젝트 풀링을 안 하면, 오브젝트 수가 늘어나서 도달 분석이나 Destroy 등 **전체 GC에 걸리는 시간**이 늘어난다.  

그러나,  
Incremental Destroy, 그리고 5.4부터 가능한 Incremental Reachability Analysis, Incremental Gather Unreachable Objects 덕분에 오브젝트 풀링을 안 해도 GC를 한 프레임에 끝내지 않아서 히치는 잘 발생하지 않는다. (그래도 풀링 안 했을 때 결과에서 Incremental Reachability Analysis가 정해 둔 Limit Time 안에 끝나지 않아서 히치가 발생하는 경우가 있었다.)  

그러나,  
GC에 걸리는 총 시간이 늘어나기 때문에 더 많은 프레임에 GC를 하게 되고, GC하는 동안에는 안 할 때보다 **UWorld_Tick 시간**이 늘어난다. 따라서 대체로 **한 프레임 프레임마다 시간**이 늘어난다.  

즉, Incremental Reachability Analysis에서 히치가 발생하기도 하고, 발생하지 않는다고 하더라도 UWorld_Tick 시간이 늘어나는 GC가 길어지므로, 객체를 자주 생성하고 삭제하는 로직이라면 풀링을 하자. (Pool에서 꺼낼 때 뭔가 처리를 많이 해 줘야 한다면 그것도 고려해서 결정)  

(추가로, 첫 GC 때 유독 오래 걸린다. 아마 그 이후부터는 캐시 등을 사용하는 것 같다.)

<br>

---

## 결론이 나온 과정

Unseen 프로젝트(5.2버전)에서 총알을 오브젝트 풀링으로 구현했었다. 그때 Garbage Collection에서 오브젝트 Destroy로 인해 히치가 발생할 것이라고 생각해서 오브젝트 풀링을 구현했던 건데, 간단하게 풀링 전과 풀링 후를 프로파일링 했을 때 모두 Destroy에서 히치가 발생하지 않았다. 알고 보니 Incremental Destroy 때문이었다. 그래도 전체적으로 FPS가 늘어났었다. 생성(Component까지), Incremental Destroy 때문일 것 같았다.  

근데 그때, 따로 기록을 남기지 않아서 다시 했다. 또한, 이 프로젝트의 언리얼엔진을 5.4버전으로 바꿔서 5.4엔진 GC 코드를 뜯어보며 발견한 Incremental Reachability Analysis도 사용해 봤다.  

Incremental Reachability Analysis : 참조 연결된 UObject Mark하는 과정  
Incremental Gather Unreachable Objects : GUObjectArray에 있는 Object 중 Mark 안 된 애 모으는 과정  

DefaultEngine.ini에서  

```cpp
[ConsoleVariables]
gc.AllowIncrementalReachability=1      ; Incremental Reachability Analysis 활성화
gc.AllowIncrementalGather=1           ; Incremental Gather Unreachable Objects 활성화
gc.IncrementalReachabilityTimeLimit=0.002  ; 프레임당 최대 2ms로 시간 제한 설정
```

세 가지 조건에서,  

- 5.2 버전
- 5.4 버전 (Incremental Reachability Analysis, Incremental Gather Unreachable Objects 미적용)
- 5.4 버전 (Incremental Reachability Analysis, Incremental Gather Unreachable Objects 적용)

네 가지 조건으로, utrace를 뽑았다. (분석은 20발까지 할 필요가 없다고 생각해서, 초당 40발 결과만 했다.)  

- ~~풀링 안 하고 초당 20발~~
- 풀링 안 하고 초당 40발
- ~~풀링하고 초당 20발~~
- 풀링하고 초당 40발

하늘로 발사. (Lifespan될 때까지 나두기 위해)  

<div>
    <img src="/assets/images/posts_img/objectpooling/image1.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

에서, 5.4 버전 (Incremental Reachability Analysis, Incremental Gather Unreachable Objects 적용)은 메모리까지 추적해 보기로 했다.  

<div>
    <img src="/assets/images/posts_img/objectpooling/image2.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

---

# 결과

## 5.4, 풀링 O, incremental O

<div>
    <img src="/assets/images/posts_img/objectpooling/image3.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

이후에 나올 no 풀링 데이터랑 비교하면, UObject 메모리가 조금 차이나는 정도? (풀링했을 때는 40 * 3.5, 풀링 안 했을 때는 PendingKill 상태까지 해서 더 많아서?).  
풀링 했을 때는 메모리 최대 사용량이 적고, 어느정도 예측 가능하다는 장점.  

<div>
    <img src="/assets/images/posts_img/objectpooling/image4.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image5.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image6.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

두 번째 GC (38.4ms)에서는 ReachabilityAnalysis, BroadcastPostGarbageCollect가 좀 빨리 끝난다.  
**프레임당 걸린 시간을 비교해 보면 GC하는 동안에는 안 할 때보다 UWorld_Tick 시간이 늘어남. 따라서 한 프레임 시간이 늘어남.**  

## 5.4, 풀링 X, incremental O

<div>
    <img src="/assets/images/posts_img/objectpooling/image7.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image8.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image9.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

전체 GC가 567ms로 길어졌다 ( 50ms → 567ms )  

<div>
    <img src="/assets/images/posts_img/objectpooling/image10.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image11.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

이번에도 두 번째는 ReachabilityAnalysis, BroadcastPostGarbageCollect가 빨리 끝났다. (아마 처음에만 하는 뭔가가 있을 것 같다.) (38.4ms → 505.3ms)  

Incremental Begin Destroy가 되어 있어서, 여러 프레임에 걸쳐서 진행된다.  

<div>
    <img src="/assets/images/posts_img/objectpooling/image12.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image13.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image14.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image15.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image16.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image17.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

### 분석

분석 종합 : 풀링을 안 했을 때 GC에서 프레임이 낮게 나온다. 그리고 GC 첫 프레임 ConditionalCollectGarbage가 시간이 오래 걸린다. 풀링 안 했을 때 결과에서 점진적 도달 가능성 분석은 나오지 않았다. 그냥 빨리 끝나버렸다. 큰 히치가 생기지는 않았다.  

---

## Memory Trace 안 한 5.4 (그냥 5.4 결과 하나 더)

### 5.4, 풀링 O, incremental O

<div>
    <img src="/assets/images/posts_img/objectpooling/image18.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image19.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image20.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image21.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image22.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image23.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

### 5.4, 풀링 X, incremental O

<div>
    <img src="/assets/images/posts_img/objectpooling/image24.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image25.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

첫 GC 때는 GC가 2개로 나뉘어졌다. ReachabilityAnalysis가 점진적으로 일어나면 나뉘어지는 것 같다.  
기대했던 결과는 ReachabilityAnalysis를 2ms씩 점진적으로 하는 거였는데 8ms가 나왔다. TimeLimit 검사하는 게 좀 간격이 있나? **그래도 점진적으로 하긴 했다.**  

<div>
    <img src="/assets/images/posts_img/objectpooling/image26.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

히치 발생. (5.5ms 느려지면서 105fps → 66fps로 확 줄어듦)  

<div>
    <img src="/assets/images/posts_img/objectpooling/image27.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

그리고 두 번째 때는 PerformReachabilityAnalysisOnObjectsInternal (566.9 µs) ? 너무 빨라서 총알 오브젝트가 뭐 잘못됐나 싶었는데 그건 아니었다. 뭐지뭐지?  

### 분석

풀링 안 했을 때 결과에서 **점진적** 도달 가능성 분석이 나왔다. 그러나 TimeLimit 설정한 2ms의 4배인 8ms로 길게 잘렸다. 그래서 히치가 발생했다.  

## 5.4 Incremental 안 썼을 때

### Yes 풀링

<div>
    <img src="/assets/images/posts_img/objectpooling/image28.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image29.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

### No 풀링

첫 GC 때 엄청난 시간이 걸렸다.  

<div>
    <img src="/assets/images/posts_img/objectpooling/image30.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

일단 항상 첫 번째 GC가 오래 걸렸는데, 이걸 풀링도 안 하고, 점진적으로도 안 해주니 이런 일이 발생했다.  

이건 두 번째  GC.  

<div>
    <img src="/assets/images/posts_img/objectpooling/image31.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

## 5.2 (그냥 결과만)

### 5.2 Yes Pool

<div>
    <img src="/assets/images/posts_img/objectpooling/image32.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image33.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

### 5.2 No Pool

<div>
    <img src="/assets/images/posts_img/objectpooling/image34.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image35.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

### 추가적으로 궁금한 거

왜 첫 GC때 오래 걸리고, 그 이후 GC는 적게 걸리는지 궁금하다. 엔진 더 뜯어보면 정답을 찾을 수 있을 것 같다.  

가끔 GC외의 이유로 히치가 났는데, GameThreadWaitForTask였다.  

<div>
    <img src="/assets/images/posts_img/objectpooling/image36.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/objectpooling/image37.png" alt="objectpool" width="100%" min-width="100px" itemprop="image">
</div>

Unaccounted는 `SCOPED_GPU_STAT` 스코프에 포함되지 않은 작업. 다른 프레임과 비교해 보면 Lumen쪽에서 좀 오래 걸리는 것 같다.  

<br>