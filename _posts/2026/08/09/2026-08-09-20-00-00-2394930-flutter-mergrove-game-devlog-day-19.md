---
layout: post
title: "오프라인 게임인데 Firebase를 왜 넣었냐고?"
description: "Flutter 게임에서 Analytics·Crashlytics·Remote Config를 운영 도구로 쓰는 방식."
date: 2026-08-09
tags: [Flutter, Mergrove, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![개발 과정 참고 이미지](https://images.unsplash.com/photo-1637563680361-3e7ee7599318?auto=format&fit=crop&w=1400&q=80)

[이미지 출처: Unsplash · Penfer](https://unsplash.com/photos/a-man-sitting-in-front-of-a-laptop-computer-_kbLtjx-pT0)

계정 없이 오프라인으로 즐기는 게임이 목표였지만, 출시 뒤의 오류와 광고 정책은 관찰할 방법이 필요했다. Firebase Analytics와 Crashlytics는 안정성 확인에, Remote Config는 최소 지원 버전과 광고 정책에만 사용했다.

핵심 보드 상태는 기기에 남기고, 원격 연결 실패가 게임 시작을 막지 않게 안전한 기본값을 두었다. 운영 도구가 게임의 의존성이 되지 않도록 범위를 좁혔다.


Remote Config의 기본값을 앱 안에도 둔 이유는 네트워크 실패 때문이다. 원격 값을 받지 못해도 광고 기본값과 최소 지원 버전 판단이 예측 가능하게 남아야 한다.

Firebase를 붙이면 무조건 온라인 게임처럼 변할까 봐 조심스러웠다. 접속이 안 돼도 게임이 열리는지부터 확인하니 범위를 정하기 쉬웠다.

## 참고 자료

[Firebase for Flutter 설정](https://firebase.google.com/docs/flutter/setup) · [Firebase Remote Config 공식 안내](https://firebase.google.com/docs/remote-config/flutter/get-started)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
