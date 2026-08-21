---
layout: post
title: "Flutter integration_test 뒤로가기 검증 - IoT 화면 스택과 MQTT 구독 정리"
description: "Flutter integration_test에서 Android 시스템 뒤로가기와 Flutter 라우트 pop을 검증하고, IoT 제어 화면을 빠져나올 때 MQTT 구독이 남는 문제를 잡는 실전 패턴을 정리했다."
date: 2026-08-22
tags: [Flutter, Dart, IoT, MQTT, Android, CI/CD]
comments: true
share: true
---
![Flutter integration_test Android 뒤로가기와 IoT 화면 검증](https://images.unsplash.com/photo-1551650975-87deedd944c3?w=1200&q=80)

Flutter integration_test에서 버튼 탭만 검증하다 보면 Android 시스템 뒤로가기에서만 깨지는 화면을 놓치기 쉽다. 특히 IoT 앱은 기기 상세 화면을 나갈 때 MQTT 구독을 해제해야 하는데, 라우트만 사라지고 구독은 살아 있는 경우가 있었다. 결론은 **뒤로가기 입력, 화면 스택, 구독 해제를 한 시나리오로 묶어 테스트해야 한다**는 것이다.

## 버튼 pop과 시스템 뒤로가기는 다르다

처음엔 앱바의 뒤로가기 아이콘을 누르는 테스트가 있으니 충분하다고 생각했다. 아니었다. Android back 입력은 `Navigator.pop` 직접 호출과 달리 `PopScope`와 라우트 상태를 거친다.

| 검증 대상 | 확인할 것 | 놓치면 생기는 문제 |
|---|---|---|
| 앱바 뒤로가기 | 상세 화면이 닫히는지 | 버튼 콜백만 정상일 수 있다 |
| Android back | 같은 라우트 결과인지 | 시스템 입력에서만 화면이 멈춘다 |
| MQTT 정리 | 구독 콜백이 끊겼는지 | 목록 화면에서도 이전 기기 이벤트가 온다 |

테스트는 실제 사용자 입력과 같은 경로를 지나가게 만든다. 아래 코드는 상세 화면에서 시스템 뒤로가기를 보내고 목록 화면의 고유 키를 확인하는 최소 예시다.

```dart
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('기기 상세에서 Android back으로 나가면 구독을 해제한다',
      (tester) async {
    await tester.pumpWidget(const TestApp(startPath: '/devices/boiler-1'));
    await tester.pumpAndSettle();

    expect(find.byKey(const Key('device-detail')), findsOneWidget);
    expect(TestMqttProbe.activeSubscriptions, contains('device/boiler-1'));

    await tester.binding.handlePopRoute();
    await tester.pumpAndSettle();

    expect(find.byKey(const Key('device-list')), findsOneWidget);
    expect(TestMqttProbe.activeSubscriptions, isNot(contains('device/boiler-1')));
  });
}
```

테스트가 통과하는데 구독이 남는다면 화면의 `dispose`에만 정리 코드를 넣었을 가능성이 크다. 라우트가 유지된 채 `canPop`이 false가 되는 흐름에서는 `dispose`가 바로 호출되지 않는다.

그래서 실제 구현에서는 화면 생명주기보다 라우트 종료 시점에 명시적으로 정리했다.

```dart
PopScope(
  canPop: true,
  onPopInvokedWithResult: (didPop, result) {
    if (didPop) controller.unsubscribeDevice();
  },
  child: const DeviceDetailView(),
)
```

단, `onPopInvokedWithResult`와 `dispose`에서 같은 해제 메서드를 둘 다 호출하면 중복 unsubscribe 로그가 나온다. `unsubscribeDevice()`를 여러 번 호출해도 안전한 멱등 메서드로 만들고, 테스트용 `TestMqttProbe`에는 현재 구독 목록을 노출한다.

솔직하게 정리하면, 뒤로가기 테스트의 핵심은 화면이 사라졌는지가 아니다. **이전 화면으로 돌아온 뒤에도 이전 기기의 이벤트가 더 이상 들어오지 않는지**까지 확인해야 한다. Android 시스템 입력과 MQTT 정리를 한 번에 검증하면 로컬에서는 멀쩡하고 실제 기기에서만 꼬이는 문제를 일찍 잡을 수 있다.
