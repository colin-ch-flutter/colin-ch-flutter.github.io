---
layout: post
title: "Flutter 2048 게임 개발기 26 - 테스터 모집을 위해 게임 홈페이지를 만든 이유"
description: "Tester Community 비공개 테스트와 Play 스토어 링크를 연결할 Mergrove 홈페이지 제작기."
date: 2026-08-16
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 게임 화면](https://ploop-games.web.app/assets/mergrove/01_core_merge_phone_1080x1920.png)

이 그림에서는 실제 게임판과 테마의 색·타일 대비를 확인하면 된다.

커뮤니티 글만으로는 게임이 어떤 분위기인지, 개인정보처리방침은 어디에 있는지, 설치 뒤 무엇을 하면 되는지 전달하기 어려웠다. 그래서 PLOOP Apps 사이트에 Mergrove 소개 페이지를 만들고 게임 화면·5개 테마·오프라인 플레이·스토어 버튼을 한곳에 모았다.

## 오늘의 구현 판단

홈페이지는 마케팅용 장식보다 신뢰의 기준점이었다. 테스터에게는 같은 링크로 게임 설명과 설치 경로를 제공하고, 스토어 지원 웹사이트로도 재사용했다.

| 확인 항목 | 오늘의 기준 |
| --- | --- |
| 플레이 흐름 | 한 손 조작으로 규칙을 이해하고 바로 한 판을 시작할 수 있어야 한다 |
| 품질 확인 | 화면 크기·저장·입력·오버레이를 분리해 재현 가능한 상태로 점검한다 |
| 출시 관점 | 기능 하나를 추가할 때도 스토어 설명과 테스트 흐름이 함께 바뀌는지 본다 |

처음엔 게임 화면만 완성하면 출시 준비도 거의 끝날 것이라 생각했다. 해보니 가장 시간이 걸린 부분은 규칙 자체보다, 끊긴 뒤 다시 이어지는 경험과 처음 보는 사람이 망설이지 않는 화면이었다. 이 날의 변경도 그 기준에서만 남겼다.

실제 구현은 게임 규칙을 순수 Dart로 두고 Flutter가 제스처·레이아웃·애니메이션을 맡는 구조를 유지했다. 그래서 보드가 달라져도 병합 결과를 같은 방식으로 검증할 수 있었고, 작은 화면의 문제도 규칙 버그와 섞이지 않았다.

## 구현 메모

홈페이지는 게임의 판매 페이지이면서 테스트 안내의 단일 출처가 됐다. 스토어 주소와 개인정보처리방침, 실제 화면을 여러 커뮤니티 메시지에 반복하지 않고 한 링크로 관리했다.

![Mergrove 보조 게임 화면](https://ploop-games.web.app/assets/mergrove/03_collect_discover_phone_1080x1920.png)

이 화면에서는 보드 밀도, 타일 구분, 상단 정보와 조작 영역의 간격을 함께 확인하면 된다.

## 참고 자료

[Google Play 비공개 테스트 요구사항](https://support.google.com/googleplay/android-developer/answer/14151465?hl=en) · [테스트 트랙 설정 안내](https://support.google.com/googleplay/android-developer/answer/9845334?hl=en)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)

- 핵심: 홈페이지는
- 확인: 기능이 작아 보여도 실제 기기와 처음 플레이 흐름에서 다시 검증한다.
