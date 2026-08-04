---
layout: post
title: "Flutter integration_test 데이터 초기화 - IoT 앱의 순서 의존성 끊기"
description: "Flutter integration_test에서 앱 상태와 로컬 데이터를 초기화해 IoT 제어 시나리오가 실행 순서에 따라 실패하는 문제를 해결하는 방법을 정리했다."
date: 2026-08-05
tags: [Flutter, Dart, Riverpod, IoT, Android, CI/CD]
comments: true
share: true
---

![Flutter integration_test 데이터 초기화와 실제 기기 테스트](https://images.unsplash.com/photo-1512941937669-90a1b58e7e9c?w=800&q=80)

이 그림에서 봐야 할 부분은 같은 앱이라도 실행 전 상태가 다르면 테스트 결과가 달라진다는 점이다.

Flutter integration_test가 혼자 실행할 때는 통과하는데 전체 파일을 돌리면 실패한다면, 테스트 코드보다 데이터 초기화를 먼저 의심하는 편이 빠르다. IoT 앱에서는 로그인 토큰, 선택된 공간, Realm 데이터, Riverpod Provider 상태가 한 번 남으면 보일러 제어 시나리오가 앞 테스트의 결과를 이어받는다. 나도 Firebase Test Lab에 올린 뒤에야 이 문제가 기기별 차이가 아니라 순서 의존성이라는 걸 확인했다.

## `setUp`만으로 부족했던 이유

`setUp`은 Dart 테스트 함수가 실행되기 전에 호출된다. 하지만 앱 프로세스가 이미 떠 있고, ProviderContainer나 로컬 DB가 앱 시작 과정에서 만들어졌다면 단순히 변수 몇 개를 비우는 것으로는 부족하다.

| 남아 있는 상태 | 실제 증상 | 초기화 위치 |
|---|---|---|
| SecureStorage 토큰 | 로그인 화면이 건너뛰어짐 | 앱 실행 전 |
| Realm 공간 데이터 | 이전 공간의 보일러가 표시됨 | 테스트 시작 전 |
| ProviderContainer | MQTT 상태가 `connected`로 남음 | 앱 재시작 시 |
| 서버 테스트 데이터 | 삭제 시나리오가 두 번째부터 실패 | API fixture |

특히 `flutter drive`처럼 앱을 한 번 띄운 채 여러 시나리오를 실행하면 `setUp`이 앱을 다시 시작해주지 않는다. 테스트마다 독립된 시작점을 만들려면 초기화 경계를 명확히 정해야 한다.

## 테스트 모드와 fixture를 분리한다

실제 서버를 매번 지우는 방식은 느리고, Test Lab에서 병렬 실행할 때 서로의 데이터를 건드릴 수 있다. 앱에 테스트 모드를 주입하고, 그 모드에서만 초기화 API와 고정 fixture를 사용했다.

`--dart-define`으로 테스트 모드를 넣으면 운영 빌드에 테스트용 초기화 코드가 섞이는 위험도 줄어든다.

```bash
# integration_test용 APK 실행 시 테스트 모드 주입
flutter test integration_test/smarthome_flow_test.dart \
  --dart-define=APP_ENV=test \
  --dart-define=RESET_FIXTURE=true
```

앱 시작점에서는 환경값을 읽어 테스트 Repository와 초기화 서비스를 선택한다.

```dart
const appEnv = String.fromEnvironment('APP_ENV', defaultValue: 'prod');
const resetFixture = bool.fromEnvironment(
  'RESET_FIXTURE',
  defaultValue: false,
);

Future<void> bootstrap() async {
  final storage = await SecureStorage.create();

  if (appEnv == 'test' && resetFixture) {
    await storage.deleteAll();
    await TestFixture.seedRealm();
  }

  runApp(App(
    overrides: appEnv == 'test' ? testOverrides : const [],
  ));
}
```

여기서 `deleteAll()`을 모든 실행에 넣으면 편해 보이지만, 로그인 상태를 유지하는 테스트까지 깨진다. 시나리오의 시작 조건을 `freshUser`, `loggedInUser`, `occupiedRoom`처럼 이름으로 나누고 필요한 fixture만 선택하는 게 낫다.

## 테스트마다 앱 상태를 다시 만든다

한 파일에서 여러 흐름을 묶어야 한다면 테스트 사이에 앱을 재시작하거나 ProviderContainer를 새로 만들어야 한다. Riverpod을 사용한다면 전역 container를 공유하지 않는 것이 핵심이다.

```dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  setUp(() async {
    await TestFixture.resetLocalData();
  });

  testWidgets('거실 보일러를 켜면 현재 공간 상태가 갱신된다', (tester) async {
    await app.main(testMode: true);
    await tester.pumpAndSettle(const Duration(seconds: 2));

    await tester.tap(find.byKey(const Key('room-living')));
    await tester.tap(find.byKey(const Key('boiler-on')));

    expect(find.text('켜짐'), findsOneWidget);
  });
}
```

실제 프로젝트에서는 `app.main()`을 여러 번 호출하는 것만으로 충분하지 않았다. 네이티브 플러그인과 이미 연결된 MQTT 클라이언트가 살아 있었기 때문이다. 테스트 전용 `disconnect()`와 `ProviderContainer.dispose()`를 앱 종료 훅에 넣고, fixture 초기화가 끝난 뒤 새 container를 만들도록 순서를 고정했다.

## Firebase Test Lab에서 확인할 기준

[Flutter Firebase Test Lab integration_test 실기기 테스트]({% post_url 2026-08-04-10-00-00-1703610-flutter-firebase-test-lab-integration-test-iot %})처럼 기기 매트릭스를 넓히기 전에 로컬에서 순서를 섞어 실행한다.

```bash
flutter test integration_test --test-randomize-ordering-seed=random
```

실패가 재현되면 로그에 `fixture`, `roomId`, `provider disposed`를 남긴다. “거실 보일러가 켜지지 않음”만 출력하면 화면 문제처럼 보이지만, 실제로는 침실 fixture가 남아 있던 경우가 있었다.

정리하면 세 가지다.

- 테스트 시작 조건을 fixture 이름으로 명시한다.
- 로컬 DB, SecureStorage, Provider, MQTT 연결을 각각 정리한다.
- Test Lab 기기 수를 늘리기 전에 실행 순서를 섞어 순서 의존성을 잡는다.

이렇게 경계를 나누면 integration_test가 느리더라도 실패 원인을 따라갈 수 있다. 테스트가 가끔 통과하는 상태를 성공으로 착각하지 않는 게 가장 중요하다.
