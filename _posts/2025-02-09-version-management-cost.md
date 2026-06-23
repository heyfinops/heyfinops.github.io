---
title: "버전 관리도 비용최적화에 중요하다"
date: 2025-02-09 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [versioning, s3, ecr, cost-optimization, finops]
---

## 삭제했는데 비용이 줄지 않는 미스터리

S3 버킷에서 파일을 삭제했는데 저장 비용이 그대로라면? 높은 확률로 Versioning이 켜져 있다. ECR에서 오래된 이미지를 안 쓰는데 저장 비용이 계속 올라간다면? 이미지 버전이 누적되고 있는 것이다. Lambda 함수를 업데이트했는데 이전 코드 패키지가 계속 과금된다면? 배포된 버전(Version)과 별칭(Alias)이 남아있기 때문이다. 클라우드에서 "버전 관리"는 데이터 보호에 필수적이지만, 방치하면 비용이 끊임없이 쌓인다.

## S3 Versioning: 삭제해도 남는 이전 버전

S3에서 Versioning을 활성화하면, 객체를 덮어쓰거나 삭제해도 이전 버전이 사라지지 않는다. "삭제"를 하면 Delete Marker만 추가될 뿐, 실제 데이터는 Noncurrent Version으로 남아서 계속 저장 비용이 발생한다.

10MB 파일을 매일 업데이트하면:
- 30일 후: 30개 버전 × 10MB = 300MB
- 1년 후: 365개 버전 × 10MB = 3.65GB

이런 파일이 1,000개면? 1년에 3.65TB의 Noncurrent 버전이 쌓인다. Standard 기준 월 $91.25의 저장 비용이 "안 쓰는 데이터"에 나가는 셈이다.

**Lifecycle으로 해결하기:**

```json
{
  "Rules": [{
    "ID": "cleanup-old-versions",
    "Status": "Enabled",
    "NoncurrentVersionTransitions": [
      {"NoncurrentDays": 30, "StorageClass": "STANDARD_IA"},
      {"NoncurrentDays": 90, "StorageClass": "GLACIER"}
    ],
    "NoncurrentVersionExpiration": {"NoncurrentDays": 180}
  }]
}
```

이 규칙은 30일 지난 이전 버전을 IA로, 90일이면 Glacier로, 180일이 되면 완전 삭제한다. Delete Marker도 `ExpiredObjectDeleteMarker: true`로 자동 정리할 수 있다.

## ECR 이미지 버전 누적

ECR(Elastic Container Registry)에 Docker 이미지를 Push할 때마다 새로운 이미지 레이어가 저장된다. CI/CD 파이프라인이 매 커밋마다 이미지를 빌드하면, 하루에도 수십 개의 이미지 버전이 쌓일 수 있다.

ECR 저장 비용: $0.10/GB-월 (서울)

500MB 이미지를 하루 5번 빌드하면:
- 일: 2.5GB 추가
- 월: 75GB 추가
- 비용: $7.50/월 누적 (정리 안 하면 계속 증가)

**ECR Lifecycle Policy로 해결하기:**

```json
{
  "rules": [{
    "rulePriority": 1,
    "description": "Keep only 10 latest images",
    "selection": {
      "tagStatus": "any",
      "countType": "imageCountMoreThan",
      "countNumber": 10
    },
    "action": {"type": "expire"}
  }]
}
```

최근 10개 이미지만 유지하고 나머지는 자동 삭제한다. 태그 기반으로 "untagged 이미지는 1일 후 삭제" 같은 규칙도 설정할 수 있다.

## Lambda 버전과 Alias 비용

Lambda 함수를 $LATEST에만 배포하면 하나의 코드 패키지만 저장된다. 하지만 버전을 발행(Publish Version)하면 해당 시점의 코드가 영구 저장된다. Alias를 통해 블루/그린 배포를 하면서 이전 버전을 유지하면, 사용하지 않는 버전의 코드 패키지가 계속 과금된다.

Lambda 저장 비용: $0.0000000309/GB-초 (실행 비용과 별개로 코드 저장 $0.025/GB-월 이후 무료 구간 75GB)

75GB 무료 구간이 있어서 적은 수의 함수라면 문제없지만, 함수 수가 많고 각 함수에 수십 개 버전이 쌓이면 무료 구간을 초과할 수 있다.

**정리 방법:**

```bash
# 2개 최신 버전만 유지하고 나머지 삭제
aws lambda list-versions-by-function --function-name my-function \
  --query 'Versions[?Version!=`$LATEST`].Version' --output text | \
  sort -rn | tail -n +3 | \
  xargs -I {} aws lambda delete-function --function-name my-function --qualifier {}
```

## 버전 관리 비용 요약

| 서비스 | 비용 발생 원인 | 정리 도구 | 권장 보존 |
|--------|-------------|----------|----------|
| S3 Versioning | Noncurrent 버전 누적 | Lifecycle Policy | 30~180일 후 삭제/전환 |
| ECR | 이미지 버전 누적 | ECR Lifecycle Policy | 최근 10~20개만 유지 |
| Lambda | 발행된 버전 코드 저장 | 수동 삭제 / 자동화 스크립트 | 최근 2~3개 유지 |
| EBS Snapshot | 스냅샷 수 증가 | AWS Backup 보존 정책 | 7~30일 |

## 자동화가 핵심이다

버전 정리를 수동으로 하면 금방 잊혀진다. 효과적인 접근은:

1. **S3**: Lifecycle Policy 설정 (한 번 설정하면 자동 동작)
2. **ECR**: Lifecycle Policy 설정 (리포지토리별 적용)
3. **Lambda**: EventBridge + Lambda 조합으로 주기적 정리 자동화
4. **전체**: AWS Config Rule로 "Lifecycle 미설정 버킷" 감지

## 마무리

버전 관리는 데이터 보호와 롤백에 필수적이지만, 정리 정책 없이 방치하면 비용이 선형으로 증가한다. S3 Lifecycle, ECR Lifecycle Policy, Lambda 버전 정리를 각각 설정해서 "보존할 기간"과 "유지할 개수"를 명확히 정의하자. 설정 한 번이 매달 수십 달러를 절약해준다.
