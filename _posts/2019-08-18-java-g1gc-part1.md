---
layout: post
title: 'Understanding Java G1GC - Part1'
subtitle: 'G1GC의 개념과 용어'
author: 'sunyoung.dev'
date: 2019-08-18
header-img: img/in-post/garbage.jpg
lang: ko
catalog: true
tags:
  - 자바
---

### Heap 메모리 구조

이전의 GC와는 다르게 Heap 영역을 (young, old) Generation 으로 분할하지 않는다. Heap 전체를 일정한 크기로 쪼개서 각 조각별로 메모리를 관리한다. 이때 쪼개진 메모리 조각을 region이라고 부른다.  region의 크기는 작게는 1MB 부터 32MB 까지 설정된다. 보통 Heap이 약 2000개로 나눠지도록 region의 크기가 정해진다. 예를 들어 메모리의 크기가 8GB 라면 region의 크기는 8GB/2000 = 4GB가 된다.   

### Region

Old, Young Generation의 개념이 아예 사라진 것은 아니다. 메모리의 크기를 정해두고 이 영역은 young generation, 나머지 메모리는 old generation이라고 정해두지 않을 뿐이지 객체의 age에 따른 generation 개념은 G1GC에도 있다. region 하나하나가 동적으로 young region이 되었다가 old region이 되었다가 한다.   

![](/img/in-post/gc-region.jpg)

> Eden : 새로 생성한 대부분의 객체가 저장되는 곳
>
> Survivor : Eden 영역에서 GC 발생 후 살아남은 객체가 이동하는 곳
>
> Old : Survivor 영역에서 GC 발생 후 계속 살아남은 객체가 이동하는 곳
>
> Humongous : region 크기의 50%가 넘는 큰 객체가 저장되는 곳 
>
> Available/unused : 새로운 region이 될 수 있는, 아무것도 할당되지 않은 곳

Eden, Survivor region을 통틀어 Young Generation이라고 하고 Old region의 집합을 Old Generation이라고 한다. 각 region은 GC가 일어남에 따라서 old region이 되기도, eden region이 되기도, survivor region이 되기도 한다.   

GC가 일어날 때는 각 region 별로 GC가 발생하는데 GC가 발생 후 살아남은 live object는 다른 region으로 copy 한다. garbage object는 region에 남고, 그 region은 다른 객체가 할당될 수 있도록 available region으로 바뀐다.   

region 별로 GC를 하면 장점이 있다.   

1. Heap 전체를 한꺼번에 정리하는 것보다 region 별로 concurrent 하게 collection을 하면 stop-the-world pause time을 줄일 수 있다.   

2. GC 이후 live object가 새로운 region으로 옮겨갈 때 한 곳에 차곡차곡 쌓을 수 있다. compaction 작업을 따로 하지 않아도 compaction을 한 효과를 볼 수 있는 것이다. 다른 GC 방식에서는 compaction 작업을 위해 전체 Heap을 스캔해야 하는 걸 생각해보면 효율적이다.

### Young GC

Young GC는 eden region이 일정 갯수 이상 할당되었을 때 발생한다. Eden 영역의 크기는 정해져 있지 않고 GC가 일어남에 따라 변하는데 이 크기가 가득 차면 Young GC를 수행한다.   

Young GC는 stop-the-world pause를 발생시키고 multi-thread로 동작한다. Young GC가 일어날 때는 Heap 영역의 모든 Young generation (Eden region, Survivor region의 집합)이 전부 collectioin의 대상이 된다.    

Age가 되지 않은 객체는 survivor영역으로 copy하고 promotion 대상 객체는 Old Generation Region으로 copy한다. 만약 survivor 영역이나 old 영역에 copy할 자리가 없다면 available region을 하나 새로 할당해서 사용한다. GC 대상 Young Generation Region은 garbage로 간주하고 Region 단위로 할당을 해지한다.   

### Old GC

Old 영역의 GC는 자바 heap 메모리의 일정 퍼센트 이상이 사용 중일 때 initial-mark가 수행되면서 발생한다. CMS는 Old 영역의 크기가 일정 퍼센트 이상 사용 중일 때 Old GC가 발생하는 반면, G1GC는 전체 Heap 크기에 의해 발생함을 주의한다. 기본 퍼센트는 45%이고 ```-XX:InitiatingHeapOccupancyPercent``` 옵션을 통해 원하는 값을 설정할 수 있다.   

Old 영역 GC가 수행될 때는 항상 Young GC가 수행될 때 같이 수행된다. Old 영역의 GC의 시작인 initial-mark 단계가 young gc가 수행될 때 같이 수행되기 때문이다.

### Mixed GC

G1GC에서는 Old GC가 Young GC가 일어날 때 같이 수행되는데 이렇게 Young GC와 Old GC가 동시에 수행되는 작업을 Mixed GC라고 한다.

### Full GC

Heap에 더이상 할당할 메모리가 없을 때 full GC가 일어난다. full GC는 자바 Heap 전체에 대한 Compaction 작업을 수행한다. 이 때 발생하는 Compaction은 Serial GC에서 수행하는 Compaction과 같이 Single Thread로 수행되며, 무지 느리다. 그래서 G1GC에서는 full gc가 일어나지 않는 것을 전제로 한다.

Humongous region은 연속된 메모리를 사용하는데 Humongous region이 사용할 연속된 메모리가 확보되지 않을 때에도 full gc가 일어난다.   


## 참고

[Oracle : Garbage first garbage collector](https://docs.oracle.com/javase/9/gctuning/garbage-first-garbage-collector.htm#JSGCT-GUID-F1BE86FA-3EDC-4D4F-BDB4-4B044AD83180)

[Plumbr : Garbage collection algorithms implementation](https://plumbr.io/handbook/garbage-collection-algorithms-implementations/g1)

[Blog : G1 Garbage Collection](https://initproc.tistory.com/entry/G1-Garbage-Collection)

[Blog : Java G1GC full gc](https://logonjava.blogspot.com/2015/08/java-g1-gc-full-gc.html)

