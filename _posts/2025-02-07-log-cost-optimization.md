---
title: "AWS 서비스의 로그를 최적화하는 법"
date: 2025-02-07 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [cloudwatch-logs, logging, cost-optimization, finops]
---

## 로그 비용, 얼마나 나가고 있는지 확인해본 적 있는가

CloudWatch Logs 비용이 월 청구서에서 상위권을 차지하는 경우가 의외로 많다. 애플리케이션 로그, Lambda 실행 로그, ECS 컨테이너 로그 등을 CloudWatch Logs로 보내면 수집 비용(Ingestion)과 저장 비용(Storage)이 동시에 발생한다. 서울 리전 기준 수집 $0.76/GB, 저장 $0.0314/GB-월이다. 하루 10GB 로그를 쌓는 환경이면 수집만 월 $228, 무기한 저장하면 저장 비용이 매월 $9.42씩 계속 누적된다. 1년이면 저장비만 $56이 추가로 쌓인다.

## CloudWatch Logs 비용 구조

| 비용 항목 | 단가 (서울) | 비고 |
|----------|-----------|------|
| 수집 (Ingestion) | $0.76/GB | 로그가 들어올 때 1회 과금 |
| 저장 (Storage) | $0.0314/GB-월 | 보존 기간 동안 계속 과금 |
| Logs Insights 쿼리 | $0.0076/GB 스캔 | 쿼리할 때마다 스캔한 데이터에 과금 |
| Vended Log 수집 | $0.114/GB | 일반 로그 대비 85% 할인 |

수집 비용은 한 번 발생하면 끝이지만, 저장 비용은 보존 기간에 비례해서 누적된다. 그리고 Logs Insights로 분석할 때 스캔 비용까지 추가된다. 비용 최적화는 이 세 가지를 모두 고려해야 한다.

## 보존 기간 설정: 기본값은 "무기한"

CloudWatch Logs의 기본 보존 기간은 "Never expire(무기한)"이다. 이 설정을 바꾸지 않으면 서비스 시작 이후의 모든 로그가 영원히 저장되고, 저장 비용이 매월 쌓인다.

대부분의 운영 로그는 7~30일이면 충분하다. 장애 분석이나 디버깅 목적이라면 14일, 감사 목적이면 90일 정도가 일반적이다. Log Group별로 보존 기간을 설정할 수 있으므로, 용도에 맞게 차등 적용하자.

| 로그 유형 | 권장 보존 기간 | 이유 |
|----------|-------------|------|
| 애플리케이션 디버그 로그 | 7~14일 | 장애 대응용, 단기 |
| Lambda 실행 로그 | 14~30일 | 실행 추적, 에러 분석 |
| 감사/보안 로그 | 90~365일 | 컴플라이언스 요구 |
| ECS/EKS 컨테이너 로그 | 7~14일 | 대량 발생, 단기 참조 |

기존 Log Group에 보존 기간을 소급 적용하면, 설정 기간을 초과한 과거 로그가 자동 삭제되면서 저장 비용이 즉시 줄어든다.

## Subscription Filter로 S3 전송

장기 보관이 필요하지만 비용을 줄이고 싶다면, CloudWatch Logs에서 S3로 내보내는 방법이 있다. Subscription Filter를 통해 실시간으로 로그를 Data Firehose → S3로 스트리밍하거나, Export Task로 배치 전송할 수 있다.

S3에 저장하면 CloudWatch의 $0.0314/GB-월 대신 S3 Standard $0.025/GB-월, Standard-IA $0.0138/GB-월, Glacier $0.005/GB-월 등 훨씬 저렴한 저장 요금이 적용된다. 분석이 필요할 때는 Athena로 쿼리하면 된다.

단, CloudWatch Logs의 수집 비용($0.76/GB)은 한 번 발생한 뒤이므로, S3 전송은 저장 비용 절감 전략이다. 아예 수집 비용까지 줄이려면 로그를 CloudWatch가 아닌 S3로 직접 보내는 구성을 검토해야 한다.

## Log Class 선택: Standard vs Infrequent Access

2023년부터 CloudWatch Logs에 Log Class 개념이 추가되었다. Infrequent Access(IA) 클래스를 선택하면 수집 비용이 50% 할인된다.

| Log Class | 수집 비용 | 저장 비용 | 제한사항 |
|-----------|----------|----------|---------|
| Standard | $0.76/GB | $0.0314/GB-월 | 제한 없음 |
| Infrequent Access | $0.38/GB | $0.0314/GB-월 | Logs Insights 사용 불가, Metric Filter 불가 |

IA 클래스는 실시간 분석(Logs Insights)과 Metric Filter를 사용할 수 없다. 하지만 "저장만 하고 문제 발생 시 Live Tail로 확인"하는 용도라면 IA가 적합하다. Lambda 함수의 대량 디버그 로그나 배치 작업 로그처럼 평소에는 보지 않는 로그에 효과적이다.

## Logs Insights 비용 주의

CloudWatch Logs Insights는 편리한 쿼리 도구지만, 스캔한 데이터 양에 비례해서 과금된다($0.0076/GB). 보존 기간이 긴 Log Group에 넓은 시간 범위로 쿼리하면 예상보다 비용이 클 수 있다.

비용을 줄이려면:
- 쿼리 시간 범위를 최소화한다 (1시간 vs 7일)
- `filter` 명령으로 초기에 데이터를 걸러낸다
- 자주 쓰는 쿼리는 결과를 캐싱하거나 대시보드에 저장한다

## 마무리

CloudWatch Logs 비용 최적화는 보존 기간 설정, Log Class 선택, 불필요한 로그 제거의 세 축으로 이루어진다. 기본값인 "무기한 보존"만 적정 기간으로 변경해도 상당한 절감이 가능하다. 장기 보관이 필요하면 S3로 내보내고, 실시간 분석이 필요 없는 로그는 IA 클래스를 적용하자.
