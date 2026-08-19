---
layout: post
title: "Flutter 2048 게임 개발기 23 - Play Console 등록 전 체크리스트"
description: "Flutter 게임을 Google Play에 올리기 전 패키지·AAB·데이터 보안을 확인한 기록."
date: 2026-08-13
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-23-tablet_10_store_home.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

앱 이름, 패키지명 `com.ploopgames.mergrove`, 광고 포함 여부, 비소모성 광고 제거 상품, 공개 HTTPS 개인정보처리방침을 하나씩 대조했다. 릴리스 빌드에는 테스트 광고 ID가 남지 않도록 별도 확인했다.

## 오늘의 구현 판단

스토어 등록은 파일 업로드 한 번이 아니었다. 데이터 보안 설문, 콘텐츠 등급, 지원 주소, 스크린샷, 배포 트랙이 서로 맞아야 심사 뒤의 수정 비용이 줄어든다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

Play Console 입력값은 코드의 패키지명, 실제 광고·결제 설정, 공개 개인정보처리방침을 같은 릴리스 기준으로 맞췄다. 체크리스트 문서가 있어도 최종 AAB를 기준으로 재확인해야 한다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 스토어
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
