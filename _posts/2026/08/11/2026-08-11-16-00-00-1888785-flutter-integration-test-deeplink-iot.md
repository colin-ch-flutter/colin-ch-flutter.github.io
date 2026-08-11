---
layout: post
title: "Flutter integration_test 딥링크 테스트 - FCM에서 IoT 제어 화면까지 이동 검증"
description: "Flutter integration_test로 스마트홈 딥링크를 실행해 거실 보일러 상세 화면으로 이동하는지, 잘못된 기기 ID와 로그인 만료를 어떻게 검증하는지 정리했다."
date: 2026-08-11
tags: [Flutter, Dart, IoT, 스마트홈, integration_test, FCM, Android, iOS]
comments: true
share: true
---

![Flutter integration_test 딥링크로 스마트홈 기기 화면을 여는 모바일 화면](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Flutter integration_test 딥링크 테스트는 URL 문자열만 확인하는 테스트가 아니다. `myhome://device/boiler-01`을 눌렀을 때 로그인 상태에 따라 거실 보일러 화면을 열고, 잘못된 기기라면 오류 화면으로 보내는 전체 흐름을 검증해야 한다. FCM 알림을 눌렀는데 홈 화면만 뜨던 문제를 이 방식으로 잡았다.

## 라우트 테스트만으로 놓친 문제

처음에는 라우터에 문자열을 넣고 결과만 확인했다.

```dart
testWidgets('기기 딥링크를 해석한다', (tester) async {
  final route = parseDeepLink('myhome://device/boiler-01');

  expect(route, '/device/boiler-01');
});
```

파서는 통과했지만 실제 앱에서는 세 가지 문제가 남았다. 앱이 백그라운드에 있으면 기존 Navigator 위에 화면이 쌓였고, 로그인 토큰이 만료된 상태에서는 기기 화면이 로딩 인디케이터에서 멈췄다. `deviceId`가 없는 링크도 `/device/null`로 이동했다.

| 시나리오 | 기대 화면 | 실패하기 쉬운 지점 |
| --- | --- | --- |
| 로그인 + 유효한 기기 | 기기 상세 화면 | 기존 스택 중복 |
| 로그인 만료 | 로그인 화면 | 로딩 상태 고착 |
| 존재하지 않는 기기 | 기기 없음 화면 | null 라우트 이동 |
| 로그아웃 + 링크 재진입 | 로그인 후 원래 기기 | intent 유실 |

이 표에서 봐야 할 부분은 같은 URL이라도 앱 상태에 따라 기대 결과가 달라진다는 점이다.

## 딥링크를 Intent로 한 번 감싼다

`integration_test`에서 플랫폼 채널을 직접 조작하면 Android와 iOS 테스트가 갈라진다. 앱 내부에서는 외부 URL을 곧바로 Navigator에 넘기지 않고, 상태를 가진 `DeepLinkIntent`로 변환했다.

```dart
class DeepLinkIntent {
  const DeepLinkIntent.device(this.deviceId);
  const DeepLinkIntent.loginRequired(this.originalUrl);

  final String? deviceId;
  final String? originalUrl;
}

DeepLinkIntent? parseDeepLink(String rawUrl) {
  final uri = Uri.tryParse(rawUrl);
  if (uri?.scheme != 'myhome' || uri?.host != 'device') return null;

  final deviceId = uri!.pathSegments.isEmpty ? null : uri.pathSegments.first;
  if (deviceId == null || deviceId.isEmpty) return null;
  return DeepLinkIntent.device(deviceId);
}
```

테스트에서는 플랫폼에서 링크를 여는 과정을 재현하는 대신, 앱이 시작될 때 받을 수 있는 intent 주입 지점을 사용했다. 실제 네이티브 링크 실행은 Patrol이나 별도 디바이스 스크립트에서 한 번 확인하고, 화면 전환 규칙은 `integration_test`에서 반복 실행하는 구조다.

```dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('유효한 기기 딥링크가 상세 화면을 연다', (tester) async {
    await launchTestApp(
      authState: AuthState.loggedIn,
      initialLink: 'myhome://device/boiler-01',
    );

    await tester.pumpAndSettle();

    expect(find.text('거실 보일러'), findsOneWidget);
    expect(find.text('현재 온도'), findsOneWidget);
  });

  testWidgets('로그인 만료 링크는 원래 목적지를 보존한다', (tester) async {
    await launchTestApp(
      authState: AuthState.expired,
      initialLink: 'myhome://device/boiler-01',
    );

    await tester.pumpAndSettle();
    expect(find.text('로그인'), findsOneWidget);

    await tester.enterText(find.byKey(const Key('email')), 'test@example.com');
    await tester.tap(find.text('로그인하기'));
    await tester.pumpAndSettle();

    expect(find.text('거실 보일러'), findsOneWidget);
  });
}
```

## 백그라운드와 중복 링크 처리

실패 원인은 `initialLink`만 처리하고 앱이 실행된 뒤 들어오는 링크를 무시한 데 있었다. `app_links`의 stream 이벤트를 `DeepLinkCoordinator`가 받고, 로그인 중이면 pending intent로 저장했다. 같은 링크가 FCM 탭과 stream에서 두 번 들어와도 한 번만 소비하도록 ID를 기록했다.

```dart
await coordinator.handle(
  DeepLinkIntent.device('boiler-01'),
);

expect(find.byKey(const Key('device-detail-boiler-01')), findsOneWidget);
```

실기기에서 확인할 때는 Android의 `adb shell am start`와 iOS의 실제 링크 탭을 각각 사용했다. iOS 시뮬레이터에서는 알림과 앱 생명주기가 다르게 동작해 통과 결과를 그대로 믿기 어려웠다. 링크 파싱은 빠른 테스트로, OS에서 앱을 깨우는 과정은 Patrol smoke 테스트로 분리하는 편이 안정적이다.

## 짧게 정리하면

딥링크 테스트의 기준은 URL 파싱 성공이 아니라 `앱 상태 + 목적지 + 재진입 결과`다. `DeepLinkIntent`로 플랫폼 의존성을 줄이고, 로그인 만료·잘못된 기기·백그라운드 재진입을 각각 시나리오로 만들면 FCM 알림을 눌렀을 때 엉뚱한 화면이 뜨는 문제를 CI에서도 재현할 수 있다.
