---
layout: post
title: "Flutter 단위 테스트 GitHub Actions CI 자동화 — 커밋마다 커버리지 측정하기"
description: "flutter test --coverage를 GitHub Actions에 연동해 PR마다 커버리지 리포트를 자동 생성하는 방법. ubuntu 러너 Flutter 세팅, lcov 리포트 아티팩트 업로드, generated 파일 제외까지 실제 workflow 파일로 정리했다."
date: 2026-06-24
tags: [Flutter, UnitTest, CI/CD, GitHub Actions, Dart]
comments: true
share: true
---

# Flutter 단위 테스트 GitHub Actions CI 자동화 — 커밋마다 커버리지 측정하기

![터미널에서 실행되는 자동화 CI 파이프라인](https://images.unsplash.com/photo-1618401471353-b98afee0b2eb?w=800&q=80)

지금까지 Repository 테스트, Controller 비동기 테스트, [BLE 격리]({% post_url 2026-06-24-10-00-00-654285-flutter-blue-plus-ble-unit-test-wrapper-isolation %}), [MQTT 격리]({% post_url 2026-06-24-14-00-00-666630-mqtt5-client-unit-test-fake-service-isolation %})까지 작성했다. 근데 문제가 있다. 로컬에서만 `flutter test`를 돌리면 팀원이 PR 올릴 때 테스트를 안 돌려도 아무도 모른다. 실제로 한 번은 BLE Wrapper를 리팩터하면서 테스트를 깨뜨렸는데, 머지 직전에야 알았다.

GitHub Actions로 연동하면 커밋마다 테스트가 자동으로 돌고, 커버리지까지 숫자로 확인할 수 있다.

## 왜 로컬 테스트만으로는 부족한가

혼자 작업할 때도 문제가 생긴다. `mqtt5_client` FakeMqttService를 수정하다가 인터페이스 시그니처를 바꿨는데, 관련 Controller 테스트가 컴파일 에러 없이 그냥 통과했다. 이유가 뭔지 추적하다 보니 테스트 파일이 오래된 버전의 Fake를 임포트하고 있었다.

CI가 있었으면 PR 체크 단계에서 걸렸을 문제다. 로컬에선 "방금 고친 거니까 되겠지"하고 넘기게 된다.

## 기본 워크플로 구조

`.github/workflows/test.yml` 파일 하나로 시작한다. 핵심은 세 단계다: Flutter 세팅 → 테스트 실행 → 커버리지 업로드.

```yaml
name: Flutter Test

on:
  push:
    branches: [master, main, develop]
  pull_request:
    branches: [master, main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.32.0'
          channel: 'stable'
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Generate mocks
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Run tests with coverage
        run: flutter test --coverage

      - name: Upload coverage artifact
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/lcov.info
```

`subosito/flutter-action@v2`가 Flutter SDK 설치를 처리해준다. `cache: true`를 빼먹으면 매번 SDK 다운로드로 2~3분이 추가된다.

## Mockito 코드 생성 처리

[Controller 테스트에서 Mockito를 쓴다면]({% post_url 2026-06-23-11-07-13-641940-flutter-getx-test-async-dialog-confirm %}) `build_runner`가 필수다. CI에서 `.g.dart` 파일을 생성하지 않으면 컴파일 에러로 테스트 자체가 안 돌아간다.

```yaml
- name: Generate mocks
  run: dart run build_runner build --delete-conflicting-outputs
```

`--delete-conflicting-outputs`는 로컬에서 생성한 `.g.dart`와 충돌을 막는다. 이게 없으면 CI에서 "output already exists" 에러가 난다. 처음에 이거 빠뜨리고 30분 날렸다.

## lcov 리포트로 커버리지 시각화

`flutter test --coverage` 실행 후 `coverage/lcov.info` 파일이 생긴다. 이걸 HTML로 변환하면 어떤 줄이 테스트됐는지 라인별로 볼 수 있다.

```yaml
      - name: Install lcov
        run: sudo apt-get install -y lcov

      - name: Remove generated files from coverage
        run: |
          lcov --remove coverage/lcov.info \
            '*/lib/*.g.dart' \
            '*/lib/*.freezed.dart' \
            '**/*.gen.dart' \
            -o coverage/lcov.info

      - name: Generate HTML report
        run: genhtml coverage/lcov.info -o coverage/html

      - name: Upload HTML report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-html
          path: coverage/html
```

![lcov HTML 커버리지 리포트 예시](https://images.unsplash.com/photo-1555949963-aa79dcee981c?w=800&q=80)

`lcov --remove`로 자동생성 파일을 제외하는 단계가 중요하다. `.g.dart`, `.freezed.dart`는 테스트하지 않아도 되는 코드인데, 커버리지 숫자를 깎아먹는다. 이걸 안 빼면 커버리지가 실제보다 20~30% 낮게 나온다.

## 커버리지 임계값 설정

PR을 머지할 때 커버리지가 특정 수치 이하면 실패시키고 싶다면 추가한다.

```yaml
      - name: Check coverage threshold
        run: |
          COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep "lines" | grep -oP '\d+\.\d+(?=%)')
          echo "Coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 70" | bc -l) )); then
            echo "Coverage ${COVERAGE}% is below threshold 70%"
            exit 1
          fi
```

숫자는 프로젝트 상황에 맞게 잡는다. IoT 앱처럼 외부 하드웨어 의존도가 높으면 70~75%가 현실적이다. 100%를 목표로 하면 의미 없는 테스트만 늘어난다.

## PR 코멘트에 커버리지 표시

PR마다 커버리지 숫자를 코멘트로 달아주면 편하다.

```yaml
      - name: Report LCOV
        uses: zgosalvez/github-actions-report-lcov@v4
        with:
          coverage-files: coverage/lcov.info
          minimum-coverage: 70
          github-token: ${{ secrets.GITHUB_TOKEN }}
          update-comment: true
```

이 액션이 PR에 커버리지 변화 (+2.3% / -1.1%)를 자동으로 코멘트로 달아준다. `update-comment: true`를 쓰면 새 커밋마다 코멘트를 새로 쓰지 않고 기존 코멘트를 업데이트한다.

## 삽질 포인트

**ubuntu에서 BLE 테스트가 있을 때:** ubuntu 러너에는 Bluetooth 스택이 없다. [BLE Wrapper 패턴]({% post_url 2026-06-24-10-00-00-654285-flutter-blue-plus-ble-unit-test-wrapper-isolation %})으로 격리한 테스트는 ubuntu에서도 돌아가지만, `flutter_blue_plus`를 직접 호출하는 코드가 테스트에 남아있으면 플랫폼 채널 에러가 난다. 격리가 잘 돼 있는지 CI에서 먼저 검증되는 셈이다.

**`dart run build_runner`가 느릴 때:** CI 캐시를 활용한다.

```yaml
      - name: Cache build_runner output
        uses: actions/cache@v4
        with:
          path: |
            .dart_tool/build
            **/*.g.dart
          key: ${{ runner.os }}-build-${{ hashFiles('pubspec.lock') }}
```

**`flutter test`가 특정 디바이스 없이 실패할 때:** `flutter test`는 기본적으로 VM에서 실행된다. `-d` 옵션 없이 실행하면 된다. 위젯 테스트가 있으면 `--platform chrome`이 필요한 경우도 있다.

## 완성된 workflow 파일

```yaml
name: Flutter Test & Coverage

on:
  push:
    branches: [master, main, develop]
  pull_request:
    branches: [master, main]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          flutter-version: '3.32.0'
          channel: 'stable'
          cache: true

      - name: Install dependencies
        run: flutter pub get

      - name: Generate mocks
        run: dart run build_runner build --delete-conflicting-outputs

      - name: Run tests with coverage
        run: flutter test --coverage

      - name: Install lcov
        run: sudo apt-get install -y lcov

      - name: Remove generated files from coverage
        run: |
          lcov --remove coverage/lcov.info \
            '*/lib/*.g.dart' \
            '*/lib/*.freezed.dart' \
            '**/*.gen.dart' \
            -o coverage/lcov.info

      - name: Generate HTML report
        run: genhtml coverage/lcov.info -o coverage/html

      - name: Report LCOV
        uses: zgosalvez/github-actions-report-lcov@v4
        with:
          coverage-files: coverage/lcov.info
          minimum-coverage: 70
          github-token: ${{ secrets.GITHUB_TOKEN }}
          update-comment: true

      - name: Upload HTML report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-html
          path: coverage/html
```

이 파일 하나를 `.github/workflows/test.yml`에 넣으면 PR마다 테스트가 돌고 커버리지가 코멘트로 달린다.

---

다음 편은 Widget Test — 화면 단위 UI 테스트를 실제 IoT 제어 화면에 적용하는 내용이다.

**참고 링크**
- [subosito/flutter-action GitHub](https://github.com/subosito/flutter-action)
- [zgosalvez/github-actions-report-lcov](https://github.com/zgosalvez/github-actions-report-lcov)
- [Flutter 공식 테스트 문서](https://docs.flutter.dev/testing)
