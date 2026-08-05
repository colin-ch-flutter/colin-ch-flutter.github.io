---
layout: post
title: "Flutter integration_test 실패 로그·스크린샷 수집 - CI에서 IoT 테스트 원인 찾기"
description: "Flutter integration_test가 CI에서만 실패할 때 화면 스크린샷과 debug 로그를 남겨 IoT 앱의 타이밍·권한·화면 문제를 재현하고 원인을 찾는 방법을 정리했다."
date: 2026-08-05
tags: [Flutter, integration_test, IoT, Android, CI/CD, Firebase]
comments: true
share: true
---

![Flutter integration_test 실패 화면과 CI 로그 아티팩트](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

Flutter integration_test가 로컬에서는 통과하고 CI에서만 실패한다면 재실행부터 하지 말고 실패 당시의 화면과 로그를 남겨야 한다. IoT 앱은 BLE 권한, MQTT 연결, 비동기 카드 갱신이 얽혀 있어서 `Expected: ...` 한 줄만으로는 원인을 알기 어렵다. Firebase Test Lab에서도 스크린샷 하나가 수십 줄의 추측보다 빨랐다.

## 실패 로그만으로는 부족했던 이유

처음에는 `flutter test integration_test`의 콘솔 출력만 GitHub Actions에 저장했다. 실패 메시지는 `findsOneWidget`이었지만, 실제 원인은 세 가지 중 하나였다.

| 실패 메시지 | 실제로 확인된 원인 | 필요한 자료 |
|---|---|---|
| 제어 버튼을 찾지 못함 | 권한 다이얼로그가 아직 남음 | 실패 화면 캡처 |
| MQTT 상태가 connected 아님 | Fake 서비스 주입 누락 | 앱 시작 로그 |
| 타임아웃 | 작은 화면에서 로딩 표시 위치 변경 | 화면 캡처 + 타임스탬프 |

테스트가 실패한 순간의 위젯 트리를 얻는 방법도 있지만, 네이티브 권한 팝업이나 실제 화면 크기 문제는 위젯 트리만 봐서는 놓친다. 그래서 테스트 단계마다 화면 캡처를 남기고, `debugPrint`에 시나리오와 시간을 붙였다.

## 테스트 코드에 캡처 지점을 둔다

스크린샷은 실패 시점에만 찍는 것보다 상태가 바뀌는 경계에 찍는 편이 분석하기 쉽다. `IntegrationTestWidgetsFlutterBinding`의 `takeScreenshot`을 감싼 헬퍼를 만들었다.

```dart
import 'dart:convert';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

class TestTrace {
  TestTrace(this.binding);

  final IntegrationTestWidgetsFlutterBinding binding;
  int _index = 0;

  Future<void> mark(String name) async {
    final label = '${_index++}_$name';
    final now = DateTime.now().toIso8601String();
    // CI 로그에서 단계별 지연 시간을 확인한다.
    print(jsonEncode({'step': label, 'time': now}));
    await binding.takeScreenshot(label);
  }
}

void main() {
  final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  final trace = TestTrace(binding);

  testWidgets('보일러 제어 흐름', (tester) async {
    await launchTestApp(tester);
    await trace.mark('app_started');

    await tester.tap(find.text('거실'));
    await tester.pumpAndSettle(const Duration(seconds: 2));
    await trace.mark('room_selected');

    await tester.tap(find.text('난방 켜기'));
    await tester.pump(const Duration(milliseconds: 500));
    await trace.mark('command_sent');
    expect(find.text('가동 중'), findsOneWidget);
  });
}
```

실제 프로젝트에서는 `takeScreenshot`이 지원되지 않는 실행 환경도 있어 캡처 파일 이름을 로그에 함께 남겼다. `app_started`, `room_selected`, `command_sent`처럼 짧고 정렬 가능한 이름을 사용해야 여러 기기의 결과를 비교하기 편하다.

## 실패하면 테스트를 중단하지 않고 자료를 보존한다

CI에서는 테스트 명령 뒤에 `if: always()`인 업로드 단계를 둔다. 테스트가 실패해도 캡처와 로그를 업로드해야 하기 때문이다.

```yaml
- name: Run integration test
  id: integration
  run: flutter test integration_test/smarthome_test.dart -d emulator-5554

- name: Upload integration artifacts
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: integration-${{ github.run_number }}
    path: |
      build/integration_test/**
      test-results/**
      integration-test.log
    retention-days: 7
```

실행 명령의 표준 출력을 파일로도 남겨야 한다. 셸 설정에 따라 `tee`를 쓰면 테스트 종료 코드가 가려질 수 있으므로 `PIPESTATUS`를 확인하는 방식이 안전하다.

```bash
set -o pipefail
flutter test integration_test/smarthome_test.dart 2>&1 | tee integration-test.log
exit ${PIPESTATUS[0]}
```

여기서 흔히 빠뜨리는 부분은 캡처 경로다. 로컬에서 생성된 파일과 Android 테스트 러너가 저장한 파일의 위치가 다를 수 있으니, CI 마지막에 `find build test-results -type f`로 실제 경로를 한 번 확인하고 업로드 패턴을 고정했다.

## 로그에 넣을 정보의 기준

BLE MAC 주소나 JWT를 그대로 출력하면 안 된다. 대신 시나리오, 현재 공간 ID의 일부, 연결 상태, 경과 시간만 기록한다.

```text
[trace] room_selected elapsed=1842ms room=li***42 ble=disconnected mqtt=fake-connected
[trace] command_sent elapsed=2310ms command=heater_on ack=timeout
```

이 정도면 “버튼을 못 찾은 것”과 “명령 ACK가 늦은 것”을 구분할 수 있다. 실제로 한 번은 화면 캡처에서 권한 팝업이 보였고, 로그에는 `mqtt=connected`가 찍혀 있었다. 이 경우 MQTT 재연결 코드를 고치는 건 완전히 엉뚱한 방향이었다.

## 짧게 정리하면

- Flutter integration_test에는 상태 경계별 스크린샷을 남긴다.
- CI 업로드 단계는 `if: always()`로 실행해 실패 자료를 보존한다.
- 로그에는 시나리오와 경과 시간을 넣고 토큰·전체 식별자는 마스킹한다.
- Firebase Test Lab 기기 매트릭스를 늘리기 전에 캡처 경로와 테스트 종료 코드를 검증한다.

실패를 다시 재현하는 데 집착하면 CI 대기 시간만 길어진다. 실패 순간의 화면, 단계, 시간 세 가지를 확보하면 flaky 테스트도 수정 가능한 문제로 바뀐다.
