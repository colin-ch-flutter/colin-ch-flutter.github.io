---
layout: post
title: "Flutter 단위 테스트에서 DateTime 모킹 — package:clock으로 보일러 스케줄 로직 테스트하기"
description: "Flutter 앱에서 DateTime.now()를 직접 호출하면 단위 테스트가 불가능하다. package:clock으로 Clock을 주입하고 withClock으로 시간을 고정하는 패턴과 보일러 예약 스케줄 로직 테스트 실전을 정리했다."
date: 2026-06-28
tags: [Flutter, Dart, CleanArchitecture, IoT, 스마트홈]
comments: true
share: true
---

![Flutter 단위 테스트 DateTime 모킹 시계 개념](https://images.unsplash.com/photo-1508057198894-247b23fe5ade?w=800&q=80)

보일러 스케줄 기능을 만들다가 단위 테스트에서 막혔다. `DateTime.now()`를 직접 호출하는 코드는 테스트에서 시간을 고정할 방법이 없어서 "오전 7시에 자동 켜짐" 같은 로직을 검증하는 게 사실상 불가능하다. 해결책은 `package:clock`으로 Clock을 주입하는 것이다.

## 왜 DateTime.now()가 테스트를 망치는가

보일러 앱에는 이런 로직이 있다.

```dart
bool shouldTurnOn(BoilerSchedule schedule) {
  final now = DateTime.now();
  return now.hour == schedule.onHour && now.minute == schedule.onMinute;
}
```

테스트를 어떻게 짤 것인가. `now`가 테스트 실행 순간의 실제 시각이라 예측이 불가능하다. `schedule.onHour = DateTime.now().hour`로 현재 시각에 맞춰 넣으면 되지 않냐고 할 수 있는데, 그러면 로직을 테스트하는 게 아니라 현재 시각을 두 번 읽는 셈이다. 시분이 바뀌는 경계에서 플래키 테스트가 된다.

처음엔 `Mockito`로 `DateTime`을 모킹하려 했다. Dart에서 `DateTime`은 final class라 모킹 자체가 안 된다. 한 시간을 낭비하고 나서야 `package:clock`이 정답임을 알았다.

## package:clock 주입 패턴

`pubspec.yaml`에 추가:

```yaml
dependencies:
  clock: ^1.1.1

dev_dependencies:
  fake_async: ^1.3.1
```

Clock을 주입받도록 코드를 수정한다. 핵심은 `DateTime.now()` 대신 `clock.now()`를 쓰는 것이다.

```dart
import 'package:clock/clock.dart';

class BoilerScheduleService {
  final Clock _clock;

  BoilerScheduleService({Clock? clock}) : _clock = clock ?? const Clock();

  bool shouldTurnOn(BoilerSchedule schedule) {
    final now = _clock.now();
    return now.hour == schedule.onHour && now.minute == schedule.onMinute;
  }

  bool isWithinActiveWindow(BoilerSchedule schedule) {
    final now = _clock.now();
    final currentMinutes = now.hour * 60 + now.minute;
    final onMinutes = schedule.onHour * 60 + schedule.onMinute;
    final offMinutes = schedule.offHour * 60 + schedule.offMinute;

    if (onMinutes <= offMinutes) {
      return currentMinutes >= onMinutes && currentMinutes < offMinutes;
    }
    // 자정 넘어가는 케이스 (23:00 ~ 06:00)
    return currentMinutes >= onMinutes || currentMinutes < offMinutes;
  }
}
```

생성자에서 `Clock?`을 받고 기본값은 `const Clock()`으로 둔다. 프로덕션 코드는 아무것도 안 바꿔도 되고, 테스트에서만 가짜 시계를 꽂을 수 있다.

## withClock으로 시간 고정

`withClock`은 `package:clock`이 제공하는 유틸리티다. 블록 안에서 `clock.now()`를 호출하면 내가 지정한 시각이 반환된다.

```dart
import 'package:clock/clock.dart';
import 'package:flutter_test/flutter_test.dart';

void main() {
  group('BoilerScheduleService', () {
    late BoilerScheduleService service;

    setUp(() {
      service = BoilerScheduleService();
    });

    test('오전 7시 정각에 shouldTurnOn이 true를 반환한다', () {
      final schedule = BoilerSchedule(onHour: 7, onMinute: 0, offHour: 9, offMinute: 0);

      withClock(Clock.fixed(DateTime(2026, 6, 28, 7, 0)), () {
        expect(service.shouldTurnOn(schedule), isTrue);
      });
    });

    test('7시 1분에는 shouldTurnOn이 false다', () {
      final schedule = BoilerSchedule(onHour: 7, onMinute: 0, offHour: 9, offMinute: 0);

      withClock(Clock.fixed(DateTime(2026, 6, 28, 7, 1)), () {
        expect(service.shouldTurnOn(schedule), isFalse);
      });
    });

    test('활성 구간(07:00~09:00) 안에 있으면 isWithinActiveWindow가 true다', () {
      final schedule = BoilerSchedule(onHour: 7, onMinute: 0, offHour: 9, offMinute: 0);

      withClock(Clock.fixed(DateTime(2026, 6, 28, 8, 30)), () {
        expect(service.isWithinActiveWindow(schedule), isTrue);
      });
    });

    test('자정 넘어가는 스케줄(23:00~06:00)에서 새벽 2시는 활성 구간이다', () {
      final schedule = BoilerSchedule(onHour: 23, onMinute: 0, offHour: 6, offMinute: 0);

      withClock(Clock.fixed(DateTime(2026, 6, 28, 2, 0)), () {
        expect(service.isWithinActiveWindow(schedule), isTrue);
      });
    });
  });
}
```

![package:clock withClock 테스트 패턴 다이어그램](https://images.unsplash.com/photo-1551288049-bebda4e38f71?w=800&q=80)

자정 넘어가는 케이스는 만들어놓지 않았다가 실제 사용자가 "23시 켜짐, 6시 꺼짐" 스케줄을 설정하고 나서야 버그가 터진 경험이 있다. 이걸 테스트로 잡을 수 있다는 게 Clock 주입의 진짜 가치다.

## fakeAsync와 결합 — 타이머 기반 스케줄러 테스트

1분마다 스케줄을 체크하는 타이머 로직이 있다면 `fakeAsync`와 결합한다. [이전 편에서 다룬 fakeAsync]({% post_url 2026-06-26-14-00-00-777735-flutter-fake-async-stream-timer-unit-test %})와 `withClock`을 같이 쓸 수 있다.

```dart
import 'package:fake_async/fake_async.dart';

test('1분마다 스케줄 체크 — 7시 도달 시 보일러 켜짐 이벤트 발생', () {
  final events = <String>[];
  final schedule = BoilerSchedule(onHour: 7, onMinute: 0, offHour: 9, offMinute: 0);

  // 06:59에서 시작
  withClock(Clock(() => DateTime(2026, 6, 28, 6, 59)), () {
    fakeAsync((async) {
      final scheduler = BoilerScheduler(schedule: schedule, onTurnOn: () => events.add('ON'));
      scheduler.start();

      // 1분 경과 — 이 시점에 clock도 7:00으로 움직여야 함
      async.elapse(const Duration(minutes: 1));

      expect(events, contains('ON'));
      scheduler.stop();
    });
  });
});
```

근데 여기서 함정이 있다. `withClock`에 `Clock.fixed`를 쓰면 시간이 고정되어서 `fakeAsync`로 시간을 흘려도 `clock.now()`는 여전히 처음 값을 반환한다. `Clock(() => DateTime(...) + async.elapsed)` 형태로 elapsed와 연동해야 한다. 처음 이걸 몰라서 타이머 테스트가 전부 엉망이 됐다.

실용적인 해결책은 타이머 콜백에서 직접 `clock.now()`를 읽는 게 아니라, 테스트에서 원하는 시각에 해당하는 Clock을 블록마다 다르게 넘기는 것이다.

```dart
test('07:00 tick에서 켜짐, 09:00 tick에서 꺼짐', () {
  final service = BoilerScheduleService();
  final schedule = BoilerSchedule(onHour: 7, onMinute: 0, offHour: 9, offMinute: 0);

  // 각 시각을 독립적으로 검증
  withClock(Clock.fixed(DateTime(2026, 6, 28, 6, 59)), () {
    expect(service.isWithinActiveWindow(schedule), isFalse);
  });

  withClock(Clock.fixed(DateTime(2026, 6, 28, 7, 0)), () {
    expect(service.isWithinActiveWindow(schedule), isTrue);
  });

  withClock(Clock.fixed(DateTime(2026, 6, 28, 9, 0)), () {
    expect(service.isWithinActiveWindow(schedule), isFalse);
  });
}); 
```

이 방식이 타이머 통합보다 훨씬 명확하다. 타이머 자체는 별도 테스트에서 `fakeAsync`로만 검증하고, 시각 판단 로직은 `withClock`으로만 검증하는 게 책임이 깔끔하게 분리된다.

## 삽질 포인트

**`const Clock()` vs `Clock.fixed()`**

생성자 기본값으로 `const Clock()`을 쓰면 프로덕션에서 실제 시각을 반환한다. `Clock.fixed(someDateTime)`을 쓰면 항상 그 시각을 반환한다. 테스트에서 `BoilerScheduleService()`로 만들면 기본 Clock이 들어가서 여전히 실제 시각이 된다. 반드시 `withClock`으로 감싸야 Clock이 바뀐다.

**`withClock` 범위 밖에서 `clock.now()` 호출**

`withClock` 블록 안에서만 가짜 시각이 적용된다. 블록이 끝나면 원래 시계로 돌아간다. 비동기 코드가 블록 바깥에서 실행되면 가짜 시각이 적용 안 된다. `async/await` 체인이 길면 `withClock` 범위를 잘 확인해야 한다.

**iOS 시뮬레이터 타임존**

실기기와 시뮬레이터의 타임존이 다를 때 골든 시간 테스트가 실패했다. `DateTime.now()`는 로컬 타임존, `DateTime.now().toUtc()`는 UTC다. 스케줄 저장 형식을 UTC로 통일하고 비교도 UTC로 맞추니 해결됐다. `Clock` 주입만큼 중요한 부분인데 문서에는 없다.

---

[스케줄 UI 구현]({% post_url 2026-05-27-10-16-04-345660-temperature-schedule-ui-time-based-auto-control %})에서 만든 화면의 로직을 이제 온전히 테스트할 수 있다. 다음 편은 테스트 코드 자체의 품질 관리 — 테스트 리뷰에서 자주 나오는 안티패턴 정리다.

**참고 링크**
- [package:clock 공식 문서](https://pub.dev/packages/clock)
- [Controlling time in Dart unit tests](https://iiro.dev/controlling-time-with-package-clock/)
- [Testing Time-dependent Code in Flutter/Dart](https://tomasrepcik.dev/blog/2022/2022-11-10-time-dependent-coding-copy/)
