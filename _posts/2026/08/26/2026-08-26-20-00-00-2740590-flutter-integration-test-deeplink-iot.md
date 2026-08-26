---
layout: post
title: "Flutter integration_test 딥링크 진입 검증 - IoT 공간 상태 복구"
description: "Flutter integration_test에서 딥링크로 IoT 공간 화면에 진입할 때 로그인·공간 선택·MQTT 구독 상태가 꼬이는 문제와 검증 방법을 정리했다."
date: 2026-08-26
tags: [Flutter, Dart, IoT, 스마트홈, CI/CD]
comments: true
share: true
---

![Flutter integration_test 딥링크와 IoT 스마트홈 화면](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

딥링크를 눌렀을 때 Flutter IoT 앱이 특정 공간의 제어 화면까지 제대로 들어가는지는 일반 위젯 테스트만으로 놓치기 쉽다. 앱이 이미 실행 중인지, 로그인 토큰이 있는지, 공간 목록을 불러왔는지에 따라 라우팅 결과가 달라지기 때문이다. `integration_test`로 실제 앱을 띄우고 딥링크 입력부터 MQTT 구독 준비까지 한 번에 검증하면 이 문제를 잡을 수 있다.

## 테스트에서 재현한 문제

알림 payload에 다음과 같은 링크가 들어온다고 가정했다.

```text
myhome://space/space-123/device/boiler-7
```

처음 구현에서는 URL만 파싱해서 바로 `DeviceControlPage`를 열었다. 콜드 스타트에서는 공간 정보와 인증 상태가 아직 준비되지 않아 화면에 `space-123`이 표시되지만, MQTT 토픽은 기본 공간으로 구독되는 문제가 생겼다. 앱을 백그라운드에서 복귀시킨 경우에는 화면만 이동하고 기기 상태가 갱신되지 않았다.

딥링크 테스트에서 확인할 상태를 세 단계로 나눴다.

| 단계 | 검증할 내용 | 실패할 때 보이는 증상 |
| --- | --- | --- |
| URL 파싱 | spaceId와 deviceId 추출 | 엉뚱한 화면 또는 404 |
| 세션·공간 준비 | 로그인과 공간 선택 완료 | 로딩 화면에 멈춤 |
| IoT 연결 | 올바른 MQTT 토픽 구독 | 보일러 상태가 갱신되지 않음 |

## 라우팅 준비를 기다리는 진입점

핵심은 딥링크 처리 함수가 화면 이동을 바로 실행하지 않는 것이다. 세션과 공간을 준비한 뒤 `openDevice`를 호출하도록 순서를 고정했다. 이 함수는 실제 앱에서 알림 탭과 OS 딥링크 콜백 양쪽에서 재사용한다.

```dart
class DeepLinkCoordinator {
  DeepLinkCoordinator({
    required this.session,
    required this.spaceStore,
    required this.router,
  });

  final SessionService session;
  final SpaceStore spaceStore;
  final AppRouter router;

  Future<void> open(String rawUrl) async {
    final link = DeviceLink.tryParse(rawUrl);
    if (link == null) {
      router.go('/home');
      return;
    }

    await session.restore();
    if (!session.isSignedIn) {
      router.go('/login', extra: rawUrl);
      return;
    }

    await spaceStore.ensureLoaded();
    final space = spaceStore.find(link.spaceId);
    if (space == null) {
      router.go('/spaces');
      return;
    }

    await spaceStore.select(space.id);
    router.go('/space/${space.id}/device/${link.deviceId}');
  }
}
```

테스트에서 중요한 것은 `restore()`와 `ensureLoaded()`가 끝난 뒤 라우터가 호출되는지다. `pumpAndSettle()` 한 번으로 모든 비동기 작업이 끝난다고 가정하면 MQTT 연결 재시도 타이머 때문에 테스트가 오래 멈추거나, 반대로 화면만 그려진 시점에 검증하게 된다.

## integration_test 시나리오

실제 네트워크 대신 테스트 빌드에서 세션·공간 저장소·MQTT 서비스를 Fake로 교체했다. 앱 전체를 띄우되 외부 서버에 의존하지 않으므로 CI에서도 같은 결과를 얻을 수 있다.

```dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('콜드 스타트 딥링크가 올바른 IoT 기기로 이동한다',
      (tester) async {
    final fakeSession = FakeSessionService(signedIn: true);
    final fakeSpaces = FakeSpaceStore(
      spaces: [Space(id: 'space-123', name: '우리 집')],
    );
    final fakeMqtt = FakeMqttService();

    await tester.pumpWidget(
      TestApp(
        session: fakeSession,
        spaceStore: fakeSpaces,
        mqtt: fakeMqtt,
        initialLink: 'myhome://space/space-123/device/boiler-7',
      ),
    );

    await tester.pump(const Duration(milliseconds: 100));
    await tester.pumpAndSettle();

    expect(find.text('우리 집'), findsOneWidget);
    expect(find.byKey(const ValueKey('device-boiler-7')), findsOneWidget);
    expect(fakeSpaces.selectedId, 'space-123');
    expect(fakeMqtt.subscribedTopics, contains('space/space-123/device/boiler-7'));
  });
}
```

여기서 `initialLink`는 실제 OS 이벤트를 흉내 낸 값이다. Android App Link나 iOS Universal Link 자체의 인증 설정까지 이 테스트에서 검증하는 것은 아니다. OS가 앱을 실행한 뒤 전달한 문자열을 앱이 올바르게 처리하는지에 범위를 둬야 테스트가 흔들리지 않는다.

## 백그라운드 복귀 링크는 별도 검증

앱이 이미 실행 중일 때는 초기화가 끝난 상태라고 착각하기 쉽다. 실제로는 MQTT 재연결 중에 링크가 들어올 수 있으므로, 복귀 시나리오에서는 구독 완료 이벤트를 기다린 뒤 화면을 확인했다.

```dart
await tester.binding.defaultBinaryMessenger.handlePlatformMessage(
  'app.lifecycle',
  const StandardMethodCodec().encodeMethodCall(
    const MethodCall('resumed'),
  ),
  (_) {},
);

await app.handleDeepLink(
  'myhome://space/space-123/device/boiler-7',
);
await tester.pumpUntilFound(find.byKey(const ValueKey('device-boiler-7')));

expect(fakeMqtt.connectCalls, 1);
expect(fakeMqtt.subscribeCalls, 1);
```

실제 프로젝트에서는 플랫폼 채널 이름이 다르므로 위 코드를 그대로 복사하기보다, 생명주기와 딥링크를 앱 내부 인터페이스로 감싼 뒤 Fake 이벤트를 주입하는 편이 낫다. 테스트가 플랫폼 채널 문자열에 묶이면 Android와 iOS 실행 결과가 달라질 때 원인을 찾기 어렵다.

## 실패했던 가정과 체크리스트

처음에는 라우트가 바뀌면 MQTT 구독도 자연스럽게 바뀔 것이라 생각했다. 하지만 구독은 Controller 초기화 시점에 한 번만 실행되고 있었다. 그래서 딥링크로 공간을 바꾸는 순간 기존 토픽을 해제하고 새 토픽을 등록하는 책임을 `SpaceStore.select()` 이후에 명시했다.

- 콜드 스타트와 백그라운드 복귀를 분리해 테스트한다.
- 로그인 실패 시 원래 딥링크를 보존해 로그인 완료 후 재처리한다.
- 존재하지 않는 공간·기기는 안전한 목록 화면으로 보낸다.
- 화면 표시뿐 아니라 선택된 `spaceId`와 MQTT topic까지 검사한다.
- `pumpAndSettle()`에만 의존하지 말고 Fake 서비스의 완료 신호를 기다린다.

정리하면 딥링크 테스트의 기준은 “해당 화면이 떴는가”에서 끝나면 안 된다. Flutter IoT 앱에서는 URL의 공간, 현재 선택 상태, MQTT 구독 대상이 모두 같은지 확인해야 실제 알림 탭 오류를 막을 수 있다.
