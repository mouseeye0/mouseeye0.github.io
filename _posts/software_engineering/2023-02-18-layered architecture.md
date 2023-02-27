---
title: Layered Architecture
author: mouseeye
date: 2023-02-18 15:00:00 +0900
categories: [software engineering]
tags: [layered architecture]
---

### Layered Architecture
- 관심사의 분리에 따라 시스템을 ***유사한 책임과 역할을 가진 layer로 분리***
- 관심사에 따라 각 layer가 높은 응집도와 결합도를 가짐
- 특정 계층의 구성요소는 해당 계층에 관련된 기능만 수행 (단일 책임)
- 각 layer는 ***하위 layer에만 의존하도록 수직 계층***을 만듬
- 각 layer간 시스템 결합도를 낮추고, 재사용성 및 유지보수성을 높이는게 목표

"순간 순간의 상황에 맞게 tradeoff를 고려한 최선의 선택을 하는 것이 바람직"
