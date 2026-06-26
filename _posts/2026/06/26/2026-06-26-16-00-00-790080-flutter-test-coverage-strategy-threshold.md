---
layout: post
title: "Flutter 테스트 커버리지 전략 — 레이어별 목표 설정과 CI 강제"
description: "Flutter IoT 앱에서 테스트 커버리지를 레이어별로 어떻게 설정하는지, 커버리지 숫자가 높아도 품질이 낮은 상황과 CI에서 임계값을 강제하는 방법을 실제 경험으로 정리했다."
date: 2026-06-26
tags: [Flutter, Dart, CleanArchitecture, CI/CD, GitHub Actions]
comments: true
share: true
---

# Flutter 테스트 커버리지 전략 — 레이어별 목표 설정과 CI 강제

![Flutter 테스트 커버리지 리포트 화면](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

Flutter에서 테스트 커버리지 목표를 설정하려면 레이어별로 다른 기준을 적용해야 한다. Domain 레이어는 90% 이상, Repository는 80% 이상, ViewModel/Controller는 70% 이상, Widget은 50% 수준이 현실적인 출발점이다. 모든 레이어에 동일한 기준을 적용하면 BLE/MQTT 연동 코드처럼 외부 의존성이 큰 부분에서 의미 없는 mock 테스트만 양산하게 된다.

처음엔 전체 커버리지 80%만 넘기면 충분한 줄 알았다. 그런데 BLE 재연결 로직에서 버그가 나왔을 때 커버리지는 84%였다. 테스트는 통과했고, 숫자도 높았는데 실기기에서 재연결이 안 됐다. 그때부터 커버리지 숫자보다 어떤 코드를 커버하는지가 중요하다는 걸 깨달았다.

## 레이어별 커버리지 목표를 왜 다르게 잡아야 하는가

Clean Architecture로 분리된 Flutter 프로젝트에서 레이어마다 테스트 가능성이 다르다.

**Domain 레이어 (UseCase, Entity): 90%+**

순수 Dart 로직만 있어서 외부 의존성이 없다. `flutter_blue_plus`도, `mqtt_client`도 없다. `usecase.execute()` 호출 하나로 비즈니스 규칙 전체를 검증할 수 있다. 90% 못 넘기면 뭔가 테스트를 안 만든 거다.

**Repository 레이어: 80%+**

DataSource를 통해 외부와 연결되는 지점이다. Fake나 Mock으로 DataSource를 교체하면 테스트 가능하다. [이전 편]({% post_url 2026-06-26-12-00-00-765390-flutter-test-doubles-fake-mock-stub-spy-patterns %})에서 다룬 Test Double 패턴을 여기서 쓴다. 에러 처리 분기(`Either<Failure, T>`)가 많아서 80%도 쉽지 않다.

**ViewModel / Controller 레이어: 70%+**

상태 전이 로직을 검증하는 게 핵심이다. UI 이벤트 → 상태 변화 → UseCase 호출 흐름을 테스트한다. [TDD로 IoT Controller를 설계한 편]({% post_url 2026-06-26-10-00-00-753045-flutter-tdd-iot-controller-design %})에서 만든 패턴이 여기 해당한다. 70%면 주요 상태 전이는 다 커버하는 수준이다.

**Widget 레이어: 50%+**

솔직히 말하면 Widget 테스트는 유지 비용이 높다. UI가 바뀔 때마다 테스트가 깨진다. 50% 목표도 사실 "핵심 인터랙션만 커버하자"는 현실적 타협이다. 연결 버튼 탭 → 로딩 상태 표시 같은 중요 흐름만 커버한다.

## `flutter test --coverage` 실행과 lcov 리포트 보는 법

```bash
# 커버리지 측정 실행 (Flutter 3.24 기준)
flutter test --coverage

# 결과: coverage/lcov.info 생성됨
# 사람이 읽기 좋은 HTML 리포트로 변환 (lcov 설치 필요)
genhtml coverage/lcov.info -o coverage/html

# macOS에서 lcov 설치
brew install lcov

# 리포트 열기
open coverage/html/index.html
```

`lcov.info` 파일은 각 파일별로 어느 줄이 실행됐는지 기록된다. 형식은 이렇다.

```
SF:lib/domain/usecases/connect_device_usecase.dart
DA:12,1
DA:13,1
DA:18,0   ← 이 줄은 테스트에서 한 번도 실행 안 됨
LH:2
LF:3
end_of_record
```

`DA:18,0`처럼 두 번째 숫자가 0인 줄이 미커버 라인이다. 에러 처리 분기가 이렇게 빠져 있으면 실기기에서 예외 던질 때 대비가 안 된 거다.

## CI에서 커버리지 임계값을 강제하는 방법

![코드 커버리지 측정 도구](https://images.unsplash.com/photo-1504639725590-34d0984388bd?w=800&q=80)

[GitHub Actions CI 설정 편]({% post_url 2026-06-24-16-00-00-678975-flutter-unit-test-github-actions-ci-coverage %})에서 기본 CI 파이프라인을 구성했다면, 여기에 커버리지 임계값 체크를 추가한다.

Flutter는 `--minimum-coverage` 플래그를 직접 제공하지 않는다. `lcov`의 `--fail-under-percent` 옵션을 이용해 CI를 실패시킨다.

```yaml
# .github/workflows/test.yml
- name: Run tests with coverage
  run: flutter test --coverage

- name: Check coverage threshold
  run: |
    COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep "lines" | grep -oP '\d+\.\d+(?=%)')
    echo "Total coverage: $COVERAGE%"
    if (( $(echo "$COVERAGE < 70" | bc -l) )); then
      echo "Coverage $COVERAGE% is below 70% threshold"
      exit 1
    fi
```

레이어별로 임계값을 다르게 적용하려면 `lcov --extract`로 특정 경로만 걸러낸다.

```bash
# Domain 레이어만 추출해서 90% 체크
lcov --extract coverage/lcov.info "*/domain/*" -o coverage/domain.info
genhtml coverage/domain.info -o coverage/domain_html
DOMAIN_COV=$(lcov --summary coverage/domain.info 2>&1 | grep "lines" | grep -oP '\d+\.\d+(?=%)')

if (( $(echo "$DOMAIN_COV < 90" | bc -l) )); then
  echo "Domain coverage $DOMAIN_COV% is below 90%"
  exit 1
fi
```

이 스크립트를 레이어별(domain, repository, viewmodel)로 반복해서 각각 임계값을 체크한다. 처음 이 구조를 만들 때 `--extract` 경로 패턴이 내 프로젝트 구조랑 안 맞아서 한참 헤맸다. `lcov --extract`는 절대 경로 패턴을 요구하는데, CI 환경의 절대 경로가 로컬이랑 달라서 패턴을 `*`로 감싸줘야 한다.

## 커버리지 100%인데 버그가 있었던 경험

근데 여기서 함정이 있다.

BLE 재연결 로직을 담당하는 `BleConnectionRepository` 코드가 있었다. `lcov` 리포트상 커버리지 100%. 테스트 12개, 전부 통과. 근데 Galaxy S24에서 앱을 백그라운드로 내렸다가 다시 올리면 재연결이 안 됐다.

원인을 찾아보니 테스트 코드가 이랬다.

```dart
test('reconnect 호출 시 연결 상태가 된다', () async {
  await repository.reconnect(deviceId);
  expect(repository.connectionState, ConnectionState.connected);
});
```

`reconnect()` 메서드 내부에서 `await Future.delayed(Duration(seconds: 1))`가 있는데, `FakeAsync`를 쓰지 않아서 실제로 딜레이가 실행됐다. [FakeAsync 편]({% post_url 2026-06-26-14-00-00-777735-flutter-fake-async-stream-timer-unit-test %})에서 다룬 것처럼 타이머 제어가 없으면 실제 타이밍 의존 버그를 못 잡는다.

더 큰 문제는 `reconnect()`가 내부적으로 Android의 `autoConnect` 플래그를 `true`로 설정해야 하는데, 테스트에선 `FakeBleDataSource`를 썼고 그 Fake가 해당 플래그를 무시했다. 코드 라인은 실행됐으니 커버됐다고 표시됐지만, 실제 동작을 검증한 게 아니었다.

커버리지가 측정하는 건 "코드가 실행됐는가"이지 "올바르게 동작했는가"가 아니다.

```dart
// 이건 커버리지에 잡힌다
test('reconnect 호출된다', () async {
  await repository.reconnect(deviceId); // 이 라인이 실행되면 커버됨
  // assert가 없어도!
});

// 이게 진짜 테스트다
test('reconnect 시 autoConnect 플래그가 true로 설정된다', () async {
  await repository.reconnect(deviceId);
  expect(fakeBleDataSource.lastAutoConnectFlag, isTrue); // 실제 동작 검증
});
```

## IoT 앱에서 BLE/MQTT 레이어는 커버리지보다 통합 테스트가 낫다

BLE와 MQTT는 외부 하드웨어·브로커와 통신하는 레이어다. 이 부분을 unit test로 100% 커버하려는 건 의미 없는 Mock 테스트만 늘리는 결과로 이어진다.

```dart
// 이런 테스트는 커버리지는 올라가는데 가치가 없다
test('BLE connect 호출 시 mock이 connect를 반환한다', () async {
  when(mockBle.connect(any)).thenAnswer((_) => Stream.value(BleConnectionState.connected));
  final result = await repository.connect('device-001');
  expect(result, isRight);
  verify(mockBle.connect('device-001')).called(1);
});
```

이 테스트는 Mock 설정이 맞는지를 검증하는 거지, 실제 BLE 연결 로직을 검증하는 게 아니다. BLE 레이어는 실기기 통합 테스트나 [Widget Test + CI Coverage 편]({% post_url 2026-06-25-14-00-00-703665-flutter-widget-test-ci-coverage-merge %})에서 다룬 통합 레벨 검증이 더 실용적이다.

BLE/MQTT DataSource는 커버리지 임계값 체크에서 제외하는 게 낫다.

```bash
# BLE, MQTT DataSource는 커버리지 체크 대상에서 제거
lcov --remove coverage/lcov.info \
  "*/datasources/ble*" \
  "*/datasources/mqtt*" \
  -o coverage/filtered.info
```

이렇게 하면 커버리지 숫자가 일시적으로 높아 보이지만, 실제로 검증 가능한 코드만 측정하는 거라 의미 있는 숫자가 된다.

## 커버리지 설정 정리

```bash
# 전체 커버리지 측정
flutter test --coverage

# BLE/MQTT 제외한 커버리지
lcov --remove coverage/lcov.info "*/datasources/ble*" "*/datasources/mqtt*" \
  -o coverage/filtered.info

# 레이어별 임계값 체크 (CI 스크립트)
check_coverage() {
  local layer=$1
  local threshold=$2
  local pattern=$3

  lcov --extract coverage/filtered.info "$pattern" -o coverage/${layer}.info
  local cov=$(lcov --summary coverage/${layer}.info 2>&1 | grep "lines" | grep -oP '\d+\.\d+(?=%)')
  echo "$layer coverage: $cov%"

  if (( $(echo "$cov < $threshold" | bc -l) )); then
    echo "FAIL: $layer coverage $cov% < $threshold%"
    exit 1
  fi
}

check_coverage "domain"     90 "*/domain/*"
check_coverage "repository" 80 "*/repositories/*"
check_coverage "viewmodel"  70 "*/viewmodels/*"
```

이 스크립트를 `.github/workflows/test.yml`의 커버리지 체크 단계에 넣으면 레이어별 임계값이 CI에서 강제된다.

---

커버리지는 테스트 품질을 보조하는 지표지, 그 자체가 목표가 아니다. 숫자를 높이는 데 집중하면 의미 없는 테스트가 쌓이고, BLE 재연결 버그처럼 실제 동작은 검증이 안 된 채로 숫자만 좋아 보이는 상태가 된다. Domain 레이어는 높게 잡고, 외부 의존성이 큰 레이어는 임계값 대상에서 빼는 게 현실적인 전략이다.

다음 편은 Flutter Riverpod 기반 상태 관리에서 ProviderContainer를 활용한 unit test 패턴 — Provider 의존성을 override하는 방법과 StateNotifier 상태 전이를 테스트하는 방법에 대한 내용이다.

**참고 링크**
- [lcov 공식 문서](https://lcov.readthedocs.io/en/latest/)
- [Flutter 공식 테스트 문서](https://docs.flutter.dev/testing)
- [Codecov GitHub Actions 연동](https://docs.codecov.com/docs/github-actions-getting-started)
