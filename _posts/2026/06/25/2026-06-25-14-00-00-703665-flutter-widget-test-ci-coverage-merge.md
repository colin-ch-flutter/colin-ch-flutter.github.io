---
layout: post
title: "Flutter Widget Test CI 통합 — Unit·Widget 커버리지 합산 GitHub Actions"
description: "기존 Unit Test CI에 Widget Test를 추가하고 lcov 커버리지를 합산하는 방법. Linux 러너에서 Widget Test가 되는 이유, coverage/lcov.info 충돌 해결, PR 커버리지 뱃지 설정까지 실제 workflow 파일과 함께 정리했다."
date: 2026-06-25
tags: [Flutter, CI/CD, GitHub Actions, UnitTest, Dart]
comments: true
share: true
---

# Flutter Widget Test CI 통합 — Unit·Widget 커버리지 합산 GitHub Actions

![GitHub Actions CI 자동화 파이프라인](https://images.unsplash.com/photo-1607706189992-eae578626c86?w=800&q=80)

[이전 글에서 Widget Test를 작성했다.]({% post_url 2026-06-25-10-00-00-691320-flutter-widget-test-iot-control-screen %}) 로컬에서 돌리면 잘 된다. 근데 CI에 안 올리면 반쪽짜리다. PR 올릴 때 Widget Test가 자동으로 돌고, Unit Test와 커버리지가 합산돼서 하나의 숫자로 나와야 진짜다.

[Unit Test CI]({% post_url 2026-06-24-16-00-00-678975-flutter-unit-test-github-actions-ci-coverage %})를 구축할 때 `flutter test --coverage`를 쓰면 `coverage/lcov.info`가 생긴다고 했다. Widget Test를 추가하면 이 파일이 덮어씌워지는 문제가 생긴다. 합산하는 방법이 따로 있다.

## Widget Test, Linux 러너에서 된다

처음에 "Widget Test는 렌더링이 있으니까 macOS 러너가 필요하겠지"라고 생각했다. 아니다. Flutter Widget Test는 가상 렌더러(`flutter_test` 패키지의 `TestFlutterView`)를 쓰기 때문에 GPU가 없어도 된다. Linux ubuntu-latest 러너에서 그냥 된다.

macOS 러너는 분당 비용이 ubuntu의 10배다. Widget Test를 위해 macOS 러너를 쓸 이유가 없다. iOS 기기 연동 Integration Test(실제 시뮬레이터 띄우는 것)를 할 때만 macOS가 필요하다.

## 문제: lcov.info 파일이 덮어씌워진다

`flutter test`는 Unit Test와 Widget Test를 구분하지 않는다. `test/` 하위 파일을 전부 실행한다. 커버리지 파일도 하나만 생긴다.

그러면 "이미 합산돼 있는 거 아닌가?"라고 생각할 수 있는데, 문제는 **테스트 디렉터리를 분리해서 관리할 때**다. 나는 이렇게 나눠뒀다.

```
test/
  unit/          ← Repository, Controller 테스트
  widget/        ← BoilerControlScreen Widget Test
```

이 구조에서 `flutter test test/unit --coverage`를 먼저 돌리고, 그 다음에 `flutter test test/widget --coverage`를 돌리면 두 번째 실행이 `coverage/lcov.info`를 덮어쓴다. Unit Test 커버리지가 날아간다.

## 해결: `--coverage-path`로 파일 분리 후 합산

`flutter test`에는 `--coverage-path` 옵션이 있다. 커버리지 파일 경로를 지정할 수 있다.

```bash
# Unit Test 커버리지
flutter test test/unit --coverage --coverage-path=coverage/unit.info

# Widget Test 커버리지
flutter test test/widget --coverage --coverage-path=coverage/widget.info

# 합산
lcov --add-tracefile coverage/unit.info \
     --add-tracefile coverage/widget.info \
     -o coverage/lcov.info
```

`lcov --add-tracefile`은 두 파일의 커버리지를 더한다. 같은 파일이 두 쪽 모두에 있으면 히트 카운트를 합산한다.

## 실제 GitHub Actions workflow

기존 `test.yml`에서 확장하는 방식이다.

```yaml
name: Flutter Test

on:
  push:
    branches: [master, main]
  pull_request:
    branches: [master, main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Flutter 세팅
        uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.24.0'
          channel: 'stable'
          cache: true

      - name: 의존성 설치
        run: flutter pub get

      - name: 코드 생성 (build_runner 사용 시)
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Unit Test
        run: flutter test test/unit --coverage --coverage-path=coverage/unit.info

      - name: Widget Test
        run: flutter test test/widget --coverage --coverage-path=coverage/widget.info

      - name: lcov 설치
        run: sudo apt-get install -y lcov

      - name: 커버리지 합산
        run: |
          lcov --add-tracefile coverage/unit.info \
               --add-tracefile coverage/widget.info \
               -o coverage/lcov.info

      - name: generated 파일 제외
        run: |
          lcov --remove coverage/lcov.info \
               '*.g.dart' '*.freezed.dart' '*.gr.dart' \
               -o coverage/lcov.info

      - name: 커버리지 리포트 생성
        run: genhtml coverage/lcov.info -o coverage/html

      - name: 커버리지 아티팩트 업로드
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/html
```

`subosito/flutter-action@v2`의 `cache: true`는 `pub get` 속도를 많이 줄여준다. 첫 번째 실행 이후 캐시가 생기면 Flutter SDK 다운로드를 건너뛴다.

## 삽질: `test/` 루트를 통째로 돌리면 안 되는 경우

처음엔 그냥 `flutter test --coverage` 하나로 다 돌리려 했다. `test/unit`과 `test/widget` 모두 잡힌다. 문제는 Widget Test에서 `GetMaterialApp`을 쓰는데, `MaterialApp`이 없으면 일부 테스트가 context 관련 오류로 죽었다.

```
A RenderFlex overflowed by 26 pixels on the right.
══╡ EXCEPTION CAUGHT BY FLUTTER TEST FRAMEWORK ╞══
```

Unit Test 파일은 Widget 렌더링이 없으니 이 오류가 생길 이유가 없는데, 같이 돌리면 Unit Test 파일이 Widget Test 환경 설정에 영향을 받는 케이스가 있었다. 분리해서 실행하는 게 맞다.

![Flutter 커버리지 리포트 HTML 출력 화면](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

## 커버리지 뱃지 추가

`Codecov`를 연동하면 PR마다 커버리지 퍼센트가 댓글로 달린다. 무료 플랜으로도 Public 레포에서 된다.

```yaml
      - name: Codecov 업로드
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: coverage/lcov.info
          fail_ci_if_error: false
```

`fail_ci_if_error: false`로 해두면 Codecov 서버가 일시적으로 죽어도 CI 자체는 통과한다. Codecov는 부가 기능이지, CI 통과 조건이 되면 안 된다.

`README.md`에 뱃지 추가:
```markdown
[![codecov](https://codecov.io/gh/username/repo/branch/master/graph/badge.svg)](https://codecov.io/gh/username/repo)
```

## 현재 시리즈 커버리지 현황

IoT 스마트홈 앱 기준으로 지금까지 쌓은 테스트 현황:

| 레이어 | 테스트 종류 | 현재 커버리지 |
|--------|-------------|--------------|
| Repository | Unit Test (Fake) | 약 85% |
| Controller | Unit Test (비동기) | 약 72% |
| BLE 서비스 | Unit Test (Wrapper) | 약 68% |
| MQTT 서비스 | Unit Test (Fake) | 약 70% |
| 제어 화면 | Widget Test | 약 60% |

100%는 목표가 아니다. 비즈니스 로직과 상태 변화 경로가 커버되면 충분하다. `Obx` 안의 렌더링 조건, 버튼 활성화 조건 — 이게 눈에 안 보이게 깨지는 부분이라서 Widget Test가 유효한 거다.

CI에 올라가면 이 숫자가 PR마다 보인다. 떨어지면 리뷰어가 바로 안다.

---

Unit Test와 Widget Test를 같은 CI 파이프라인에서 돌리고, lcov로 합산하면 진짜 의미 있는 커버리지 숫자가 나온다. 시리즈 2는 여기까지다. 다음 시리즈는 Flutter 앱에서 Integration Test를 실기기 없이 돌리는 방법 — Firebase Test Lab 연동을 다룰 예정이다.

**참고 링크**
- [Flutter 공식 테스트 가이드](https://docs.flutter.dev/testing/overview)
- [subosito/flutter-action GitHub](https://github.com/subosito/flutter-action)
- [Codecov GitHub Actions](https://github.com/codecov/codecov-action)
- [lcov 공식 문서](https://github.com/linux-test-project/lcov)
