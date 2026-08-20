---
layout: post
title: "앱 껐다 켰더니 내 게임이 사라졌다… 저장부터 고친 날"
description: "SharedPreferences로 Flutter 게임판·점수·설정을 로컬 저장한 방법."
date: 2026-07-31
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://images.unsplash.com/photo-1753715613831-9e48eac86813?auto=format&fit=crop&w=1400&q=80)

[이미지 출처: Unsplash · Jakub Żerdzicki](https://unsplash.com/es/fotos/una-persona-codifica-mientras-toma-notas-QUtrcUo5-GI)

모바일 퍼즐은 한 판을 길게 이어가는 사람이 많다. 앱이 종료됐을 때 보드가 사라지면 좋은 점수보다 신뢰를 잃는다. 보드 값, 점수, 테마, 보드 크기, 발견 목록을 기기 저장소에 분리해 기록했다.

복원 실패는 새 게임으로 안전하게 떨어지게 했고, 튜토리얼 표시 여부도 같은 기준으로 관리했다. 서버 계정 없이도 기본 플레이가 이어지는 구조를 우선했다.


저장 값의 버전과 파싱 실패를 고려했다. 앱 업데이트 뒤 과거 형식을 읽지 못하면 예외를 노출하는 대신, 안전하게 새 보드를 시작하도록 해 플레이 진입을 막지 않았다.

앱을 강제로 닫았다가 점수가 사라지는 장면은 보기 싫었다. 저장은 눈에 안 보이는 기능이지만, 여기서 신뢰가 한 번에 깨진다는 걸 느꼈다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
