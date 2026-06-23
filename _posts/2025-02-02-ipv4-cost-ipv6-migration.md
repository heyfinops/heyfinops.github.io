---
title: "과금되는 IPv4 주소값, 그리고 IPv6로 전환하기"
date: 2025-02-02 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [ipv4, ipv6, elastic-ip, cost-optimization, finops]
---

## 공짜였던 IP 주소에 값이 매겨졌다

2024년 2월 1일부터 AWS는 모든 Public IPv4 주소에 시간당 $0.005를 부과하기 시작했다. 이전에는 실행 중인 EC2에 연결된 Elastic IP만 무료였고, 미사용 EIP만 과금됐다. 이제는 사용 여부와 관계없이 Public IPv4 주소를 보유하는 것 자체에 비용이 붙는다. 주소 하나당 월 $3.65, 10개면 $36.5, 100개면 $365다. IP 주소를 넉넉히 할당해두는 습관이 있었다면 비용 청구서에서 새로운 항목을 발견하게 될 것이다.

## 영향 범위: 생각보다 넓다

Public IPv4 과금은 EC2의 Public IP만 해당되는 게 아니다. AWS에서 Public IPv4를 할당받는 모든 서비스가 대상이다.

| 서비스 | IPv4 할당 지점 | 비고 |
|--------|--------------|------|
| EC2 | Public IPv4, Elastic IP | 인스턴스당 1개 이상 |
| RDS | Public 접근 활성화 시 | Multi-AZ면 2개 |
| ELB (ALB/NLB) | AZ당 1개 | 3 AZ면 3개 |
| NAT Gateway | AZ당 1개 | 탄력적 IP 포함 |
| ECS (awsvpc + Public) | 태스크당 1개 | Fargate 포함 |
| Global Accelerator | 고정 IP 2개 | Anycast IP |

예를 들어 ALB 1개(3 AZ = 3 IP) + NAT Gateway 2개(2 IP) + RDS 1개(Public 1 IP) + EC2 5개(5 IP) = 11개 IP라면, 월 $40.15가 순수히 IP 주소 유지 비용으로 나간다. 규모가 큰 환경에서는 수백 개의 IPv4 주소가 쓰이고 있을 수 있다.

## IPv6 전환으로 비용 제거

IPv6를 사용하면 Public IPv4 주소가 필요 없어지므로 이 비용 항목 자체를 제거할 수 있다. AWS VPC는 IPv6를 기본 지원하며, 서브넷에 IPv6 CIDR을 할당하면 인스턴스에 IPv6 주소가 부여된다. IPv6 주소는 과금 대상이 아니다.

다만 완전한 IPv6 전환은 간단하지 않다. 인터넷의 상당 부분이 아직 IPv4를 사용하고 있고, 클라이언트(모바일 앱, 레거시 시스템)가 IPv6를 지원하지 않을 수 있다. 내부 통신은 IPv6로 전환하고, 외부 접점만 IPv4를 유지하는 하이브리드 접근이 현실적이다.

## Dual-Stack: 점진적 전환 전략

Dual-Stack은 하나의 리소스에 IPv4와 IPv6 주소를 모두 할당하는 방식이다. 완전한 IPv6 전환이 어려울 때 과도기적으로 사용한다.

**1단계: VPC IPv6 활성화**

VPC에 IPv6 CIDR을 할당하고, 서브넷에도 IPv6 CIDR을 추가한다. 라우팅 테이블에 `::/0 → Egress-Only Internet Gateway` 경로를 추가하면 IPv6 아웃바운드가 가능해진다.

**2단계: 내부 통신 IPv6 전환**

프라이빗 서브넷의 EC2, ECS 태스크 간 통신을 IPv6로 전환한다. 이 단계에서 NAT Gateway를 제거할 수 있다면 IPv4 비용 + NAT Gateway 비용($0.059/h + 데이터 처리)까지 한꺼번에 절감된다.

**3단계: 외부 접점 Dual-Stack 적용**

ALB를 Dual-Stack으로 설정하면 IPv4와 IPv6 클라이언트 모두 접속할 수 있다. CloudFront도 기본적으로 IPv6를 지원하므로, CF → Origin 통신을 IPv6로 하면 Origin에서 Public IPv4가 필요 없어진다.

## 전환 시 주의사항

IPv6 전환에는 몇 가지 함정이 있다. Security Group은 IPv4 규칙과 IPv6 규칙을 별도로 관리해야 하며, IPv4 전용으로 설정한 규칙이 있으면 IPv6 트래픽이 의도치 않게 허용/차단될 수 있다. DNS도 A 레코드(IPv4)와 AAAA 레코드(IPv6)를 모두 관리해야 한다.

VPC Peering은 IPv6를 지원하지만, 피어링 양쪽의 IPv6 CIDR이 겹치지 않도록 설계해야 한다. 서드파티 서비스(결제 게이트웨이, 외부 API 등)가 IPv6를 지원하는지도 사전에 확인이 필요하다.

| 항목 | 확인 사항 |
|------|----------|
| Security Group | IPv6 규칙 별도 추가 필요 |
| NACL | IPv6 규칙 별도 추가 필요 |
| DNS | AAAA 레코드 추가 |
| 서드파티 연동 | IPv6 지원 여부 확인 |
| 모니터링 | IPv6 트래픽 가시성 확보 |

## 마무리

Public IPv4 주소당 월 $3.65는 작아 보이지만, 수십~수백 개를 사용하면 무시할 수 없는 금액이 된다. 내부 통신부터 IPv6로 전환하고, 외부 접점은 Dual-Stack으로 운영하면 IPv4 의존도를 점진적으로 줄여나갈 수 있다. 당장 전체 전환이 어렵더라도, 불필요한 Public IPv4 할당을 정리하는 것부터 시작해보자.
