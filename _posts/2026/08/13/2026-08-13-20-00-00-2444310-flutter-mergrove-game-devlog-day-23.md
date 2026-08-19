---
layout: post
title: "AAB 올리면 끝인 줄 알았는데, Play Console 체크 폭탄"
description: "Flutter 게임을 Google Play에 올리기 전 패키지·AAB·데이터 보안을 확인한 기록."
date: 2026-08-13
tags: [Flutter, Dart, Android, 게임개발, PlayStore]
comments: true
share: true
---

![Mergrove 실제 게임·스토어 화면](/images/mergrove/day-23-tablet_10_store_home.png)

앱 이름, 패키지명 `com.ploopgames.mergrove`, 광고 포함 여부, 비소모성 광고 제거 상품, 공개 HTTPS 개인정보처리방침을 하나씩 대조했다. 릴리스 빌드에는 테스트 광고 ID가 남지 않도록 별도 확인했다.

스토어 등록은 파일 업로드 한 번이 아니었다. 데이터 보안 설문, 콘텐츠 등급, 지원 주소, 스크린샷, 배포 트랙이 서로 맞아야 심사 뒤의 수정 비용이 줄어든다.


Play Console 입력값은 코드의 패키지명, 실제 광고·결제 설정, 공개 개인정보처리방침을 같은 릴리스 기준으로 맞췄다. 체크리스트 문서가 있어도 최종 AAB를 기준으로 재확인해야 한다.

콘솔 입력칸은 각각 별개처럼 보이지만, 하나라도 엇나가면 출시가 멈춘다. 체크 표시를 채우는 일이 꽤 현실적인 개발 작업이라는 걸 실감했다.

## 참고 자료

[Flutter 테스트 개요](https://docs.flutter.dev/testing/overview) · [Flutter 통합 테스트 안내](https://docs.flutter.dev/testing/integration-tests)

[Google Play에서 Mergrove 설치하기](https://play.google.com/store/apps/details?id=com.ploopgames.mergrove)
