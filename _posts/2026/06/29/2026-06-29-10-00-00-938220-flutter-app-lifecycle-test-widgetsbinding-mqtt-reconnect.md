---
layout: post
title: "Flutter 앱 생명주기 테스트 — WidgetsBindingObserver로 MQTT 재연결 로직 검증"
description: "AppLifecycleState.paused/resumed 전환을 테스트 코드에서 시뮬레이션하는 방법과, IoT 앱에서 백그라운드 전환 시 MQTT 자동 재연결 로직을 단위 테스트로 검증하는 실전 패턴을 정리했다."
date: 2026-06-29
tags: [Flutter, MQTT, mqtt5_client, IoT, 스마트홈]
comments: true
share: true
---

# Flutter 앱 생명주기 테스트 — WidgetsBindingObserver로 MQTT 재연결 로직 검증

![Flutter 앱 생명주기와 MQTT 연결 테스트](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

앱 생명주기 테스트는 처음에 어떻게 접근해야 할지 감이 안 잡혔다. `AppLifecycleState`가 바뀌는 걸 테스트 코드에서 어떻게 시뮬레이션하나? 실기기에서 홈 버튼 눌러보는 것 외엔 방법이 없는 줄 알았다. 근데 `WidgetsBinding.instance.handleAppLifecycleStateChanged()`를 직접 호출하면 된다. IoT 앱에서 백그라운드 전환 시 MQTT가 끊기고 복귀 시 자동 재연결하는 로직을 이 방식으로 완전히 단위 테스트할 수 있다.

## 왜 생명주기 테스트가 필요한가

스마트홈 앱에서 MQTT는 앱이 살아있는 동안 계속 연결을 유지해야 한다. 문제는 사용자가 앱을 백그라운드로 보냈다가 복귀할 때다.

실제로 겪은 버그: 보일러 제어 앱에서 사용자가 앱을 5분 정도 백그라운드에 두었다가 복귀하면, UI에는 '연결됨'이 표시되는데 실제로 MQTT는 끊긴 상태였다. MQTT keep-alive 시간(30초)이 지나면 브로커가 연결을 끊는데, 앱 쪽에서는 그걸 감지 못하고 있었던 것이다.

이런 흐름을 자동화된 테스트로 잡아두지 않으면, 릴리스마다 수동으로 확인해야 한다.

## WidgetsBindingObserver 구현 먼저

테스트하려면 먼저 테스트 가능한 구조로 코드를 짜야 한다. `MqttController`가 `WidgetsBindingObserver`를 구현하고 `WidgetsBinding`에 직접 의존하면 테스트가 어렵다. Binding을 추상화로 감싸는 게 핵심이다.

`LifecycleObserver` 추상 인터페이스를 만들면 테스트에서 교체할 수 있다:

```dart
abstract class AppLifecycleListener {
  void addObserver(WidgetsBindingObserver observer);
  void removeObserver(WidgetsBindingObserver observer);
}

class RealAppLifecycleListener implements AppLifecycleListener {
  @override
  void addObserver(WidgetsBindingObserver observer) {
    WidgetsBinding.instance.addObserver(observer);
  }

  @override
  void removeObserver(WidgetsBindingObserver observer) {
    WidgetsBinding.instance.removeObserver(observer);
  }
}
```

`MqttController`는 이 추상 인터페이스를 주입받는다:

```dart
class MqttController extends GetxController with WidgetsBindingObserver {
  final MqttRepository _mqttRepository;
  final AppLifecycleListener _lifecycleListener;

  MqttController({
    required MqttRepository mqttRepository,
    AppLifecycleListener? lifecycleListener,
  })  : _mqttRepository = mqttRepository,
        _lifecycleListener = lifecycleListener ?? RealAppLifecycleListener();

  @override
  void onInit() {
    super.onInit();
    _lifecycleListener.addObserver(this);
    _connect();
  }

  @override
  void onClose() {
    _lifecycleListener.removeObserver(this);
    super.onClose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
      case AppLifecycleState.paused:
        _mqttRepository.disconnect();
      case AppLifecycleState.resumed:
        _mqttRepository.reconnect();
      default:
        break;
    }
  }
}
```

## 테스트에서 생명주기 시뮬레이션하기

테스트용 Fake Lifecycle Listener를 만들고, `WidgetsBinding.instance.handleAppLifecycleStateChanged()`로 상태 변화를 직접 트리거한다.

`FakeAppLifecycleListener`는 등록된 옵저버 목록을 들고 있다가 나중에 테스트에서 상태를 주입할 수 있게 해준다:

```dart
class FakeAppLifecycleListener implements AppLifecycleListener {
  final List<WidgetsBindingObserver> _observers = [];

  List<WidgetsBindingObserver> get observers => List.unmodifiable(_observers);

  @override
  void addObserver(WidgetsBindingObserver observer) {
    _observers.add(observer);
  }

  @override
  void removeObserver(WidgetsBindingObserver observer) {
    _observers.remove(observer);
  }

  void notifyStateChange(AppLifecycleState state) {
    for (final observer in _observers) {
      observer.didChangeAppLifecycleState(state);
    }
  }
}
```

그리고 `FakeMqttRepository`도 필요하다. 호출 기록을 추적해서 실제로 `disconnect()`와 `reconnect()`가 호출됐는지 검증한다:

```dart
class FakeMqttRepository implements MqttRepository {
  int disconnectCallCount = 0;
  int reconnectCallCount = 0;
  bool isConnected = true;

  @override
  Future<void> disconnect() async {
    disconnectCallCount++;
    isConnected = false;
  }

  @override
  Future<void> reconnect() async {
    reconnectCallCount++;
    isConnected = true;
  }

  @override
  Future<void> connect() async {
    isConnected = true;
  }
}
```

## 실제 테스트 코드

`flutter_test`와 `get` 패키지만 있으면 된다. `WidgetsBinding`은 `testWidgets` 안에서 자동으로 초기화된다:

```dart
void main() {
  late FakeMqttRepository fakeMqttRepository;
  late FakeAppLifecycleListener fakeLifecycle;
  late MqttController controller;

  setUp(() {
    fakeMqttRepository = FakeMqttRepository();
    fakeLifecycle = FakeAppLifecycleListener();
    controller = MqttController(
      mqttRepository: fakeMqttRepository,
      lifecycleListener: fakeLifecycle,
    );
  });

  tearDown(() {
    controller.onClose();
  });

  group('앱 생명주기 MQTT 재연결', () {
    testWidgets('paused 전환 시 disconnect 호출', (tester) async {
      controller.onInit();

      // 앱이 백그라운드로 전환됨
      fakeLifecycle.notifyStateChange(AppLifecycleState.paused);

      expect(fakeMqttRepository.disconnectCallCount, 1);
      expect(fakeMqttRepository.isConnected, false);
    });

    testWidgets('paused 후 resumed 전환 시 reconnect 호출', (tester) async {
      controller.onInit();

      fakeLifecycle.notifyStateChange(AppLifecycleState.paused);
      fakeLifecycle.notifyStateChange(AppLifecycleState.resumed);

      expect(fakeMqttRepository.disconnectCallCount, 1);
      expect(fakeMqttRepository.reconnectCallCount, 1);
      expect(fakeMqttRepository.isConnected, true);
    });

    testWidgets('여러 번 백그라운드/포그라운드 반복', (tester) async {
      controller.onInit();

      fakeLifecycle.notifyStateChange(AppLifecycleState.paused);
      fakeLifecycle.notifyStateChange(AppLifecycleState.resumed);
      fakeLifecycle.notifyStateChange(AppLifecycleState.paused);
      fakeLifecycle.notifyStateChange(AppLifecycleState.resumed);

      expect(fakeMqttRepository.disconnectCallCount, 2);
      expect(fakeMqttRepository.reconnectCallCount, 2);
    });

    testWidgets('onClose 시 옵저버 해제 확인', (tester) async {
      controller.onInit();
      expect(fakeLifecycle.observers.length, 1);

      controller.onClose();
      expect(fakeLifecycle.observers.length, 0);
    });
  });
}
```

## WidgetsBinding.instance 직접 사용 방식과 차이

처음에는 `FakeAppLifecycleListener` 없이 `WidgetsBinding.instance.handleAppLifecycleStateChanged(state)`를 테스트에서 직접 호출하는 방식을 써봤다. 이것도 된다. 근데 이 방식은 `TestWidgetsFlutterBinding`이 초기화된 상태에서만 작동하고, `testWidgets` 안에서만 써야 한다는 제약이 있다.

반면 `FakeAppLifecycleListener` 패턴은 `test()` 안에서도 쓸 수 있고, Flutter 바인딩에 의존하지 않아 더 순수한 단위 테스트가 된다. Dart 전용 환경에서도 실행 가능하다.

IoT 앱처럼 생명주기 이벤트가 비즈니스 로직에 직접 영향을 주는 경우엔 Fake 패턴이 훨씬 낫다.

![Flutter 앱 생명주기 상태 전환 흐름](https://images.unsplash.com/photo-1504639725590-34d0984388bd?w=800&q=80)

## inactive 상태는 무시하는 이유

`AppLifecycleState.inactive`도 있는데 처리를 안 한다. iOS에서 전화가 올 때나 멀티태스킹 스와이퍼를 열 때 잠깐 이 상태로 전환됐다가 바로 돌아오는 경우가 많다. 이 타이밍에 MQTT를 끊었다 다시 연결하면 오히려 노이즈가 생긴다. 실제 테스트해보니 `inactive` 구간은 수백 밀리초 수준이라, 여기서 disconnect를 날리면 `resumed` 시점에 reconnect 요청이 겹쳐서 연결이 꼬이는 경우가 있었다.

`paused`는 확실히 "앱이 보이지 않는 상태"이므로 이 시점에 처리하는 게 안전하다.

## detached 상태 처리

`detached`는 앱이 완전히 종료되기 직전이다. 여기서 `disconnect()`를 호출하는 게 맞긴 한데, 실제로 이 콜백이 항상 불린다는 보장이 없다. Android에서는 프로세스가 강제 종료되면 이 콜백 없이 그냥 죽는다.

그래서 MQTT에서 `will` 메시지를 설정해두는 게 더 신뢰할 수 있는 오프라인 감지 방법이다. MQTT Will과 keep-alive 설계는 이전에 따로 정리해뒀다.

---

다음 편은 `easy_localization`을 사용하는 위젯에서 다국어 테스트를 격리하는 방법이다. `Locale('ko', 'KR')` 환경에서만 보이는 UI 버그를 테스트로 잡는 패턴을 다룬다.

**참고 링크**
- [Flutter AppLifecycleState API 문서](https://api.flutter.dev/flutter/dart-ui/AppLifecycleState.html)
- [WidgetsBindingObserver 공식 문서](https://api.flutter.dev/flutter/widgets/WidgetsBindingObserver-mixin.html)
- [mqtt5_client pub.dev](https://pub.dev/packages/mqtt5_client)
