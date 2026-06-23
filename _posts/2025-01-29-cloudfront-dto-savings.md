---
title: "CloudFront를 사용하여 DTO 비용 줄이기"
date: 2025-01-29 10:00:00 +0900
categories: [AWS, Cost Optimization]
tags: [cloudfront, data-transfer, dto, cost-optimization, finops]
---

## S3에서 직접 내보내면 왜 비싼가

AWS 요금 명세서를 받아보면 Data Transfer Out(DTO) 항목이 예상보다 크게 찍히는 경우가 있다. S3에 올려둔 이미지, 영상, 정적 파일을 인터넷으로 직접 서빙하면 GB당 $0.126(서울 리전)이 부과된다. 하루 100GB만 나가도 월 $378이다. 문제는 동일한 파일을 수백, 수천 명이 반복 요청하는 상황인데, 매번 S3에서 꺼내서 인터넷으로 내보내면 전송량이 요청 수에 비례해서 불어난다. CloudFront를 앞에 두면 이 구조가 극적으로 바뀐다.

## CloudFront가 DTO를 줄이는 3가지 원리

CloudFront는 단순한 CDN이 아니라 FinOps 관점에서 "DTO 비용 절감 도구"다. 세 가지 메커니즘이 동시에 작동한다.

첫째, S3에서 CloudFront로 나가는 전송(Origin Fetch)은 무료다. S3 → 인터넷 직접 전송은 $0.126/GB인데, S3 → CloudFront는 $0이다. 이것만으로도 과금 구조가 완전히 달라진다. 둘째, CloudFront → 인터넷 전송 단가가 S3 직접 전송보다 저렴하다. 서울 기준 $0.120/GB로 약간의 차이지만, 전송량이 클수록 체감된다. 셋째, 캐시 히트(Cache Hit)로 인해 실제 전송량 자체가 줄어든다. 히트율 90%라면 원본에서 꺼내는 양이 1/10로 감소하므로, Origin Fetch 자체가 거의 발생하지 않는다.

## 적용 전후 비용 비교

월 1TB의 정적 콘텐츠를 서빙하는 상황을 가정해보자.

| 항목 | S3 직접 서빙 | CloudFront 경유 (히트율 90%) |
|------|-------------|---------------------------|
| S3 → 인터넷 전송 | 1,024GB × $0.126 = $129.02 | 0 (CF 경유이므로 무료) |
| S3 → CloudFront 전송 | — | 102.4GB × $0 = $0 |
| CloudFront → 인터넷 전송 | — | 1,024GB × $0.120 = $122.88 |
| Origin Fetch 비용 | — | 거의 무시 가능 |
| **월 합계** | **$129.02** | **$122.88** |

히트율 90% 기준으로도 절감이 발생하지만, 실제 절감의 핵심은 캐시 히트로 인해 Origin에서 꺼내는 빈도가 줄어든다는 데 있다. 만약 S3 직접 서빙에서는 동일 파일이 10번 요청되면 10번 전송되지만, CloudFront에서는 1번만 Origin에서 가져오고 나머지 9번은 엣지에서 처리된다.

## OAC 설정으로 직접 접근 차단

CloudFront를 도입했는데 S3 버킷에 직접 접근하는 경로가 남아 있으면 DTO 절감 효과가 반감된다. Origin Access Control(OAC)을 설정해서 S3에 대한 직접 인터넷 접근을 차단하고, 모든 트래픽을 CloudFront 경유로 강제해야 한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
      }
    }
  }]
}
```

이렇게 하면 S3 버킷 정책에서 CloudFront만 허용하고, 나머지는 모두 거부된다.

## Price Class로 추가 절감

CloudFront는 전 세계 엣지 로케이션을 사용하는데, 모든 리전의 단가가 동일하지 않다. 남미, 호주 등은 GB당 $0.170 이상으로 비싸다. 주 사용자가 한국과 일본에 집중된 서비스라면 Price Class 200이나 Price Class 100으로 제한해서 비싼 리전의 엣지 사용을 배제할 수 있다.

| Price Class | 포함 리전 | 비고 |
|-------------|----------|------|
| All | 전체 엣지 | 글로벌 서비스 |
| 200 | 북미, 유럽, 아시아, 중동, 아프리카 | 남미·호주 제외 |
| 100 | 북미, 유럽 | 가장 저렴한 리전만 |

대부분의 한국 서비스는 Price Class 200이면 충분하다. 비싼 리전을 빼는 것만으로도 예상치 못한 DTO 비용 급등을 방지할 수 있다.

## 마무리

CloudFront는 속도만 빠르게 해주는 도구가 아니다. S3 → CF 무료 전송, 저렴한 CF → 인터넷 단가, 캐시 히트로 인한 전송량 감소까지 세 가지 메커니즘으로 DTO 비용을 구조적으로 줄여준다. OAC로 직접 접근을 차단하고, Price Class를 적절히 설정하면 추가 절감까지 챙길 수 있다.
