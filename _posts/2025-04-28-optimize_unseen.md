---
title: "Project_Unseen 프레임드랍 해결"
excerpt: "Stat 명령어, GPU Visualizer"

categories:
  - UnrealEngine
  - pfselect
tags:
  - [UnrealEngine, Stat, GPU, Visualizer]

permalink: /unrealengine/optimize_project_unseen/

toc: true
toc_sticky: true

date: 2025-04-28 18:00:00
last_modified_at: 2025-04-28
---
<br>



<div>
    <img src="/assets/images/posts_img/optimizeunseen/image1.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

일반 상황에는 16.67ms  

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image2.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image3.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

보스가 스킬 썼을 때 Game Thread, GPU Time이 올라가는 모습  

Game Thread는 왜 올라가지?  

런타임 CPU 히치 검출을 위해 **Stat Dumphitches**를 썼다.  

결과 : 229 hitches       73655ms total hitch time  


**Stat SceneRendering** 했더니  
스킬 쓸 때 파편들이 많아서 그런지 drawcall이 엄청나게 많이 일어난다.  

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image14.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

**그리고 스킬 1→2→3 갈수록 Game이 늘어남.**  
**gpu time은 스킬 파편 쳐다보면 일어남.**  

<br>

### 문제 찾기

**Stat Game**  

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image4.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

파편 터질 때 world tick time이랑 tick time 엄청나게 늘어남  

gpu visualizer (ctrl + shift + ,)  

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image5.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

기본은 13.56  

스킬 터질 때 16.19  

#### 늘어난 항목들  

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image6.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image7.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image8.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image9.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

PrePass DDM_AllOpaque에서는 DepthPassParallel이 문제였고  

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image10.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

BasePass에서는 BasePassParallel  

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image11.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

Translucency에서는 Translucency(BeforDistortion Parallel) 1576x846 이 문제였다.  

<br>

---

나이아가라가 문제인 거 같았다.  

확인하기 위해, **Stat Niagara**을 했다.  
(**stats.MaxPerGroup**으로 보이는 칸 개수 조절 가능)  

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image12.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

### 최적화

Niagara의  
주 축이 되는 돌맹이를 제외하고 부스러기들은 Collision을 끄니깐 CPU 사용이 쭉 줄어들었다. (collision을 cpu로 계산한다)  

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image13.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

NE_Smoke 빼보자 : 반투명이라 GPU 많이 사용한다.  

다른 건 안 없어지고,  

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image15.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

Translucency는 정상화됐다.  

그리고 필요이상으로 spawn rate가 높은 것 같아서 줄였다.  

그랬더니  

<div>
    <img src="/assets/images/posts_img/optimizeunseen/image16.png" alt="optimize" width="100%" min-width="100px" itemprop="image">
</div>

좋아졌다!

<br>