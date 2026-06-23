---
title: "Gateway 유형의 VPC Endpoint를 적극적으로 활용하는 법"
date: 2025-02-06 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [vpc-endpoint, gateway, s3, dynamodb, finops]
---

## NAT Gateway를 거치는 S3 트래픽, 돈이 새고 있다

프라이빗 서브넷에서 S3나 DynamoDB에 접근할 때, NAT Gateway를 경유하는 구성을 흔히 본다. 동작에는 문제가 없지만, NAT Gateway는 시간당 $0.059 + 데이터 처리 $0.059/GB가 부과된다. 하루에 100GB를 S3로 전송하는 워크로드라면 NAT Gateway 데이터 처리 비용만 월 $177이다. 여기에 시간당 비용 $43/월까지 더하면 $220이 NAT을 거치는 대가로 나간다. Gateway 유형 VPC Endpoint를 만들면 이 비용을 0으로 만들 수 있다.

## Gateway Endpoint vs Interface Endpoint

VPC Endpoint는 두 종류가 있다. 이름이 비슷해서 혼동하기 쉽지만, 비용 구조가 완전히 다르다.

| 구분 | Gateway Endpoint | Interface Endpoint (PrivateLink) |
|------|-----------------|-------------------------------|
| 지원 서비스 | S3, DynamoDB | 나머지 대부분의 AWS 서비스 |
| 비용 | **무료** | $0.013/h (AZ당) + $0.01/GB |
| 연결 방식 | 라우팅 테이블에 경로 추가 | ENI (Elastic Network Interface) 생성 |
| 보안 제어 | Endpoint Policy | Endpoint Policy + Security Group |
| 가용 영역 | 리전 레벨 | AZ별 ENI 생성 필요 |

Gateway Endpoint는 S3와 DynamoDB에만 사용할 수 있지만, 가장 큰 장점은 **완전 무료**라는 것이다. 설정만 하면 데이터 전송량에 관계없이 비용이 0이다.

## NAT Gateway 대비 비용 절감 효과

프라이빗 서브넷에서 S3로 일 100GB를 전송하는 시나리오로 비교해보자.

| 경로 | 시간당 비용 | 데이터 비용 (월 3TB) | 월 합계 |
|------|-----------|-------------------|---------|
| NAT Gateway 경유 | $0.059 × 730h = $43.07 | 3,000GB × $0.059 = $177 | **$220.07** |
| Gateway Endpoint 경유 | $0 | $0 | **$0** |
| 절감액 | — | — | **$220.07** |

NAT Gateway를 완전히 제거할 수는 없더라도(다른 인터넷 접속이 필요하므로), S3/DynamoDB 트래픽을 Gateway Endpoint로 우회시키면 NAT Gateway의 데이터 처리량이 줄어들어 비용이 감소한다.

## 설정 방법

Gateway Endpoint 설정은 매우 간단하다. VPC에 Endpoint를 생성하고, 라우팅 테이블에 자동으로 경로가 추가된다.

**1단계: Endpoint 생성**

VPC 콘솔 → Endpoints → Create endpoint에서:
- Service: `com.amazonaws.ap-northeast-2.s3` (Gateway 유형 선택)
- VPC: 대상 VPC 선택
- Route Tables: 프라이빗 서브넷의 라우팅 테이블 선택

**2단계: 라우팅 테이블 확인**

생성 후 라우팅 테이블에 다음과 같은 경로가 자동 추가된다:

```
Destination: pl-78a54011 (S3 prefix list)
Target: vpce-0abc123def456
```

S3로 향하는 트래픽이 NAT Gateway 대신 VPC Endpoint를 통해 나가게 된다. 애플리케이션 코드 변경은 전혀 필요 없다.

**3단계: DynamoDB도 동일하게 설정**

```
Service: com.amazonaws.ap-northeast-2.dynamodb
```

DynamoDB를 사용하는 Lambda, ECS, EC2가 프라이빗 서브넷에 있다면 반드시 Gateway Endpoint를 만들어두자.

## Endpoint Policy로 접근 제어

Gateway Endpoint에 정책(Policy)을 붙여서 특정 버킷이나 테이블에 대한 접근만 허용할 수 있다. 보안과 비용 제어를 동시에 달성하는 방법이다.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-approved-bucket/*"
  }]
}
```

기본 정책은 Full Access이므로, 필요에 따라 제한적으로 설정하면 된다.

## 언제 Gateway Endpoint를 만들어야 하나

정답은 "항상"이다. 무료이고, 설정이 간단하고, 부작용이 없다. VPC를 만들면 S3와 DynamoDB용 Gateway Endpoint를 기본으로 생성하는 것을 표준으로 삼아야 한다. CloudFormation이나 Terraform 템플릿에 포함시켜두면 빠뜨릴 일이 없다.

```hcl
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.ap-northeast-2.s3"
  route_table_ids = [aws_route_table.private.id]
}
```

## 마무리

Gateway Endpoint는 무료이면서 NAT Gateway 비용을 직접적으로 줄여주는 가장 쉬운 최적화 중 하나다. S3와 DynamoDB를 사용하는 프라이빗 서브넷이 있다면, 지금 바로 Gateway Endpoint가 설정되어 있는지 확인해보자. 설정 한 번으로 매월 수십~수백 달러를 절감할 수 있다.
