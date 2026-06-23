---
title: "포함된 비용, 분리된 비용 (라이센스, 스토리지 등)"
date: 2025-01-18 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [licensing, bundled-cost, pricing, finops]
---

## 인스턴스 요금에 뭐가 포함이고 뭐가 별도인가

EC2 인스턴스 하나를 띄웠을 뿐인데 청구서를 자세히 보면 항목이 여러 개다. 인스턴스 시간당 비용 외에 EBS 스토리지, 데이터 전송, 스냅샷 등이 따로 잡혀 있다. 어떤 비용은 인스턴스 요금에 "포함(Bundled)"되어 있고, 어떤 비용은 "분리(Separate)"되어 별도 과금된다. 이 경계를 모르면 예상 비용과 실제 청구서 사이에 괴리가 생긴다. 특히 OS 라이센스 비용이 대표적인 사례다. Linux는 인스턴스 요금에 포함이지만, Windows나 RHEL은 라이센스 비용이 인스턴스 단가에 추가로 번들되어 있다.

## OS 라이센스: Linux vs Windows vs RHEL

EC2에서 Linux를 선택하면 OS 라이센스 비용이 없다. 오픈소스 OS이므로 인스턴스 컴퓨팅 비용만 나간다. 반면 Windows 인스턴스를 실행하면 OS 라이센스 비용이 인스턴스 시간당 요금에 포함되어 단가가 올라간다. 예를 들어 m5.xlarge(us-east-1, 2025년 기준) Linux가 시간당 $0.192인데, Windows는 $0.376이다. 차이인 $0.184가 Windows 라이센스 비용이다. RHEL(Red Hat Enterprise Linux)도 마찬가지로 라이센스가 번들되어 Linux보다 단가가 높다.

| OS | m5.xlarge 시간당 (us-east-1) | 라이센스 비용 포함 여부 |
|----|------------------------------|----------------------|
| Amazon Linux / Ubuntu | $0.192 | 해당 없음 (무료 OS) |
| Windows | $0.376 | 포함 (+$0.184) |
| RHEL | $0.282 | 포함 (+$0.090) |
| SUSE | $0.262 | 포함 (+$0.070) |
| Windows + SQL Standard | $1.046 | 포함 (OS+DB 라이센스) |

## RDS 라이센스: License Included vs BYOL

RDS에서도 라이센스 이슈가 있다. MySQL, PostgreSQL, MariaDB는 오픈소스 엔진이므로 라이센스 비용이 없다. Oracle과 SQL Server는 **License Included** 모델에서 라이센스가 인스턴스 비용에 번들된다. Oracle SE2(Standard Edition 2) License Included가 db.m5.xlarge 기준 시간당 $0.682인 반면, 같은 크기의 PostgreSQL은 $0.348이다. 라이센스만으로 2배 가까이 차이난다.

대안은 **BYOL(Bring Your Own License)**이다. 기업이 이미 보유한 Oracle/SQL Server 라이센스를 AWS에 가져와 쓰는 방식으로, 인스턴스 비용에서 라이센스분이 빠진다. 다만 BYOL은 Oracle의 경우 RDS Custom이나 EC2에 직접 설치해야 하고, 라이센스 조건(프로세서 코어 매핑 등)을 충족해야 해서 사전 검토가 필요하다.

## 마켓플레이스 소프트웨어 비용 구조

AWS Marketplace에서 AMI 형태로 제공되는 서드파티 소프트웨어도 비용 구조를 잘 봐야 한다. 구조는 보통 이렇다.

- **인프라 비용 (EC2)**: AWS에 지불
- **소프트웨어 비용**: Marketplace 판매자에게 지불

청구서에서는 하나로 합쳐져 나오지만, 실제로는 두 요소가 분리되어 있다. 소프트웨어 비용은 인스턴스 타입에 따라 차등인 경우가 많다(vCPU 수 기준 과금). 예를 들어 특정 보안 솔루션이 vCPU당 $0.05/h라면, c5.4xlarge(16 vCPU)에서는 소프트웨어 비용만 시간당 $0.80이 추가된다. 인스턴스를 Right-sizing 할 때 소프트웨어 비용 감소 효과도 함께 고려하면 절감이 더 커질 수 있다.

## 라이센스 최적화 전략

라이센스 비용을 줄이는 실무 전략은 몇 가지가 있다.

| 전략 | 방법 | 효과 |
|------|------|------|
| OS 전환 | Windows → Linux 마이그레이션 | 인스턴스당 40~50% 절감 |
| DB 엔진 전환 | Oracle → PostgreSQL 마이그레이션 | 라이센스 비용 제거 |
| BYOL 활용 | 기존 라이센스를 클라우드에 적용 | License Included 대비 절감 |
| License Manager | AWS License Manager로 추적 | 초과 사용 방지, 최적 배치 |
| 인스턴스 최적화 | vCPU 기반 과금 소프트웨어 → 작은 인스턴스 | 라이센스 비용 비례 감소 |

특히 Windows에서 Linux로의 전환은 FinOps 관점에서 가장 ROI가 높은 마이그레이션 중 하나다. 워크로드가 .NET Core나 컨테이너 기반이라면 Linux 전환이 비교적 수월하고, 인스턴스당 40~50%의 비용 절감이 가능하다.

## 마무리

AWS 서비스의 비용은 "보이는 단가"가 전부가 아니다. OS 라이센스가 번들인지, DB 라이센스가 포함인지, 마켓플레이스 소프트웨어 비용이 별도인지를 파악해야 정확한 비용 예측이 가능하다. 라이센스가 비용의 큰 비중을 차지하는 워크로드라면, OS/엔진 전환이나 BYOL을 적극적으로 검토할 가치가 있다.
