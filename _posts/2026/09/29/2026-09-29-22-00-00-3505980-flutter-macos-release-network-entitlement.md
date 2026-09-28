---
layout: post
title: "Flutter macOS Release SocketException - debug는 되는데 네트워크가 막힐 때"
description: "Flutter macOS 앱이 debug에서는 API에 연결되지만 release에서 Operation not permitted를 내는 이유와 entitlements 파일, 검증 명령을 정리한다."
date: 2026-09-29
tags: [Flutter, macOS, CI/CD, 배포·운영]
comments: true
share: true
---

![Flutter macOS release 빌드와 App Sandbox 권한 점검](https://images.unsplash.com/photo-1517336714739-489689fd1ca8?auto=format&fit=crop&w=1600&q=80)

그림에서 볼 부분은 macOS 앱의 실행 환경이 개발 중인 Flutter 프로세스와 배포된 sandbox 앱으로 나뉜다는 점이다.

Flutter macOS 앱이 `flutter run`에서는 API를 잘 호출하다가 `flutter build macos --release` 결과에서 아래 오류를 낸다면 코드보다 entitlements 설정을 의심해야 한다.

```text
SocketException: Connection failed
(OS Error: Operation not permitted, errno = 1)
```

macOS App Sandbox는 네트워크 같은 권한을 자동으로 허용하지 않는다. Flutter 공식 문서도 외부로 나가는 요청에는 `com.apple.security.network.client`가 필요하다고 설명한다. 특히 DebugProfile 파일만 고치면 개발 실행은 통과하지만 Release 앱에서는 실패할 수 있다. Mac App Store 배포를 포기하고 sandbox를 끄는 방식은 권장하지 않는다. 앱을 배포할 계획이 없고 로컬 전용 도구인 경우에만 별도 선택지로 남긴다.

## 파일 두 곳을 같은 기준으로 맞춘다

`macos/Runner/DebugProfile.entitlements`와 `macos/Runner/Release.entitlements`에 필요한 권한을 각각 넣는다. 네트워크 요청을 보내는 앱이라면 아래 항목이 핵심이다.

```xml
<key>com.apple.security.app-sandbox</key>
<true/>
<key>com.apple.security.network.client</key>
<true/>
```

파일 선택 기능을 함께 쓴다면 권한의 범위도 구분해야 한다.

| 기능 | entitlement | 선택 기준 |
|---|---|---|
| HTTPS API 호출 | `network.client` | 외부 서버로 나가는 연결 |
| 파일 선택 후 읽기 | `files.user-selected.read-only` | 원본을 수정하지 않음 |
| 파일 선택 후 저장·수정 | `files.user-selected.read-write` | 사용자가 고른 파일을 변경함 |
| 앱으로 들어오는 연결 | `network.server` | 로컬 서버나 소켓을 열 때만 |

`file_selector`를 쓴다고 `read-write`를 무조건 추가하면 안 된다. 필요한 범위보다 넓은 권한을 주면 심사와 보안 검토에서 설명할 내용이 늘어난다.

## Release 서명 결과를 직접 확인한다

빌드 성공만으로 entitlement가 최종 앱에 들어갔다고 판단할 수 없다. 앱 번들에 실제로 서명된 값을 확인하는 명령은 아래와 같다.

```bash
flutter clean
flutter build macos --release

codesign -d --entitlements :- \
  "build/macos/Build/Products/Release/앱이름.app"
```

출력에 `com.apple.security.network.client`가 없으면 Xcode의 Signing & Capabilities 화면만 수정했거나 Release 설정이 다른 파일을 가리키는 경우가 많다. Flutter 문서는 capabilities 편집기가 한 파일만 바꾸거나 새 entitlement 파일을 만들어 모든 구성에 연결하는 문제도 경고한다. 두 파일을 직접 비교하고, 실제 Release 앱에서 API 호출을 한 번 수행해야 한다.

## 출시 전 체크리스트

- [ ] DebugProfile과 Release entitlement에 필요한 항목이 모두 있다.
- [ ] HTTP가 아니라 HTTPS를 쓰는지 확인했다.
- [ ] `codesign`으로 Release `.app`의 서명된 권한을 확인했다.
- [ ] 파일 접근은 사용자가 파일 선택 창에서 고른 경로로만 테스트했다.
- [ ] App Store용 sandbox 앱과 로컬 개발 앱을 같은 권한으로 가정하지 않았다.

Flutter의 [macOS App Sandbox 안내](https://docs.flutter.dev/platform-integration/macos/building)는 `network.client` 누락 시 같은 `Operation not permitted` 예시를 보여준다. Apple의 [App Sandbox 문서](https://developer.apple.com/documentation/xcode/configuring-the-macos-app-sandbox)도 제한된 자원은 entitlement로 의도를 선언해야 한다고 설명한다. 2026년 9월 29일 확인 기준으로, 이 문제는 Dio나 HTTP 클라이언트를 교체해서 해결하는 유형이 아니다. Release 서명에 필요한 권한만 넣고 실제 배포 번들에서 검증하는 것이 가장 짧은 해결 경로다.
