---
title: "Server와 Serverless를 동시에 제공하는 AWS 서비스들"
date: 2025-01-25 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [serverless, server, aurora-serverless, fargate, finops]
---

## 같은 기능인데 왜 두 가지 옵션이 있을까

AWS 콘솔에서 서비스를 생성하다 보면 "Provisioned"와 "Serverless" 중 하나를 고르라는 화면을 자주 마주친다. Aurora도 그렇고, OpenSearch도 그렇고, Redshift도 그렇다. 같은 기능을 하는데 왜 굳이 두 가지를 제공하는 걸까? 답은 비용 구조의 차이에 있다. Provisioned(서버 기반)는 용량을 미리 정해두고 고정으로 과금되는 방식이고, Serverless는 실제 사용량에 따라 자동으로 스케일되며 과금되는 방식이다. 워크로드 패턴에 따라 어느 쪽이 저렴한지가 갈린다.

## 대표 서비스 비교

### Aurora vs Aurora Serverless v2

Aurora Provisioned는 인스턴스 타입(db.r6g.large 등)을 지정하고 시간당 고정 비용을 낸다. 트래픽이 없어도 인스턴스가 켜져 있으면 과금된다. Aurora Serverless v2는 ACU(Aurora Capacity Unit) 단위로 0.5~128 ACU 사이에서 자동 스케일링되며, 사용한 ACU만큼만 초 단위로 과금된다(서울 리전 ACU당 시간당 $0.26).

### ECS on EC2 vs Fargate

ECS on EC2는 EC2 인스턴스를 직접 관리하면서 컨테이너를 올리는 방식이다. 인스턴스 비용을 직접 제어할 수 있고, Savings Plans/RI를 적용할 수 있다. Fargate는 인스턴스 관리 없이 vCPU와 메모리를 지정하면 AWS가 알아서 인프라를 제공한다. vCPU당 시간당 $0.04656, GB당 $0.00511(서울 리전)로 과금된다.

### OpenSearch vs OpenSearch Serverless

OpenSearch Provisioned는 인스턴스 타입과 노드 수를 지정한다. OpenSearch Serverless는 OCU(OpenSearch Compute Unit) 단위로 과금되며 최소 4 OCU(인덱싱 2 + 검색 2)가 항상 필요하다. 최소 비용이 시간당 약 $0.48이라 트래픽이 적어도 월 $350 이상이 발생한다.

### Redshift vs Redshift Serverless

Redshift Provisioned는 노드 타입과 수를 지정하고 시간 단위로 과금된다. Redshift Serverless는 RPU(Redshift Processing Unit) 단위로 쿼리 실행 시간만큼만 과금된다. 간헐적으로 분석 쿼리를 돌리는 패턴에 유리하다.

| 서비스 | Provisioned 과금 | Serverless 과금 | Serverless 최소 비용 |
|--------|-----------------|-----------------|---------------------|
| Aurora | 인스턴스 시간당 | ACU 초당 | 0.5 ACU 유지 시 ~$95/월 |
| ECS/Fargate | EC2 인스턴스 시간당 | vCPU+Memory 초당 | 태스크 수 × 시간 |
| OpenSearch | 인스턴스 시간당 | OCU 시간당 | 4 OCU ~$350/월 |
| Redshift | 노드 시간당 | RPU 초당 (실행 시만) | 쿼리 없으면 $0 |

## 언제 Server를, 언제 Serverless를 선택해야 하는가

판단 기준은 **사용 패턴의 예측 가능성**과 **트래픽의 변동폭**이다.

**Provisioned가 유리한 경우**: 사용량이 일정하고 예측 가능할 때. 24시간 꾸준히 트래픽이 있는 프로덕션 DB, 대규모 상시 워크로드. RI/SP를 적용하면 Serverless보다 단가를 훨씬 낮출 수 있다.

**Serverless가 유리한 경우**: 사용 패턴이 불규칙하고 유휴 시간이 많을 때. 개발/테스트 환경, 야간에 트래픽이 거의 없는 내부 서비스, 간헐적 분석 쿼리. 안 쓸 때 비용이 0에 가까워지므로 총 비용이 줄어든다.

**주의할 점**: Serverless가 항상 저렴한 건 아니다. OpenSearch Serverless처럼 최소 비용이 높은 경우, 소규모 워크로드에서 오히려 Provisioned(t3.small.search 등)가 저렴할 수 있다. 또한 Serverless는 콜드 스타트 지연이 발생할 수 있고, 세밀한 설정(파라미터 그룹, 커스텀 플러그인 등)이 제한되는 경우도 있다.

## 비용 시뮬레이션 예시

Aurora db.r6g.large(2 vCPU, 16GB) vs Aurora Serverless v2로 비교해보자(서울 리전 기준).

- Provisioned: $0.35/h × 730h = **$255.50/월** (상시 가동)
- Serverless (평균 4 ACU): $0.26 × 4 ACU × 730h = **$759.20/월**
- Serverless (평균 1 ACU, 야간 0.5 ACU): 약 **$190~220/월**

트래픽이 꾸준하면 Provisioned + RI가 압도적으로 저렴하다. 하지만 야간에 거의 0에 가까워지는 개발 DB라면 Serverless가 유리해진다.

## 마무리

Server(Provisioned)와 Serverless 선택은 "고정 비용을 감수하고 단가를 낮출 것인가, 변동 비용으로 유연성을 얻을 것인가"의 문제다. 사용 패턴을 먼저 분석하고, 각 옵션의 최소 비용과 예상 사용량으로 시뮬레이션한 뒤 결정하는 것이 FinOps적 접근이다.
