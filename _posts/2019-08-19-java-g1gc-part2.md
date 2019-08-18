---
layout: post
title: 'Understanding Java G1GC - Part2'
subtitle: 'G1GC 로그와 사이클'
author: 'sunyoung.dev'
date: 2019-08-19
header-img: img/in-post/garbage.jpg
lang: ko
catalog: true
tags:
  - 자바
---

### G1GC

Time based : ```-XX:MaxGCPauseMillis=200``` 옵션을 통해 Max Pause Time을 설정할 수 있다.

### Data Structure

**Collections Sets (CS)** : Collection의 대상이 되는 region의 목록이다. Collections set 단위로 GC가 수행된다. Collection set에는 항상 모든 Young Generation 영역이 포함되고, 일부 Old Generation 영역이 포함되기도 포함되지 않기도 한다. Collection의 대상은 목표 GC Pause time에 얼마나 적합하느냐, Live Objects를 얼마나 가지고 있느냐에 따라 결정된다. Collection set에 Garbage를 많이 가지고 있는 Region이 먼저 할당되기 때문에 G1GC의 이름이 Garbage First Collector 인 것이다.

**Remembered Sets (RS)** : Region 별로 가지고 있는 바깥으로부터 참조에 대한 목록이다. 이 목록을 가지고 있기 때문에 Gc Pause Time이 얼마나 걸릴지 예측할 수 있다.



## G1GC Cycle

G1GC 사이클은 Evacuation Pause Phase와 Concurrent Marking Phase로 구성된다.   

Evacuation Pause 단계에서는 application 실행을 멈추고 multi-thread로 collection set의 region을 정리한다. 이 단계에서 Heap의 모든 Young Generation(Eden+Survivor region의 집합)에 대한 Young GC가 수행되고 일부 Old Region에 대한 GC가 수행되기도 한다.  

Evacuation Pause 단계에서 Old GC (Initial-mark)가 수행되었으면 Concurrent Marking 단계가 수행된다. Young GC만 일어났다면 이 단계는 건너뛰고 다음 Evacuation Pause 단계가 실행된다. Concurrent Marking 단계에서는 CMS의 Concurrent Marking 단계처럼 application 실행과 함께 concurrent 하게 live objects를 마킹한다. 이 단계를 통해 Mixed GC의 대상이 될 객체를 찾으면 다음 Evcuation Pause 단계에서 Mixed GC가 일어난다.   

### Evacuation Pause
![](/img/in-post/gc-evacuation.jpg)

Evacuation Pause 단계에서는 young GC만 수행되기도 하고, initial-mark가 같이 수행되기도, mixed GC가 수행되기도 한다.

#### Eden, Survivor 영역 변화
> [Eden: 140.0M(140.0M)->0.0B(277.0M) Survivors: 4096.0K->11264.0K Heap: 146.3M(240.0M)->13087.6K(480.0M)]

Eden 영역의 크기가 140.0M 에서 277.0M로 늘어났고 Survivor 영역도 4096.0K 에서 11264.0K로 늘어났다. G1GC에서는 Young, Old Generation의 영역의 위치가 고정되어있지 않을 뿐만 아니라 Young, Old Generation의 크기도 고정되어 있지 않고 변화한다.


### Concurrent Marking
![](/img/in-post/gc-marking.jpg)

Concurrent Marking 단계에서는 live objects를 마크하고, 빈 region이 발견되면 Cleanup 단계에서 바로 할당 해제한다. live objects도 있고 garbage object도 있는 region이 있다면 다음 단계의 Evacuation Pause 단계에서 Mixed GC가 수행된다.   

G1GC에서 Mixed GC가 Old 영역의 메모리를 해제하는 주된 방법이기 때문에 Heap 영역이 꽉 차기 전에 Concurrent Marking 단계가 수행되는 것이 중요하다. 이 단계가 마무리 되기 전에 Old 영역 메모리 할당에 실패한다면 Full GC가 일어날 것이다.

#### 1. Initial marking
> 6.562: [GC pause (G1 Evacuation Pause) (young) (initial-mark), 0.1264036 secs]   

stop-the-world pause를 발생시키며 GC Root가 참조하는 모든 객체를 마킹한다. 이 작업은 Young GC가 일어날 때 어차피 수행되어야 하는 작업이기 때문에 Young GC가 일어날 때 같이 수행한다. (piggy-backed) 그래서 매우 빠르게 완료된다.

#### 2. Concurrent root region scanning
> 6.688: [GC concurrent-root-region-scan-start]   
> 6.728: [GC concurrent-root-region-scan-end, 0.0395895 secs]

application 실행과 concurrent하게 실행된다. initial-mark에서 살아남은 survivor region이 참조하는 객체들을 스캔한다. 이 작업은 Evcuation Pause가 일어나기 전에 완료해야 하는데, Evacuation Pause가 일어나면 initial-mark가 새로 수행될 것이고 그러면 survivor region이 변경되기 때문이다.

#### 3. Concurrent marking
> 6.728: [GC concurrent-mark-start]   
> 6.790: [GC concurrent-mark-end, 0.0625920 secs]

CMS의 concurrent marking 과 같이 multi-thread로 application 실행과 동시에 heap 전체의 live objects를 마킹한다.

#### 4. Remarking
> 6.791: [GC remark 6.791: [Finalize Marking, 0.0010554 secs] 6.792: [GC ref-proc, 0.0003267 secs] 6.792: [Unloading, 0.0124570 secs], 0.0142482 secs]

stop-the-world pause를 발생시키며 수행된다. concurrent marking 단계가 application 실행과 동시에 수행되느라, 새로 참조가 생긴 live objects를 추가적으로 마킹한다.

#### 5. Cleanup
> 6.805: [GC cleanup 19745K->19745K(480M), 0.0013692 secs]

live objects가 하나도 없는 region은 바로 reclaimed 되어서 avaialble region으로 메모리 할당을 해제한다. live objects도 있고 garbage object도 있는 region이 있다면 다음 단계의 Evacuation Pause 단계에서 Mixed GC가 수행된다. 


## 참고

[Oracle : Understanding G1GC logs](https://blogs.oracle.com/poonam/understanding-g1-gc-logs)

[Redhat : Collecting and reading G1GC logs part2](https://www.redhat.com/en/blog/collecting-and-reading-g1-garbage-collector-logs-part-2)

[Garbage First GC](http://www.informit.com/articles/article.aspx?p=2496621&seqNum=5)