---
title: "복수 계정에서 공용으로 쓸만한 자원들"
date: 2025-02-19 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [multi-account, shared-resources, organization, finops]
---

## 계정은 나눴는데, 자원까지 다 따로?

AWS Organization을 도입해서 팀별, 환경별로 계정을 분리하는 건 보안과 거버넌스 측면에서 좋은 전략이다. 그런데 계정을 나누고 보니, 비슷한 자원이 각 계정에 중복으로 만들어지는 상황이 생긴다. NAT Gateway, Directory Service, Transit Gateway 같은 것들이 계정마다 하나씩 만들어지면 비용이 계정 수에 비례해서 늘어난다. 공유할 수 있는 건 공유하는 게 합리적이다.

## 공유 대상이 되는 대표 자원들

모든 자원을 공유하면 보안 경계가 무너지니, "공유해도 괜찮은 것"과 "공유하면 안 되는 것"을 구분해야 한다. 대체로 인프라 기반 자원(네트워크, 인증, 이미지)은 공유 대상이고, 애플리케이션 데이터가 있는 자원(DB, S3 버킷)은 신중해야 한다.

| 공유 자원 | 용도 | 비용 이점 | 공유 방법 |
|-----------|------|-----------|-----------|
| Transit Gateway | 계정 간 네트워크 연결 | 계정별 VPN/Peering 불필요 | RAM 공유 |
| Directory Service | AD 인증 중앙화 | 계정별 AD 중복 제거 | RAM 공유 |
| 공용 AMI | 표준화된 서버 이미지 | 이미지 빌드/관리 1회 | AMI 공유 설정 |
| S3 공용 버킷 | 로그 집중, 공유 데이터 | 중복 저장 방지 | 버킷 정책 |
| VPC Endpoint | 프라이빗 서비스 접근 | 계정별 Endpoint 비용 절감 | RAM 공유 |
| Route53 Hosted Zone | DNS 관리 통합 | 중복 Zone 제거 | 크로스 계정 연동 |

## AWS RAM(Resource Access Manager) 활용

AWS RAM은 Organization 내에서 리소스를 안전하게 공유하기 위한 서비스다. Transit Gateway, 서브넷, License Manager 구성, Route53 Resolver 규칙 등을 다른 계정과 공유할 수 있다. RAM으로 공유하면 해당 리소스가 공유 대상 계정의 콘솔에 자연스럽게 나타나므로 사용성도 좋다. 다만, 공유된 리소스에 대한 비용은 소유 계정에 청구되므로 Chargeback 체계와 맞춰야 한다.

## 거버넌스 주의점

공유의 이점이 크지만 주의할 점도 있다. 첫째, 공유 자원에 장애가 나면 모든 의존 계정에 영향이 간다. 단일 장애점(Single Point of Failure)이 되지 않도록 고가용성 구성이 필수다. 둘째, 공유 자원의 비용을 누가 부담할지(소유 계정 vs 사용 비례 배분) 미리 합의해야 한다. 셋째, 공유 범위를 Organization 전체로 열 것인지, 특정 OU(Organizational Unit)로 제한할 것인지 결정해야 한다. RAM에서 "Organization 내 자동 공유"를 켜면 새 계정이 추가될 때마다 자동으로 공유되므로 편리하지만, 범위가 넓어지는 만큼 관리가 필요하다.

## 마무리

멀티 계정 환경에서 네트워크, 인증, 이미지 같은 인프라 기반 자원은 공유하면 비용을 크게 줄일 수 있다. RAM을 활용하면 보안 경계를 유지하면서 자원을 공유할 수 있다. 단, 고가용성 설계와 비용 배분 합의를 빠뜨리지 않도록 하자.
