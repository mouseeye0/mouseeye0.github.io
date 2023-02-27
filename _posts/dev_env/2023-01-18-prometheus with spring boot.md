---
title: prometheus with spring boot
author: mouseeye
date: 2023-01-18 15:00:00 +0900
categories: [prometheus]
tags: [prometheus]
---

## monitoring system의 특징
### Dimensionality
- 다양한 수집 정보들을 어떻게 key value로 매핑 할지에 대한것
### Rate Aggregation
- 일정 시간동안 수집된 데이터의 집계
### Publishing
- 측정된 매트릭을 수집하기 위해서 저장소에 일정한 시간 간격으로 push할지 poll할지에 대한것

## Registry
- mirometer는 수집된 metric들을 MeterRegistry에 저장한다.
- MeterRegistry는 monitoring system마다 구현체가 있다.
- 로컬 메모리에 저장하고, 따로 사용할 monitoring system이 없다면, SimpleMeterRegistry를 사용할 수 있다.
- SimpleMeterRegistry : 로컬 메모리에 저장하고, 따로 사용할 monitoring system이 없다면, SimpleMeterRegistry를 사용할 수 있다.
- CompositeMeterRegistry : 여러개의 Monitoring system를 등록 할 수 있는 MeterRegistry
- GlobalRegistry : 전역으로 접근 가능한 registry, composite registry로 여러개 등록 가능

## Meters
- metric을 수집하는 방식을 나타내는 타입, 여러가지 meter 타입이 있음
- Timer, Counter, Gauge, DistributionSummary, LongTaskTimer, FunctionCounter, FunctionTimer, and TimeGauge
- single metric을 수집하는 타입 : Gauge
- multi metric을 수집하는 타입 : Timer
- meter는 name과 tag들로 구분 가능
- naming convention : 모니터링 시스템마다 다르지만, 소문자 + .으로 구분 / 특수문자 사용 불가
  - registry.timer("http.server.requests");
    - prometheus : http_server_requests_duration_seconds
    - Atlas : httpServerRequests
- common tags : regitry 전체에 적용될 공통 태그
  - registry.config().commonTags();

### 참고자료
- https://acafela.github.io/monitoring/2021/11/28/prometheus-grafana-springboot-1.html
  - https://prometheus.io/docs/concepts/metric_types/
- https://www.baeldung.com/micrometer
-

