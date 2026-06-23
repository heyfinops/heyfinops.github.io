---
title: "비용 가시성을 확보하는 새로운 기준, Tag"
date: 2025-01-15 10:00:00 +0900
categories: [FinOps, Cost Optimization]
tags: [tagging, cost-allocation, visibility, finops]
---

## 이름표 없는 택배 상자, 누구 건지 모른다

월말에 AWS 청구서를 열었더니 EC2 비용이 $3,000이다. 그런데 이 $3,000이 어느 팀 것인지, 어떤 프로젝트에서 발생한 건지 알 수가 없다. 리소스에 이름표(태그)가 안 붙어 있으면 이런 상황이 벌어진다. 택배 상자에 받는 사람 이름이 없으면 누구 건지 모르는 것과 같다. AWS에서 태그(Tag)란 리소스에 붙이는 Key-Value 형태의 메타데이터다. `Environment: Production`, `Team: Backend`, `Project: OrderService` 같은 식이다. 이 태그가 비용 가시성의 핵심이 된다.

## Cost Allocation Tag vs 일반 태그

AWS 리소스에 붙일 수 있는 태그는 두 종류로 나뉜다. 하나는 사용자가 직접 정의하는 **User-defined Tag**이고, 다른 하나는 AWS가 자동으로 생성하는 **AWS-generated Tag**(예: `aws:createdBy`)이다. 여기서 중요한 점은, 태그를 붙였다고 해서 자동으로 비용 보고서에 반영되는 게 아니라는 것이다. Billing 콘솔에서 해당 태그를 **Cost Allocation Tag로 활성화**해야 CUR(Cost and Usage Report)과 Cost Explorer에서 필터/그룹핑 기준으로 사용할 수 있다. 활성화하지 않은 태그는 리소스 관리용으로만 존재하고, 비용 분석에는 나타나지 않는다.

| 구분 | 일반 태그 | Cost Allocation Tag |
|------|-----------|---------------------|
| 생성 방법 | 리소스에 Key-Value 부여 | 일반 태그를 Billing에서 활성화 |
| 비용 보고서 반영 | X | O |
| Cost Explorer 필터 | X | O |
| 활성화 후 반영 시점 | - | 약 24시간 소요 |
| 과거 데이터 소급 | - | 활성화 이전 데이터에는 미적용 |

## 태그 전략 설계: 어떤 키를 써야 하는가

태그를 무작정 붙이면 오히려 혼란이 커진다. "팀명을 Team으로 할까 team으로 할까 Department로 할까?" 이런 불일치가 쌓이면 결국 비용 귀속이 안 된다. 좋은 태그 전략은 조직 차원에서 표준 키를 정의하는 것부터 시작한다. 일반적으로 권장되는 태그 키는 다음과 같다.

- `Environment` — Production, Staging, Development
- `Team` 또는 `Owner` — 비용 책임을 질 팀이나 담당자
- `Project` — 프로젝트명 또는 서비스명
- `CostCenter` — 재무 부서에서 관리하는 비용 코드
- `Application` — 애플리케이션 단위 구분

키 이름은 대소문자를 통일하고(예: PascalCase), Value의 허용 값을 목록화해두면 나중에 분석할 때 훨씬 깔끔하다.

## 태깅 누락의 대가와 강제 방법

태그가 빠진 리소스는 비용 귀속이 불가능하다. Cost Explorer에서 "Team별 비용"을 보면 태그 없는 리소스는 전부 "No Tag" 또는 "Untagged"로 뭉뚱그려진다. 이게 전체 비용의 30~40%를 차지하면 사실상 비용 가시성이 없는 것과 다름없다. 그래서 태깅을 강제하는 메커니즘이 필요하다.

강제 수단으로는 세 가지가 대표적이다. 첫째, **SCP(Service Control Policy)**로 특정 태그가 없으면 리소스 생성 자체를 막을 수 있다. 둘째, **AWS Config Rules**의 `required-tags` 규칙으로 태그 미준수 리소스를 탐지하고, 자동 알림 또는 자동 태깅 조치를 할 수 있다. 셋째, **Tag Policy**(AWS Organizations 기능)로 허용되는 태그 키와 값의 범위를 조직 수준에서 규정할 수 있다.

| 강제 방법 | 시점 | 효과 |
|-----------|------|------|
| SCP | 리소스 생성 시 | 태그 없으면 생성 차단 |
| AWS Config Rules | 생성 후 탐지 | 미준수 리소스 식별 및 알림 |
| Tag Policy | 조직 수준 규정 | 허용 키/값 범위 제한 |

## 마무리

태그는 AWS 리소스에 붙이는 Key-Value 메타데이터이며, Cost Allocation Tag로 활성화해야 비용 분석에 반영된다. 조직 표준 키를 정의하고, SCP나 Config Rules로 태깅을 강제해야 비용 가시성이 확보된다. 비용을 줄이기 전에 먼저 "어디서 얼마가 나가는지" 보이게 만드는 것, 그게 태그의 역할이다.
