---
title: "S3의 Storage Class 관리, 기본편"
date: 2025-02-03 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [s3, storage-class, lifecycle, cost-optimization, finops]
---

## 모든 데이터를 Standard에 둘 필요는 없다

S3에 데이터를 올리면 기본적으로 Standard 클래스에 저장된다. 가용성 99.99%, 내구성 99.999999999%(11 Nines), 어디서든 즉시 접근 가능. 하지만 이 최고 사양의 스토리지에 "분기 1회 조회하는 보고서"나 "1년 전 백업 파일"까지 보관할 이유가 있을까? S3는 접근 빈도에 따라 6가지 Storage Class를 제공하며, 데이터의 성격에 맞게 배치하면 저장 비용을 최대 95%까지 줄일 수 있다.

## Storage Class 종류와 특성

서울 리전(ap-northeast-2) 기준 단가와 주요 특성을 정리하면 다음과 같다.

| Storage Class | 저장 비용 (GB-월) | 검색 비용 | 최소 보관 기간 | 최소 객체 크기 | 접근 시간 |
|--------------|-----------------|----------|-------------|-------------|----------|
| Standard | $0.025 | 없음 | 없음 | 없음 | 즉시 |
| Standard-IA | $0.0138 | $0.01/GB | 30일 | 128KB | 즉시 |
| One Zone-IA | $0.011 | $0.01/GB | 30일 | 128KB | 즉시 |
| Glacier Instant Retrieval | $0.005 | $0.03/GB | 90일 | 128KB | 즉시 |
| Glacier Flexible Retrieval | $0.0045 | $0.03~$12/GB | 90일 | 없음 | 분~시간 |
| Glacier Deep Archive | $0.002 | $0.02~$12/GB | 180일 | 없음 | 12~48시간 |

핵심 패턴은 명확하다: 저장 비용이 낮을수록 검색 비용이 높고, 접근 시간이 길어진다.

## 각 클래스는 언제 쓰는 게 맞을까

**Standard**: 자주 접근하는 데이터. 웹 서비스의 정적 파일, 활성 로그, 빈번히 읽는 데이터셋 등.

**Standard-IA(Infrequent Access)**: 월 1~2회 정도 접근하지만, 필요할 때 즉시 읽어야 하는 데이터. 백업, 과거 보고서, DR용 복제본 등. Standard 대비 45% 저렴하다.

**One Zone-IA**: Standard-IA와 동일하지만, 단일 AZ에만 저장된다. AZ 장애 시 데이터 유실 위험이 있으므로, 원본이 다른 곳에 있는 복제본이나 재생성 가능한 데이터에 적합하다. Standard-IA 대비 20% 더 저렴하다.

**Glacier Instant Retrieval**: 분기에 1번 정도 접근하는 데이터. 밀리초 단위 접근이 필요하지만 자주 읽지는 않는 경우. Standard 대비 80% 저렴하다.

**Glacier Flexible Retrieval**: 연 1~2회 접근하는 아카이브 데이터. 검색에 1분~12시간 걸리지만, 그래도 괜찮은 감사 로그, 규정 보관 데이터 등.

**Glacier Deep Archive**: 거의 접근하지 않지만 규정상 7~10년 보관해야 하는 데이터. 가장 저렴하지만 검색에 12~48시간 소요된다.

## Lifecycle Policy로 자동 전환

데이터를 수동으로 클래스 변경하는 건 비현실적이다. S3 Lifecycle Policy를 사용하면 객체 생성 후 경과 일수에 따라 자동으로 클래스를 전환할 수 있다.

예를 들어 로그 데이터의 일반적인 Lifecycle은:

```
생성 → 30일: Standard (활발히 분석)
30일 → 90일: Standard-IA (가끔 참조)
90일 → 365일: Glacier Instant Retrieval (드물게 조회)
365일 →: Glacier Deep Archive (규정 보관)
730일: 삭제
```

이렇게 설정하면 1GB 데이터의 2년간 평균 월 저장 비용이 Standard 고정($0.025)에서 약 $0.006 수준으로 떨어진다.

## Lifecycle Policy 설정 예시

```json
{
  "Rules": [{
    "ID": "log-lifecycle",
    "Status": "Enabled",
    "Filter": {"Prefix": "logs/"},
    "Transitions": [
      {"Days": 30, "StorageClass": "STANDARD_IA"},
      {"Days": 90, "StorageClass": "GLACIER_IR"},
      {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
    ],
    "Expiration": {"Days": 730}
  }]
}
```

Prefix나 Tag 기반으로 필터링할 수 있어서, 버킷 안에서도 데이터 유형별로 다른 Lifecycle을 적용할 수 있다.

## 주의할 점

Lifecycle 전환에는 몇 가지 제약이 있다. 첫째, 최소 보관 기간 내에 삭제하거나 다른 클래스로 전환하면 나머지 기간의 비용이 과금된다. Standard-IA에 넣은 객체를 10일 만에 삭제해도 30일치 비용이 부과된다. 둘째, 128KB 미만의 소형 객체는 Standard-IA나 Glacier Instant에 넣어도 128KB 기준으로 과금되므로 손해다. 셋째, Standard에서 Glacier로 직접 전환은 가능하지만, Glacier에서 Standard-IA로의 역전환은 불가능하다.

## 마무리

S3 비용 최적화의 첫걸음은 데이터의 접근 패턴을 파악하고, 그에 맞는 Storage Class를 배치하는 것이다. Lifecycle Policy로 시간 경과에 따라 자동 전환하면 수작업 없이도 비용이 최적화된다. 다음 심화편에서는 Intelligent-Tiering과 전환 비용(Transition Cost)의 함정을 다룬다.
