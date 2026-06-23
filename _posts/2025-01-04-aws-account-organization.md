---
title: "AWS Account와 AWS Organization의 수상한 관계"
date: 2025-01-04 10:00:00 +0900
categories: [AWS, 입문]
tags: [aws-organization, account, billing, finops]
---

## 계정 하나면 충분하지 않나요?

AWS를 처음 쓸 때는 계정 하나로 시작한다. 개인 프로젝트든 소규모 팀이든, 하나의 AWS Account에 모든 자원을 넣어서 운영한다. 그런데 조직이 커지면 계정이 2개, 10개, 100개로 늘어난다. 왜일까? 보안 분리, 환경 분리(dev/staging/prod), 팀별 독립성, 비용 추적 편의성 등 여러 이유가 있다. 그리고 이 여러 계정을 하나의 우산 아래 묶는 게 AWS Organizations다.

AWS Organizations는 복수의 계정을 계층 구조(OU: Organizational Unit)로 묶어주는 서비스다. 단순한 계정 관리 도구가 아니라, 비용 관점에서 매우 중요한 역할을 한다. 통합 결제(Consolidated Billing)를 통해 모든 계정의 비용이 하나의 청구서로 합쳐지고, 이 합산 사용량 기반으로 볼륨 할인을 받을 수 있기 때문이다.

## 통합 결제의 비용 효과

Organizations에 묶인 계정들은 개별적으로 보면 각자 100GB의 S3를 쓰고 있지만, AWS 입장에서는 합산 500GB로 계산해준다. S3의 계단식 요금 구조에서 합산 볼륨이 커질수록 단가가 떨어지니, 자연스럽게 할인 효과가 생긴다. 마찬가지로 약정(Reserved Instances, Savings Plans)도 Organization 내에서 공유가 가능하다. A 계정에서 산 RI가 B 계정의 인스턴스에도 적용될 수 있다.

다만 이 공유 구조는 양날의 검이다. 약정이 의도치 않은 계정에 적용되면서 정작 원래 대상 계정에는 할인이 안 되는 경우도 발생한다. 이걸 제어하려면 RI/SP 공유 설정을 잘 관리해야 하고, 이게 FinOps에서 꽤 중요한 거버넌스 영역이다.

## 계정 구조가 비용 추적에 미치는 영향

계정을 어떻게 나누느냐는 비용 가시성에 직접적인 영향을 준다. 팀별로 계정을 나누면, 팀별 비용이 계정 단위로 깔끔하게 분리된다. 서비스별로 나누면, 서비스 단위 비용 추적이 쉽다. 하지만 하나의 계정에 여러 팀, 여러 서비스가 섞여 있으면 태그(Tag)에 의존해야 하고, 태깅 누락 시 비용 귀속이 불가능해진다.

잘 설계된 계정 구조는 그 자체로 비용 관리 도구 역할을 한다. 반대로 계정 구조가 엉망이면, 아무리 좋은 FinOps 도구를 써도 "이 비용이 누구 건지" 파악하기 어렵다. 비용 관리의 첫 단추는 인프라 설계가 아니라 계정 설계에서 시작된다고 해도 과언이 아니다.

## Organization 구조 설계 예시

| OU | 포함 계정 | 목적 |
|----|-----------|------|
| Root | Management Account | 통합 결제, 조직 관리 |
| Production | prod-service-a, prod-service-b | 운영 환경 분리 |
| Development | dev-team-a, dev-team-b | 개발 환경 (비용 상한 설정) |
| Shared | logging, security, network | 공용 인프라 |
| Sandbox | sandbox-experiments | 실험용 (자동 삭제 정책) |

## 마무리

AWS Account는 비용의 최소 단위이자, 보안/권한의 경계선이다. Organizations는 이 계정들을 묶어서 볼륨 할인과 약정 공유를 가능하게 한다. 계정 구조를 잘 설계하면 비용 추적이 자동으로 깔끔해지고, 잘못 설계하면 어떤 도구를 써도 비용 귀속이 어려워진다.
