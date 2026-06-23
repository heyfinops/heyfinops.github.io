---
title: "S3의 Storage Class 관리, 심화편"
date: 2025-02-04 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [s3, intelligent-tiering, storage-class, cost-optimization, finops]
---

## Lifecycle만으로 충분할까

기본편에서 Lifecycle Policy로 일수 기반 자동 전환을 다뤘다. 하지만 현실의 데이터는 "30일 지나면 접근 안 한다"처럼 깔끔하게 패턴이 나뉘지 않는 경우가 많다. 3개월 전 파일을 갑자기 다시 읽기도 하고, 최신 파일인데도 한 번도 안 읽히는 것도 있다. 접근 패턴을 예측하기 어려운 데이터에는 Intelligent-Tiering이 답이 될 수 있다. 하지만 공짜 점심은 없다. 모니터링 비용과 전환 비용이라는 숨겨진 비용을 이해하고 써야 한다.

## Intelligent-Tiering의 동작 원리

Intelligent-Tiering은 객체별 접근 패턴을 모니터링하고, 접근 빈도에 따라 자동으로 티어를 이동시킨다.

| 티어 | 조건 | 저장 비용 (서울, GB-월) |
|------|------|---------------------|
| Frequent Access | 최근 30일 내 접근 | $0.025 (Standard 동일) |
| Infrequent Access | 30일 미접근 | $0.0138 |
| Archive Instant Access | 90일 미접근 (옵트인) | $0.005 |
| Archive Access | 90일 미접근 (옵트인) | $0.0045 |
| Deep Archive Access | 180일 미접근 (옵트인) | $0.002 |

30일 동안 접근하지 않으면 자동으로 IA 티어로 내려가고, 다시 접근하면 즉시 Frequent Access로 복귀한다. 복귀 시 검색 비용이 없다는 게 Standard-IA와 다른 점이다. Standard-IA는 읽을 때 $0.01/GB 검색 비용이 붙지만, Intelligent-Tiering은 티어 간 이동 시 추가 비용이 없다.

대신 모니터링 비용이 있다. 객체당 월 $0.0025(1,000객체당 $2.50)가 부과된다. 이것이 "공짜가 아닌" 이유다.

## 모니터링 비용의 함정: 소형 객체 문제

모니터링 비용은 객체 수에 비례한다. 1GB짜리 파일 1개와 1KB짜리 파일 1,000개는 저장 공간은 비슷하지만, 모니터링 비용은 1,000배 차이가 난다.

예를 들어 128KB 미만의 소형 객체 100만 개를 Intelligent-Tiering에 넣으면:
- 모니터링 비용: 100만 × $0.0000025 = $2.50/월
- 실제 저장 데이터: 100만 × 50KB = 약 50GB
- IA 티어 절감액: 50GB × ($0.025 - $0.0138) = $0.56/월

모니터링 비용($2.50)이 절감액($0.56)보다 크다. 게다가 128KB 미만 객체는 Intelligent-Tiering에서도 항상 Frequent Access 요금이 적용되어 티어 이동의 혜택을 받지 못한다. 소형 객체가 많은 버킷에는 Intelligent-Tiering이 오히려 손해다.

## 전환 비용(Transition Cost) 함정

Lifecycle Policy로 Storage Class를 전환할 때마다 전환 요청 비용이 발생한다.

| 전환 대상 | 전환 비용 (1,000건당) |
|----------|-------------------|
| Standard-IA | $0.01 |
| One Zone-IA | $0.01 |
| Glacier Instant Retrieval | $0.02 |
| Glacier Flexible Retrieval | $0.05 |
| Deep Archive | $0.05 |
| Intelligent-Tiering | $0.01 |

파일 100만 개를 Standard에서 Glacier Instant로 전환하면: 1,000,000 / 1,000 × $0.02 = $20. 한 번이면 괜찮지만, Lifecycle에서 Standard → IA → Glacier Instant → Deep Archive로 단계별 전환을 설정하면 매 단계마다 전환 비용이 발생한다. 파일 수가 많을 때는 단계를 줄이거나, Standard에서 최종 목적지로 바로 전환하는 게 유리할 수 있다.

## Lifecycle vs Intelligent-Tiering 선택 기준

둘 다 비용을 줄여주는 도구지만, 적합한 상황이 다르다.

| 판단 기준 | Lifecycle Policy | Intelligent-Tiering |
|----------|-----------------|-------------------|
| 접근 패턴 | 예측 가능 (시간이 지나면 접근 감소) | 예측 불가능, 불규칙 |
| 객체 크기 | 크기 무관 | 128KB 이상 권장 |
| 객체 수 | 많아도 전환 비용만 1회 | 많으면 모니터링 비용 증가 |
| 재접근 가능성 | 낮음 | 높음 (비용 없이 복귀) |
| 관리 복잡도 | 규칙 설계 필요 | 설정 후 자동 |

## 실전 조합 전략

실무에서는 둘을 조합해서 쓰는 것이 최선이다.

- **로그 데이터**: Lifecycle (접근 패턴이 명확 → 시간순 감소)
- **사용자 업로드 파일**: Intelligent-Tiering (언제 다시 접근할지 예측 불가)
- **데이터 레이크**: Intelligent-Tiering + Archive 티어 옵트인 (대용량, 접근 불규칙)
- **백업/스냅샷**: Lifecycle → Glacier Deep Archive (재접근 거의 없음)

버킷 레벨이 아니라 Prefix 레벨로 Lifecycle과 Intelligent-Tiering을 분리 적용하면, 하나의 버킷 안에서도 데이터 특성에 맞는 전략을 병행할 수 있다.

## 마무리

Intelligent-Tiering은 접근 패턴을 예측할 수 없을 때 강력하지만, 소형 객체가 많으면 모니터링 비용이 절감액을 초과할 수 있다. Lifecycle은 패턴이 명확한 데이터에 효과적이지만, 전환 비용과 단계 수를 고려해야 한다. 데이터 특성을 먼저 파악하고, 적합한 도구를 선택하는 것이 S3 비용 최적화의 완성이다.
