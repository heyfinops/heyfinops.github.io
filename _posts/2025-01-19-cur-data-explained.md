---
title: "CUR 데이터엔 어떤 정보가 들어 있나요?"
date: 2025-01-19 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [cur, cost-usage-report, billing, finops]
---

## 청구서보다 10배 상세한 보고서가 있다

AWS 비용을 분석하다 보면 Cost Explorer만으로는 한계를 느끼게 된다. "이 비용이 정확히 어떤 리소스에서 나왔는지", "약정 할인이 어디에 적용됐는지" 같은 디테일이 필요할 때가 있다. 이때 등장하는 것이 CUR(Cost and Usage Report)이다. CUR은 AWS가 제공하는 가장 상세한 비용 데이터 소스로, 모든 사용량과 비용 정보가 행 단위로 기록된 거대한 CSV(또는 Parquet) 파일이다. Cost Explorer가 "요약본"이라면, CUR은 "원장"에 해당한다.

## CUR의 주요 컬럼 구조

CUR 파일을 열면 수백 개의 컬럼이 존재한다. 전부 외울 필요는 없고, 핵심 카테고리를 이해하면 된다. 주요 컬럼 그룹은 다음과 같다.

- **identity/** — 고유 식별자. `identity/LineItemId`로 각 행을 구분한다.
- **bill/** — 청구 계정 정보. `bill/PayerAccountId`, `bill/BillingPeriodStartDate` 등.
- **lineItem/** — 핵심 비용 정보. `lineItem/UsageType`, `lineItem/Operation`, `lineItem/UsageAmount`, `lineItem/UnblendedCost` 등이 여기에 있다.
- **product/** — 서비스 및 리소스 메타데이터. `product/region`, `product/instanceType`, `product/operatingSystem` 등.
- **pricing/** — 단가 정보. `pricing/publicOnDemandRate`, `pricing/term`(OnDemand/Reserved) 등.
- **reservation/** — RI 관련 정보. `reservation/ReservationARN`, `reservation/EffectiveCost` 등.
- **savingsPlan/** — SP 관련 정보. `savingsPlan/SavingsPlanRate`, `savingsPlan/SavingsPlanEffectiveCost` 등.
- **resourceTags/** — 리소스에 붙은 태그 정보. `resourceTags/user:Team` 형태로 표시된다.

| 컬럼 그룹 | 주요 정보 | 활용 예시 |
|-----------|-----------|-----------|
| lineItem | 사용량, 비용, 사용 유형 | 서비스별 비용 집계 |
| product | 인스턴스 타입, 리전, OS | 리소스 레벨 분석 |
| pricing | 공시 단가, 약정 단가 | 할인율 계산 |
| reservation / savingsPlan | 약정 적용 내역 | Coverage/Utilization 계산 |
| resourceTags | 태그 키-값 | 팀별/프로젝트별 비용 귀속 |

## CUR로 할 수 있는 분석

CUR 데이터가 있으면 Cost Explorer에서 불가능한 심층 분석이 가능하다. 첫째, **리소스 레벨 비용 추적**이다. Cost Explorer는 서비스 단위까지만 드릴다운되지만, CUR은 개별 리소스 ARN까지 식별할 수 있다. 둘째, **약정 효과 정밀 분석**이다. 어떤 RI/SP가 어떤 인스턴스에 적용됐는지, 얼마만큼 절감됐는지를 행 단위로 확인할 수 있다. 셋째, **커스텀 Chargeback**이다. 태그 기반으로 팀별, 프로젝트별 비용을 정밀하게 배분할 수 있다. 넷째, **이상 비용 탐지**다. 일별 비용 추이를 세밀하게 모니터링해서 갑작스러운 증가를 빠르게 잡아낼 수 있다.

## CUR vs Cost Explorer

| 비교 항목 | Cost Explorer | CUR |
|-----------|---------------|-----|
| 데이터 상세도 | 서비스/계정 단위 | 리소스/행 단위 |
| 분석 도구 | 웹 콘솔 내장 | Athena, QuickSight, 외부 BI |
| 설정 난이도 | 별도 설정 불필요 | S3 버킷 설정, Athena 연동 필요 |
| 비용 | 무료 (API 호출 제한 있음) | S3 저장 비용 + Athena 쿼리 비용 |
| 과거 데이터 | 최대 12개월 | 설정 이후 무제한 보관 가능 |
| 약정 상세 | 요약 수준 | 행 단위 적용 내역 |

## CUR 설정 방법

CUR을 활성화하려면 Billing 콘솔 → Cost & Usage Reports → Create Report 순서로 진행한다. 보고서 이름을 지정하고, 데이터를 저장할 S3 버킷을 선택한다. 권장 설정은 다음과 같다. 파일 형식은 Parquet(압축률 높고 Athena 쿼리 성능 우수), 시간 단위는 Hourly(가장 상세), Resource ID 포함 옵션 체크, 그리고 데이터 통합(Data Integration)에서 Athena를 선택하면 자동으로 Glue 테이블이 생성된다. 설정 후 첫 데이터가 도착하기까지 최대 24시간이 소요되며, 이후에는 하루에 여러 번 업데이트된다.

## 마무리

CUR은 AWS 비용의 모든 디테일이 담긴 원천 데이터다. lineItem, product, pricing, reservation 등의 컬럼 그룹을 이해하면 리소스 레벨 분석부터 약정 효과 검증, 팀별 Chargeback까지 가능하다. Cost Explorer로 큰 그림을 보고, CUR로 정밀 분석을 하는 것이 FinOps 실무의 기본 조합이다.
