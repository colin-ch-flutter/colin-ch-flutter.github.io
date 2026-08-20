---
layout: post
title: "홈페이지 첫 화면에 기능 다 넣었다가 망할 뻔"
description: "Flutter 게임 홈페이지에서 스크린샷·스토어 버튼·개인정보 링크를 배치한 이유."
date: 2026-08-17
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://images.unsplash.com/photo-1637563680361-3e7ee7599318?auto=format&fit=crop&w=1400&q=80)

[이미지 출처: Unsplash · Penfer](https://unsplash.com/photos/a-man-sitting-in-front-of-a-laptop-computer-_kbLtjx-pT0)

첫 화면에는 ‘차분한 타일 머지 퍼즐’이라는 한 줄, 실제 게임 이미지, Google Play 버튼을 배치했다. 이어서 4×4·5×5·6×6 보드와 다섯 자연 테마, 컬렉션·저장·되돌리기 기능을 보여줬다.

홈페이지의 이미지는 앱과 다른 약속을 하면 안 된다. 그래서 스토어 캡처와 같은 실제 게임 장면을 사용했고, App Store는 준비 중임을 명확히 표시했다.


페이지의 첫 CTA는 Google Play 한 곳으로만 연결했다. 버튼이 많으면 선택이 늘지만 테스터가 설치 경로를 놓치기 쉬워, 현재 제공 플랫폼과 준비 중인 플랫폼을 명확히 구분했다.

첫 화면에 모든 장점을 넣고 싶었다. 결과적으로 아무것도 안 남는 화면이 됐고, 설치 버튼과 실제 게임 장면만 남기는 쪽을 택했다.

## 참고 자료

[Google Play 비공개 테스트 요구사항](https://support.google.com/googleplay/android-developer/answer/14151465?hl=en) · [테스트 트랙 설정 안내](https://support.google.com/googleplay/android-developer/answer/9845334?hl=en)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
