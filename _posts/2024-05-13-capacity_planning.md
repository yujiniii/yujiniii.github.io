---
title: "[가상면접 사례로 배우는 대규모 시스템 설계] 규모 추정"
author: yujiniii
date: 2024-05-13 01:00:00 +0900
categories: [study, 가상면접 사례로 배우는 대규모 시스템 설계]
tags: ["infra"]
img_path: "/assets/img/posts/PN0/"
pin: false
---
> 가상 면접 사례로 배우는 대규모 시스템 설계를 스터디하고 있습니다.
> 내용정리보단 기록하고 싶은 부분만 블로그로 작성합니다. 

책 2장 개략적인 규모측정중 **예제: 트위터 QPS와 저장소 요구량 측정 방법** 을 보고 흥미로워서 좀 더 찾아보았습니다. 

## 개요

#### **MAU와 DAU**
1. MAU(Monthly Active Users): 한 달 동안 시스템을 사용하는 활성 사용자 수입니다.
2. DAU(Daily Active Users): 하루 동안 시스템을 사용하는 활성 사용자 수입니다.
이러한 지표를 사용하여 시스템이 얼마나 많은 사용자를 처리해야 하는지 추정할 수 있습니다.


#### **요청량 추정**
DAU를 기반으로 사용자가 시스템에 보내는 평균 요청(또는 쿼리) 수를 추정합니다.   
예를 들어, DAU가 100,000명이고 각 사용자가 평균적으로 10개의 요청을 보낸다고 가정하면, 하루 총 요청량은 1,000,000(100,000 * 10)입니다.

#### **QPS**
1. QPS(Query Per Second)는 시스템이 초당 처리해야 하는 쿼리 수입니다.
2. peak QPS 는 가정에서 나올 수 있는 최대 QPS입니다. 



## 책 예제: 트위터 QPS와 저장소 요구량 측정 방법
(이 수치는 책에서 임의로 정한 값으로 트위터와 무관합니다.)

#### 가정
- 월간 능동 사용자(monthly active user)는 3억(300milion) 명이다.
- 50%의 사용자가 트위터를 매일 사용한다.
- 평균적으로 각 사용자는 매일 2건의 트윗을 올린다
- 미디어를 포함하는 트윗은 10% 정도다.
- 데이터는 5년간 보관된다.


#### 추정
- **QPS(Query Per Second) 추정치**
  - DAU = 3억 * 50% = 1.5억(150million)
  - QPS = 1.5억 * (2트윗) / (24시간) / (3600초) = 약 3500
  - Peak QPS = 2 * QPS = 약 7000 `(100%의 사용자가 트위터를 매일 사용한다고 가정)`

- **미디어 저장을 위한 저장소 요구량**
  - 평균 트윗 크기
    - tweet_id에 64바이트 (uuid)
    - 텍스트에 140바이트
    - 미디어에 1MB
  - 미디어 저장소 요구량: 1.5억 * 2 * 10% * 1MB = 30,000,000MB = 30,000GB = 30TB/일
  - 5년간 미디어를 보관하기 위한 저장소 요구량: 30TB * 365 * 5 = 약 55PB


## 문제풀어보기(from, chatGPT)
#### 가정 
1. DAU = 500,000
사용자마다 평균적으로 하루에 4번의 읽기 쿼리와 1번의 쓰기 쿼리를 보냅니다.
이 때 초당 요청량(QPS)은 얼마입니까?

2. MAU = 1,000,000
사용자마다 평균적으로 한 달에 10번의 읽기 쿼리와 3번의 쓰기 쿼리를 보냅니다.
이 때 초당 요청량(QPS)은 얼마입니까?

3. DAU = 2,000,000
사용자마다 평균적으로 하루에 6번의 읽기 쿼리와 2번의 쓰기 쿼리를 보냅니다.
이 때 초당 요청량(QPS)은 얼마입니까?

4. 온라인 스토어
   - 한 온라인 스토어에서의 월간 이용자 수는 1천만 명입니다.
   - 이 중 30%의 이용자가 매일 이 온라인 스토어를 방문합니다.
   - 하루에 평균 3번의 주문이 이루어집니다.
   - 이 온라인 스토어의 상품 사진은 전체 상품의 20%를 차지하고 있습니다.
   - 데이터는 3년간 보관됩니다.
     - 상품 ID에 16바이트
     - 상품 설명에 200바이트
     - 상품 이미지에 500KB
   - 이 온라인 스토어의 상품 사진을 위한 일일 저장소 요구량은 얼마입니까? 
   - 이 온라인 스토어의 상품 사진을 3년간 보관하기 위해 필요한 총 저장소 용량은 얼마입니까?


#### 정답
1. (DAU * 쿼리 수)/(24시간) = 500,000 * (4 + 1) / (24 * 60 * 60) = 28.9 QPS
2. (MAU * 쿼리 수)/(30일*24시간) = 1,000,000 * (10 + 3) / (30 * 24 * 60 * 60) = 5.01 QPS
3. (DAU * 쿼리 수)/(24시간) = 2,000,000 * (6 + 2) / (24 * 60 * 60) = 185.18 QPS
4. 정답
   - (1) 이 온라인 스토어의 상품 사진을 위한 일일 저장소 요구량은 얼마입니까? 
     - 상품 사진 데이터의 크기는 상품 ID에 16바이트, 상품 설명에 200바이트, 상품 이미지에 500KB로 가정합니다
     - 상품 사진 데이터 하나의 크기는 16B + 200B + 500KB 입니다.
     - 하루에 이루어지는 주문 수는 1일에 3번으로 가정합니다.
     - 하루에 업로드되는 상품 사진의 총 크기는 3 * (16B + 200B + 500KB) 입니다.
     - 따라서, 일일 저장소 요구량은 다음과 같이 계산됩니다.
       - 일일 저장소 요구량 = 3 * (16B + 200B + 500KB) = 3 * (16 + 200 + 500,000)B
       - = 1,500,648B
       - = 약 1.5 MB
   - (2) 이 온라인 스토어의 상품 사진을 3년간 보관하기 위해 필요한 총 저장소 용량은 얼마입니까?
     - 1년은 365일로 계산합니다.
     - 따라서, 3년 동안의 총 저장소 요구량은 다음과 같이 계산됩니다:
      - 총 저장소 용량 = 1.5MB * 365 * 3  = 1,642 MB
      - = 약 1.64 GB




### 함께 읽어보면 좋을 아티클
[capacity-planning](https://blog.bytebytego.com/p/capacity-planning)