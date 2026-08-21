---
layout: post
title: "Flutter integration_test 실패 증거 수집 - IoT CI에 스크린샷과 로그 남기기"
description: "Flutter integration_test가 CI에서만 실패할 때 테스트 이름·스크린샷·앱 로그를 GitHub Actions 아티팩트로 보존해 flaky 원인을 추적하는 실전 패턴을 정리했다."
date: 2026-08-21
tags: [Flutter, Dart, IoT, CI/CD, Android, Firebase]
comments: true
share: true
---
![Flutter integration_test CI 실패 증거 수집](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=1200&q=80)

Flutter integration_test는 로컬에서 통과하는데 CI에서만 실패할 때가 있다. 이때 실패 메시지만 남기면 `Expected: true, Actual: false` 정도만 보이고, 어떤 화면에서 멈췄는지 알 수 없다. IoT 앱은 MQTT 연결, 권한 다이얼로그, 비동기 기기 상태가 얽혀 있어서 더 답답하다.

결론은 간단하다. **각 시나리오에 테스트 이름을 붙이고, 실패 직후 스크린샷·로그·환경 정보를 한 디렉터리에 저장한 뒤 CI 아티팩트로 업로드해야 한다.** 나도 처음에는 assertion만 늘렸는데, 해보니 원인은 assertion이 아니라 그 앞에서 보일러 카드가 아직 로딩 중이었던 경우가 많았다.

## 실패 증거는 세 가지를 한 세트로 남긴다

| 증거 | 확인할 수 있는 것 | 없을 때 생기는 문제 |
|---|---|---|
| 테스트 이름 | 어느 시나리오인지 | 긴 integration_test 파일에서 위치를 찾기 어렵다 |
| 화면 스크린샷 | 실패 순간의 로딩·권한·에러 상태 | CI 기기에 직접 접속해야 한다 |
| 앱 로그 | MQTT·Repository·라우팅 흐름 | 마지막 assertion만 보고 추측하게 된다 |

화면 캡처는 테스트 코드에서 직접 남긴다. `takeScreenshot`은 테스트 이름을 파일명에 포함해야 여러 테스트가 동시에 실행돼도 덮어쓰지 않는다.

```dart
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';

final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();

Future<void> captureFailure(String name) async {
  final bytes = await binding.takeScreenshot(name);
  final dir = Directory('build/test-artifacts');
  await dir.create(recursive: true);
  await File('${dir.path}/$name.png').writeAsBytes(bytes);
}

testWidgets('보일러 전원 상태를 갱신한다', (tester) async {
  const testName = 'boiler-power-state';
  try {
    await tester.pumpWidget(const TestApp());
    await tester.pumpAndSettle(const Duration(seconds: 3));
    expect(find.text('켜짐'), findsOneWidget);
  } catch (error, stackTrace) {
    await captureFailure(testName);
    File('build/test-artifacts/$testName.log').writeAsStringSync(
      '$error\n$stackTrace',
    );
    rethrow;
  }
});
```

여기서 `pumpAndSettle`만 긴 시간으로 늘리는 건 해결책이 아니다. 실패 화면을 먼저 확보해야 권한 팝업이 가렸는지, MQTT Fake가 이벤트를 보내지 않았는지 구분할 수 있다. 앱 로그는 `debugPrint`와 함께 테스트용 `AppLogger`에도 기록한다.

## GitHub Actions에서 아티팩트 업로드

테스트가 실패해도 후속 단계가 실행되도록 `if: always()`를 붙인다. 이 조건이 없으면 테스트 실패와 함께 job이 끝나서 가장 필요한 파일이 업로드되지 않는다.

```yaml
- name: Run integration test
  run: flutter test integration_test/boiler_test.dart -d emulator-5554

- name: Upload integration test evidence
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: integration-test-evidence-${{ github.run_number }}
    path: build/test-artifacts/
    retention-days: 7
```

아티팩트 이름에 `run_number`를 넣으면 재실행 결과를 헷갈리지 않는다. IoT 로그에는 토픽명이나 계정 식별자가 섞일 수 있으므로 보존 기간은 7일 정도로 제한했다. BLE 연결 끊김이나 MQTT 타임아웃은 화면만 정상처럼 보일 수 있어 Flutter 버전, 기기 API 레벨, 테스트 파일명도 로그 첫 줄에 남기는 편이 좋다.

솔직하게 정리하면, integration_test의 실패를 재현하는 능력은 assertion 개수보다 **실패 순간을 보존하는 장치**에서 나온다. CI가 가끔 실패한다면 대기 시간을 늘리기 전에 스크린샷과 로그부터 아티팩트로 남겨라.
