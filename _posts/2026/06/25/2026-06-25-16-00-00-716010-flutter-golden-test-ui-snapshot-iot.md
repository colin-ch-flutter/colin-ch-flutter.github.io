---
layout: post
title: "Flutter Golden Test — IoT 대시보드 UI 스냅샷으로 화면 깨짐 잡기"
description: "matchesGoldenFile()로 IoT 제어 카드의 픽셀 단위 UI를 스냅샷 테스트하는 방법. CI 환경에서 폰트·DPR 차이로 골든 파일이 안 맞는 문제와 alchemist 패키지로 해결한 과정을 정리했다."
date: 2026-06-25
tags: [Flutter, Dart, IoT, CI/CD, 스마트홈]
comments: true
share: true
---

![Flutter UI 스냅샷 테스트 골든 파일](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

Widget Test를 도입하고 나서 생긴 착각이 있었다. "이제 UI 버그도 다 잡힌다"는 생각. 근데 그게 아니었다.

Widget Test가 잡는 건 텍스트가 있는지, 버튼이 활성화됐는지 같은 **논리 상태**다. 온도 표시 카드의 색깔이 바뀌거나, BLE 상태 인디케이터 아이콘 크기가 뒤틀리거나, 다크모드에서 텍스트가 배경에 묻히는 — 이런 **픽셀 단위 회귀**는 Widget Test가 전혀 못 잡는다. 실제로 다크모드 작업하다가 보일러 카드 텍스트 색이 의도치 않게 바뀐 걸 배포 직전 수동 확인으로 발견했다. 골든 테스트가 있었으면 PR에서 바로 잡혔을 일이다.

## Golden Test가 하는 것

Flutter의 `matchesGoldenFile()` 매처는 위젯을 이미지로 렌더링해서 **기준 파일(golden file)** 과 픽셀 단위로 비교한다. 기준 파일과 다르면 테스트가 실패하고, diff 이미지를 생성해서 어디가 달라졌는지 보여준다.

```dart
testWidgets('보일러 카드 — 연결됨 상태 골든', (tester) async {
  await tester.pumpWidget(
    MaterialApp(home: BoilerStatusCard(isConnected: true, temperature: 72)),
  );
  await expectLater(
    find.byType(BoilerStatusCard),
    matchesGoldenFile('goldens/boiler_card_connected.png'),
  );
});
```

처음 실행할 때는 `--update-goldens` 플래그를 줘서 기준 이미지를 생성한다.

```bash
flutter test --update-goldens test/widget/boiler_card_test.dart
```

이후에는 `--update-goldens` 없이 그냥 `flutter test`를 돌리면 기준 이미지와 비교한다. 디자인을 의도적으로 바꿨을 때만 `--update-goldens`로 기준을 갱신하고 변경된 기준 파일을 PR에 같이 커밋한다.

## 문제는 CI에서 바로 터진다

로컬(M1 Mac)에서 기준 파일 생성하고 GitHub Actions(Ubuntu)에서 테스트 돌렸더니 바로 실패했다.

```
Golden "goldens/boiler_card_connected.png": Pixel difference found.
Percent different: 3.2%
```

원인 두 가지:
1. **폰트 렌더링 차이** — macOS는 CoreText, Linux는 FreeType. 같은 폰트도 픽셀 단위로 다르게 그린다.
2. **Device Pixel Ratio(DPR) 차이** — 테스트 환경마다 DPR이 달라서 이미지 크기 자체가 달라진다.

이걸 로컬에서 `--update-goldens`로 기준 만들고 CI에서 비교하면 항상 실패한다. 폰트 번들 + DPR 고정으로 해결할 수 있는데, 그 처리를 직접 하려면 생각보다 손이 많이 간다.

## alchemist로 해결하기

[alchemist](https://pub.dev/packages/alchemist) 패키지가 이 문제를 깔끔하게 처리해준다. Very Good Ventures에서 만든 패키지고, **CI 모드와 로컬 모드를 자동으로 분리**해서 환경 차이 문제를 처음부터 막는다.

```yaml
# pubspec.yaml
dev_dependencies:
  alchemist: ^0.9.0
  flutter_test:
    sdk: flutter
```

alchemist의 핵심 아이디어: 로컬에서는 비교를 건너뛰고 생성만 하고, CI에서는 CI 전용 기준 파일과만 비교한다. 환경별 기준 파일을 아예 분리하는 방식이다.

```
test/
  goldens/
    ci/          ← CI(Linux)에서 생성한 기준 파일
      boiler_card_connected.png
    local/       ← 로컬(Mac)에서 생성한 기준 파일 (선택적)
      boiler_card_connected.png
```

CI 환경은 `CI=true` 환경변수로 감지한다. GitHub Actions는 기본으로 `CI=true`를 설정하기 때문에 별도 설정 없이 작동한다.

## IoT 대시보드 카드 골든 테스트

![IoT 대시보드 카드 UI 컴포넌트](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

골든 테스트는 **전체 화면보다 컴포넌트 단위**에서 더 가치 있다. 전체 화면은 바뀌는 요소가 많아서 골든 파일이 자주 깨지고 관리가 힘들어진다.

스마트홈 앱에서 골든 테스트가 유효한 컴포넌트:
- 보일러 상태 카드 (연결됨/끊김, 온도 범위별 색상)
- BLE 신호 강도 인디케이터
- 다크모드 전환 시 UI 일관성

```dart
// test/widget/boiler_card_golden_test.dart
import 'package:alchemist/alchemist.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('BoilerStatusCard 골든 테스트', () {
    goldenTest(
      '연결됨 — 정상 온도',
      fileName: 'boiler_card_connected_normal',
      builder: () => GoldenTestGroup(
        children: [
          GoldenTestScenario(
            name: '72°C',
            child: const BoilerStatusCard(
              isConnected: true,
              temperature: 72,
            ),
          ),
        ],
      ),
    );

    goldenTest(
      '연결됨 — 고온 경고',
      fileName: 'boiler_card_connected_hot',
      builder: () => GoldenTestGroup(
        children: [
          GoldenTestScenario(
            name: '85°C 경고',
            child: const BoilerStatusCard(
              isConnected: true,
              temperature: 85,
            ),
          ),
        ],
      ),
    );

    goldenTest(
      '연결 끊김',
      fileName: 'boiler_card_disconnected',
      builder: () => GoldenTestGroup(
        children: [
          GoldenTestScenario(
            name: 'disconnected',
            child: const BoilerStatusCard(
              isConnected: false,
              temperature: 0,
            ),
          ),
        ],
      ),
    );
  });
}
```

`GoldenTestGroup`과 `GoldenTestScenario`를 쓰면 여러 상태를 **한 이미지에 격자로 렌더링**한다. 상태별로 파일을 분리하는 것보다 보기 편하고 비교도 쉽다.

## alchemist 세팅 — flutter_test_config.dart

```dart
// test/flutter_test_config.dart
import 'dart:async';
import 'package:alchemist/alchemist.dart';

Future<void> testExecutable(FutureOr<void> Function() testMain) async {
  final isRunningInCI = Platform.environment.containsKey('CI');

  return AlchemistConfig.runWithConfig(
    config: AlchemistConfig(
      forceUpdateGoldenFiles: !isRunningInCI, // 로컬에서는 항상 재생성
      platformGoldensConfig: const PlatformGoldensConfig(
        // 로컬 기준 파일은 비교 건너뜀 (환경 차이 때문)
        enabled: !isRunningInCI,
      ),
      ciGoldensConfig: CiGoldensConfig(
        enabled: isRunningInCI,
        obscureText: false,
      ),
    ),
    run: testMain,
  );
}
```

`flutter_test_config.dart`를 `test/` 바로 아래에 두면 해당 디렉토리 아래 모든 테스트 실행 전에 자동으로 적용된다. 테스트마다 개별 설정을 넣을 필요 없다.

## CI 기준 파일 생성 방법

처음 CI 기준 파일을 만들 때는 GitHub Actions를 직접 돌려야 한다.

```yaml
# .github/workflows/golden_update.yml
name: Update Golden Files

on:
  workflow_dispatch: # 수동 트리거

jobs:
  update:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.32.0'
      - run: flutter pub get
      - run: flutter test --update-goldens test/widget/
      - uses: actions/upload-artifact@v4
        with:
          name: golden-files
          path: test/goldens/ci/
```

`workflow_dispatch`로 수동 실행하면 Ubuntu에서 기준 파일을 생성하고 artifact로 올라온다. 그 파일을 내려받아서 `test/goldens/ci/`에 커밋하면 이후엔 일반 CI 플로우에서 비교가 가능해진다.

디자인 변경 시에도 같은 과정: 수동으로 `--update-goldens` 돌리고, 새 기준 파일을 PR에 포함해서 리뷰어가 이미지 diff로 변경 사항을 확인할 수 있게 한다.

## 막혔던 부분 — 폰트가 없어서 텍스트가 네모

```
Warning: A test used "packages/flutter_test/..." font, but couldn't find it.
```

골든 테스트에서 텍스트가 전부 빈 사각형으로 렌더링됐다. Flutter 테스트 환경에서는 기본 폰트가 로드되지 않아서 생기는 문제다.

```dart
// 테스트 시작 전 폰트 로드
setUpAll(() async {
  await loadAppFonts(); // alchemist 제공
});
```

alchemist의 `loadAppFonts()`를 `setUpAll`에 넣으면 해결된다. `pubspec.yaml`에 선언한 폰트 에셋을 테스트 환경에 로드해준다.

폰트 없이 실행하면 텍스트 렌더링 자체가 안 되니까, 처음 골든 테스트 세팅하면 이 에러를 높은 확률로 만난다. 당황하지 말고 `loadAppFonts()` 추가하면 된다.

---

골든 테스트는 도입 허들이 높지 않다. 컴포넌트 하나에 `matchesGoldenFile()` 추가하는 것만으로도 시작할 수 있다. IoT 앱처럼 상태 조합이 많은 UI는 특히 효과가 크다 — 연결 상태 × 온도 범위 × 다크모드 조합을 수동으로 매번 확인하는 건 불가능하다.

[이전 글에서 만든 Widget Test]({% post_url 2026-06-25-10-00-00-691320-flutter-widget-test-iot-control-screen %})에 골든 테스트를 추가하면 논리 검증 + 픽셀 검증이 동시에 돌아간다. [CI 파이프라인]({% post_url 2026-06-24-16-00-00-678975-flutter-unit-test-github-actions-ci-coverage %})에 `flutter test test/widget/`을 이미 넣어놨다면 골든 테스트도 자동으로 포함된다.

다음 편은 Integration Test다. Widget Test + 골든 테스트가 컴포넌트 단위 검증이라면, Integration Test는 실제 사용자 흐름 — BLE 연결부터 보일러 제어까지 — 을 end-to-end로 검증한다. `integration_test` 패키지와 Patrol로 실기기에서 자동화하는 내용이다.

**참고 링크**
- [alchemist 패키지](https://pub.dev/packages/alchemist)
- [Flutter 공식 Golden Test 가이드](https://api.flutter.dev/flutter/flutter_test/matchesGoldenFile.html)
- [Flutter Testing Overview](https://docs.flutter.dev/testing/overview)
