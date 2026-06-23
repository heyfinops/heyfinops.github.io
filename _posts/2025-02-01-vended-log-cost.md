---
title: "Vended Log 수집비용 최적화 공략"
date: 2025-02-01 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [cloudwatch, logs, vended-log, cost-optimization, finops]
---

## Vended Log가 뭔가요

AWS에서 직접 생성하는 로그를 Vended Log라고 부른다. 내가 애플리케이션에서 찍는 로그가 아니라, AWS 서비스가 알아서 만들어내는 로그다. 대표적으로 VPC Flow Logs, CloudTrail 데이터 이벤트, Route 53 Resolver Query Logs, AWS WAF Logs 등이 여기에 해당한다. 이 로그들은 보안, 감사, 네트워크 트러블슈팅에 필수적이지만, 한번 켜놓으면 생각보다 엄청난 양이 쏟아진다. VPC Flow Logs 하나만 켜도 하루에 수십 GB가 생성되는 환경이 흔하다.

## 수집 비용 구조

Vended Log의 비용은 "어디로 보내느냐"에 따라 크게 달라진다. AWS는 세 가지 목적지를 제공한다: CloudWatch Logs, S3, Data Firehose.

| 목적지 | 수집 비용 (서울, GB당) | 저장 비용 | 실시간 분석 |
|--------|---------------------|----------|------------|
| CloudWatch Logs | $0.76/GB (Vended Log 할인: $0.114/GB) | $0.0314/GB-월 | CloudWatch Logs Insights |
| S3 직접 전송 | $0.076/GB | S3 요금 적용 (~$0.025/GB-월) | Athena 별도 사용 |
| Data Firehose | $0.076/GB + Firehose 비용 | 목적지에 따라 상이 | 중간 변환 가능 |

주목할 점은 CloudWatch Logs의 Vended Log 할인 단가($0.114/GB)와 S3 직접 전송($0.076/GB) 사이에 약 33%의 차이가 있다는 것이다. 월 1TB의 Vended Log가 발생하면, CloudWatch Logs로 보내면 수집만 $114, S3로 보내면 $76이다. 저장 비용까지 합치면 격차가 더 벌어진다.

## S3 직접 전송 vs CloudWatch Logs

대부분의 Vended Log는 실시간 분석보다는 사후 조사(보안 감사, 인시던트 분석) 목적이 강하다. 이런 경우 CloudWatch Logs에 보낼 이유가 없다. S3로 직접 전송하면 수집 비용이 저렴하고, 저장 비용도 S3 Standard-IA나 Glacier로 Lifecycle 전환하면 훨씬 싸진다.

CloudWatch Logs가 필요한 경우는 실시간 알림(Metric Filter), 실시간 대시보드, 즉시 검색이 필요할 때다. 예를 들어 "특정 IP에서의 비정상 접근을 실시간으로 감지하고 알림을 보내야 한다"면 CloudWatch Logs가 적합하다.

| 사용 목적 | 추천 목적지 | 이유 |
|----------|-----------|------|
| 보안 감사·컴플라이언스 보관 | S3 | 저렴, 장기 보관 용이 |
| 실시간 이상 탐지·알림 | CloudWatch Logs | Metric Filter 활용 |
| 네트워크 분석 (주기적) | S3 + Athena | 비용 효율적 쿼리 |
| 멀티 계정 중앙 집중 | S3 (중앙 계정) | 구조 단순, 비용 저렴 |

## 필터링으로 불필요한 로그 줄이기

VPC Flow Logs는 모든 네트워크 패킷의 메타데이터를 기록한다. 하지만 실제로 분석에 필요한 건 REJECT된 트래픽이나 특정 포트의 통신인 경우가 많다. Flow Logs 생성 시 필터를 REJECT로 설정하면 허용된(ACCEPT) 트래픽 로그가 제외되어 로그량이 대폭 줄어든다.

CloudTrail의 경우, 데이터 이벤트(S3 GetObject, Lambda Invoke 등)가 관리 이벤트보다 수백 배 많다. 모든 S3 버킷의 GetObject를 기록하면 로그량이 폭발하므로, 감사가 필요한 버킷만 선별해서 데이터 이벤트를 활성화하는 것이 좋다.

## 보존 기간 최적화

Vended Log를 S3에 저장하더라도, 무기한 보관하면 저장 비용이 계속 쌓인다. 규정 요구사항에 따라 보존 기간을 정하고, Lifecycle Policy로 자동 삭제하거나 Glacier Deep Archive로 전환해야 한다.

일반적인 기준으로 VPC Flow Logs는 90일, CloudTrail은 1년, WAF Logs는 30~90일이 적정하다. 물론 산업별 규정(금융: 5년, 의료: 7년 등)에 따라 조정이 필요하다. 중요한 것은 "보존 기간 없이 무기한"이 기본값이 되면 안 된다는 것이다.

## 마무리

Vended Log는 켜는 순간 비용이 시작된다. S3 직접 전송으로 수집 단가를 낮추고, 필터링으로 불필요한 로그량을 줄이고, Lifecycle으로 보존 기간을 관리하는 세 단계가 Vended Log 비용 최적화의 기본 공식이다. 목적에 맞는 목적지를 선택하는 것부터 시작해보자.
