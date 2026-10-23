---
layout: post
title: "Flutter iOS BGTaskScheduler 오류 - debug에서는 되는데 release 백그라운드 작업이 안 도는 조건"
description: "Flutter iOS 백그라운드 작업이 debug에서는 실행되지만 TestFlight·release에서 멈출 때 BGTaskScheduler 식별자, Capabilities, Info.plist와 검증 명령을 점검한다."
date: 2026-10-23
tags: [Flutter, iOS, 배포·운영, 테스트, CI/CD]
comments: true
share: true
---

![Flutter iOS release 백그라운드 작업과 BGTaskScheduler 설정을 점검하는 화면](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?auto=format&fit=crop&w=1600&q=80)

이 그림에서 볼 부분은 앱 화면이 아니라 iOS가 백그라운드 작업을 허용하는 설정과 실제 task 식별자다.

Flutter iOS 백그라운드 작업이 Xcode로 실행한 debug에서는 동작하고 TestFlight나 release에서는 멈춘다면, Dart 콜백부터 고치기보다 `BGTaskScheduler` 등록 조건을 확인하는 편이 맞다. 이 문제는 `Info.plist`, Xcode의 Background Modes, 네이티브 등록 코드의 식별자가 하나라도 다를 때 생긴다. 단, iOS는 주기 실행 시각을 보장하지 않으므로 매분 동기화나 즉시 알림이 목적이면 서버 push나 별도 foreground 설계가 필요하다.

## debug 성공이 release 성공을 뜻하지 않는 이유

iOS는 허용된 식별자와 launch handler를 확인한 뒤 배터리·사용 패턴·네트워크 상태를 보고 실행 시점을 정한다. `BGAppRefreshTask`는 짧은 새로고침, `BGProcessingTask`는 비교적 무거운 동기화에 쓰인다.

| 점검 대상 | 빠졌을 때 보이는 증상 | 확인 기준 |
|---|---|---|
| Background Modes | 예약은 성공하지만 실행되지 않음 | `fetch` 또는 `processing` 선택 |
| `BGTaskSchedulerPermittedIdentifiers` | `notPermitted`, 작업 미실행 | Info.plist 배열에 식별자 포함 |
| AppDelegate 등록 | launch 직후 등록 오류 또는 무반응 | 앱 시작 초기에 handler 등록 |
| 식별자 문자열 | debug와 TestFlight 결과가 다름 | Dart·Info.plist·Swift가 완전히 일치 |
| 시스템 실행 조건 | 일정한 시간에 실행되지 않음 | iOS 정책상 지연·생략 가능 |

## Workmanager를 쓸 때 먼저 맞출 값

작업 이름을 `com.example.app.refresh`로 정했다면 Dart·Info.plist·Swift에서 짧은 `refresh`를 섞지 않는다. 아래는 세 위치의 문자열을 맞추는 최소 예시다.

```xml
<!-- ios/Runner/Info.plist -->
<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>com.example.app.refresh</string>
</array>
```

Xcode의 Runner target에서 Background Modes를 추가하고 작업 종류에 맞게 `Background fetch` 또는 `Background processing`을 켠다. `Info.plist`만 추가하고 capability를 빠뜨리면 `BGTaskScheduler.Error.Code.notPermitted`가 날 수 있다.

작업 등록은 앱 launch sequence가 끝나기 전에 이뤄져야 한다. Workmanager 버전에 따라 자동 등록을 제공할 수 있으므로 네이티브 코드를 중복으로 추가하지 말고, 현재 버전의 등록 규칙을 확인한다.

```swift
// AppDelegate.swift의 앱 시작 경로에서 호출하는 형태
WorkmanagerPlugin.registerBGProcessingTask(
    withIdentifier: "com.example.app.refresh"
)
```

```dart
const taskId = 'com.example.app.refresh';

await Workmanager().registerOneOffTask(
  taskId,
  taskId,
  constraints: Constraints(networkType: NetworkType.connected),
);
```

## release에서 확인할 순서

1. Release target의 `Info.plist`와 Debug plist를 비교한다.
2. 식별자와 `UIBackgroundModes`를 확인한다. `notPermitted`면 이 단계가 우선이다.
3. Background App Refresh가 켜져 있는지 확인한다.
4. 등록 직후 `printScheduledTasks()` 결과가 비어 있지 않은지 본다.
5. Xcode Debug 메뉴에서 Background Fetch 또는 BGTask를 시뮬레이션한다.
6. TestFlight에서 즉시 실행을 기대하지 않는다. iOS는 실행 시각을 보장하지 않는다.

작업이 예약 목록에는 있지만 실행되지 않으면 시스템 지연, Background App Refresh, 네트워크 조건을 분리한다. 콜백에는 짧고 재실행 가능한 동기화만 둔다.

## 출시 전 체크리스트

- [ ] Release target의 `Info.plist`에 모든 task identifier가 있다.
- [ ] Xcode Runner target의 Background Modes가 작업 종류와 맞다.
- [ ] Dart·AppDelegate·Info.plist 식별자가 완전히 같다.
- [ ] 앱 launch 초기에 handler가 등록된다.
- [ ] 예약 목록에 작업이 실제로 보인다.
- [ ] 네트워크 없음·Background App Refresh 꺼짐을 실패 조건으로 기록했다.
- [ ] “정확한 시각 실행”이 필요한 기능을 백그라운드 작업 하나에 맡기지 않았다.

iOS 백그라운드 작업은 콜백을 계속 살려두는 기능이 아니다. 예약·등록·실행을 각각 확인하고, 정확한 시각이 필요한 기능은 push나 foreground 설계로 분리한다.

참고 문서: [Flutter 백그라운드 프로세스 안내](https://docs.flutter.dev/packages-and-plugins/background-processes), [Apple 백그라운드 작업 설정](https://developer.apple.com/documentation/uikit/using-background-tasks-to-update-your-app), [`BGTaskSchedulerPermittedIdentifiers` 공식 문서](https://developer.apple.com/documentation/bundleresources/information-property-list/bgtaskschedulerpermittedidentifiers), [Flutter Workmanager 디버깅 안내](https://docs.page/fluttercommunity/flutter_workmanager~647/debugging)
