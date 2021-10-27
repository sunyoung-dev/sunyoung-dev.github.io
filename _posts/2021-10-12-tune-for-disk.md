---
layout: post
title: '[Elasticsearch] 디스크 사용량 튜닝'
subtitle: '엘라스틱서치 디스크 사용량을 튜닝하는 여러가지 방법'
author: 'sunyoung.dev'
date: 2021-10-12
header-img: img/in-post/disk.jpg
lang: ko
catalog: true
tags:
  - 엘라스틱서치
---

다음은 [https://www.elastic.co/guide/en/elasticsearch/reference/7.15/tune-for-disk-usage.html](https://www.elastic.co/guide/en/elasticsearch/reference/7.15/tune-for-disk-usage.html) 문서를 번역한 내용입니다.

### 사용하지 않는 기능 비활성화

기본적으로, 엘라스틱서치는 대부분의 필드가 검색되고 통계처리될 수 있도록 doc value에 추가하고 인덱싱한다. 만약 `foo` 라는 숫자 필드가 있는데 히스토그램에서는 사용하지만 filter로 사용할 일은 없다면, mapping 설정을 통해 안전하게 인덱스를 비활성화 시킬 수 있다.

```bash
PUT index
{
  "mappings": {
    "properties": {
      "foo": {
        "type": "integer",
        "index": false
      }
    }
  }
}
```

`text` 필드는 인덱스에 문서 스코어링을 용이하게 하기 위해 normalization factor를 저장한다. 만약 해당 text 필드에 의한 스코어는 사용하지 않고 매치되는지 여부만 사용한다면, `match_only_text` 타입을 사용할 수 있다. 이 필드타입은 스코어링과 포지셔닝 정보를 사용하지 않아서 꽤 많은 공간을 절약하게 된다.

### 디폴트 dynamic string mapping 사용하지 않기

기본 dynimic string mapping은 text와 keyword 타입을 모두 이용해서 인덱싱 할 것이다. 둘 중에 하나만 사용한다면 낭비이다. 보통 id 필드는 keyword만 사용하고 body 필드는 text 타입으로만 사용된다.

이 값을 비활성화하려면 명시적으로 매핑을 설정해주거나 dynamic templates을 설정하면 된다.

한 예로, 다음은 string 필드를 keyword로 매핑하기 위해 사용할 수 있는 템플릿이다.

```bash
PUT index
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings": {
          "match_mapping_type": "string",
          "mapping": {
            "type": "keyword"
          }
        }
      }
    ]
  }
}
```

### 샤드 사이즈 살펴보기

샤드가 커지면 데이터를 저장하기에는 효율적이다. 샤드의 사이즈를 키우기 위해서 인덱스를 생성할 때 프라이머리 샤드 개수를 작게 할당 할 수 있고, 인덱스의 개수를 적게 생성할 수 있고 (rolloverAPI 를 이용해서), 아니면 Shrink API를 이용해서 현재 존재하는 인덱스의 크기를 수정할 수도 있다.

샤드 사이즈가 커지는 것에 따른 단점 (긴 복구 시간 등)도 염두에 두자.

### _source 사용하지 않기

`_source` 필드는 문서의 원본을 JSON 형태로 저장한다. 이 필드에 접근하지 않는다면 비활성화 하자. 하지만 이 필드를 비활성화 하면 `_source` 필드에 접근하는 update API나 reindex API를 사용할 수 없다. 

### best_compression 사용하기

`_source` 필드와 저장된 필드들은 쉽게 무시할 수 없을 만큼의 디스크 용량을 차지한다. 이 필드들은 `best_compression` codec을 이용해 좀 더 공격적으로 압축 할 수 있다. 

### 강제 머지

엘라스틱서치의 인덱스는 한 개 이상의 샤드에 저장된다. 각각의 샤드는 루씬 인덱스 이고 한 개 이상의 세그먼트(디스크에 있는 실제 파일)로 이루어져 있다. 세그먼트가 크면 데이터를 저장하기에는 더 효율적이다.

force merge API는 샤드 당 세그먼트의 수를 줄이기 위해 사용될 수 있다. 대부분의 경우, `max_num_segments=1` 옵션을 사용하면 샤드 당 세그먼트의 수를 한 개로 줄일 수 있다.

> 강제 머지는 쓰기 작업 이 종료된 후에 실행되어야 한다. 강제 머지는 매우 큰 크기의 세그먼트를 만든다. 만약 해당 인덱스에 계속 쓰기 작업을 실행한다면 자동 머지 정책은 세그먼트가 삭제된 문서로 가득 차기 전까지 이 세그먼트의 머지 작업을 고려하지 않을 것이다. (?) 이렇게 되면 매우 큰 크기의 세그먼트가 인덱스에 남아있게 되고, 디스크 사용률은 증가하고 검색 성능은 더 느려지게 된다.
> 

### 인덱스 줄이기

shirnk API를 사용하면 인덱스의 샤드 개수를 줄일 수 있다. 위의 force merge API와 함께, 인덱스의 샤드와 세그먼트 크기를 확연히 줄일 수 있는 방법이다.

### 숫자 타입 사용하기

숫자 데이터를 저장하기 위해 고른 타입이 디스크 사용량에 큰 영향을 끼칠 수 있다. 특히 정수는 integer 타입으로 저장되는 게 좋고, 부동소수점은 scaled_float 타입으로 저장되는게 좋다. use-case에 맞게 double 대신 float을 쓰거나 float 대신 half_float 타입을 쓰면 디스크 사용량을 절약하는데에 도움이 될 것이다.

### 비슷한 문서는 같은 장소에 저장되도록 index sorting 사용하기

엘라스틱서치가 _source 필드를 저장할 때, 전체적인 압축률 향상을 위해 여러 문서를 한꺼번에 압축해서 저장한다. 예를 들면 필드명이 같은 문서는 매우 흔하고, 필드 값이 같은 경우도 종종 있다.

기본적으로 문서가 인덱스에 추가된 순서대로 같이 압축된다. index sorting을 활성화 시키면 정렬된 순서로 압축되게 된다. 비슷한 구조, 비슷한 필드, 비슷한 값으로 정렬된 문서는 압축률을 높인다.

### 같은 순서대로 필드 집어넣기

여러 문서가 압축되어서 block에 저장되기 때문에, 필드가 같은 순서로 등장하면 _source 문서에서 더 긴 중복 문자열을 발견할 가능성이 높다.

### 과거 데이터 롤업

오래된 데이터를 유지하면 나중에 분석하는 데 유용할 수 있지만 스토리지 비용이 크다. 데이터 롤업을 사용하면 raw data 저장 비용의 일부만으로 기록 데이터를 요약하고 저장할 수 있다.