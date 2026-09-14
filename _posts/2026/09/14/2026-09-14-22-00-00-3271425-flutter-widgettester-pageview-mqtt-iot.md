---
layout: post
title: "Flutter WidgetTester PageView 테스트 - MQTT 기기 모드 전환 검증"
description: "Flutter WidgetTester에서 PageView를 스와이프해 스마트홈 기기 모드를 바꾸고, MQTT 명령이 중복 발행되지 않는지 검증하는 실전 테스트 패턴을 정리했다."
date: 2026-09-14
tags: [Flutter, Dart, 테스트, MQTT, IoT, 스마트홈]
comments: true
share: true
---

![Flutter WidgetTester PageView MQTT 스마트홈 기기 모드 테스트](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

이 그림에서 볼 것은 스마트홈 앱의 기기 제어 화면이다. 화면을 넘기는 동작과 실제 명령 발행은 서로 다른 책임으로 나눠야 테스트가 흔들리지 않는다.

`PageView`는 화면에 보이는 페이지만 검사하면 되는 것처럼 보이지만, 실제로는 스와이프 애니메이션이 끝나는 시점과 MQTT 명령을 보내는 시점이 엇갈린다. 처음엔 `tester.drag()` 한 번이면 끝날 줄 알았는데 아니었다. 페이지는 바뀌었는데 `onPageChanged`가 아직 실행되지 않아 테스트가 간헐적으로 실패했다.

## 테스트 대상 화면

보일러 제어 화면에 세 페이지를 둔다. 수동 모드, 예약 모드, 외출 모드다. 사용자가 페이지를 넘겼을 때만 선택 모드를 저장하고 MQTT 명령을 발행한다. 버튼을 누를 때 명령을 발행하는 구조와 구분하는 게 포인트다.

| 확인할 것 | 테스트 기준 |
|---|---|
| 페이지 이동 | `PageView`가 목표 페이지에 도착한다 |
| 상태 반영 | 현재 모드 라벨이 한 번만 갱신된다 |
| MQTT 발행 | `mode/set` 명령이 선택된 모드로 한 번만 전송된다 |
| 애니메이션 | 이동 중이 아니라 정착 후 결과를 검증한다 |

실제 MQTT 클라이언트를 위젯 테스트에 넣으면 브로커 연결 상태가 테스트 결과를 결정한다. 그래서 화면에는 `MqttCommandPort`만 주입하고, 테스트에서는 호출 기록을 남기는 Fake를 사용했다.

```dart
abstract interface class MqttCommandPort {
  Future<void> publishMode(String deviceId, String mode);
}

class RecordingMqttCommand implements MqttCommandPort {
  final calls = <({String deviceId, String mode})>[];

  @override
  Future<void> publishMode(String deviceId, String mode) async {
    calls.add((deviceId: deviceId, mode: mode));
  }
}
```

이 Fake는 네트워크를 흉내 내지 않는다. 어떤 기기에 어떤 모드를 몇 번 보냈는지만 기록한다. 연결 재시도나 JSON 직렬화는 별도 단위 테스트의 책임이다.

## PageView를 테스트용 위젯으로 감싸기

테스트 대상은 `PageController`를 외부에서 만들지 않고 내부에서 관리하도록 단순화했다. 현재 페이지를 직접 읽기보다 화면에 표시되는 키를 기준으로 검증하면 구현이 바뀌어도 테스트 의도가 남는다.

```dart
class ModePager extends StatefulWidget {
  const ModePager({required this.deviceId, required this.mqtt, super.key});

  final String deviceId;
  final MqttCommandPort mqtt;

  @override
  State<ModePager> createState() => _ModePagerState();
}

class _ModePagerState extends State<ModePager> {
  static const modes = ['manual', 'schedule', 'away'];
  var selectedIndex = 0;

  Future<void> _changeMode(int index) async {
    setState(() => selectedIndex = index);
    await widget.mqtt.publishMode(widget.deviceId, modes[index]);
  }

  @override
  Widget build(BuildContext context) {
    return PageView.builder(
      key: const Key('modePager'),
      itemCount: modes.length,
      onPageChanged: _changeMode,
      itemBuilder: (context, index) => Center(
        child: Text(modes[index], key: Key('mode-${modes[index]}')),
      ),
    );
  }
}
```

핵심은 `onPageChanged` 안에서만 MQTT를 호출하는 부분이다. `build()`에서 현재 페이지를 감지하거나 `PageController`의 listener에서 매번 발행하면 드래그 중 중복 명령이 생길 수 있다.

## fling과 pumpAndSettle의 조합

페이지를 넘기는 테스트 코드는 다음처럼 작성했다. `drag`보다 `fling`이 실제 손가락 넘김에 가깝고, `pumpAndSettle`은 페이지 스냅 애니메이션이 끝날 때까지 프레임을 진행한다.

```dart
testWidgets('예약 모드로 넘기면 MQTT 명령을 한 번 보낸다', (tester) async {
  final mqtt = RecordingMqttCommand();

  await tester.pumpWidget(
    MaterialApp(
      home: SizedBox(
        height: 400,
        child: ModePager(deviceId: 'boiler-01', mqtt: mqtt),
      ),
    ),
  );

  expect(find.byKey(const Key('mode-manual')), findsOneWidget);

  await tester.fling(
    find.byKey(const Key('modePager')),
    const Offset(-500, 0),
    1200,
  );
  await tester.pumpAndSettle();

  expect(find.byKey(const Key('mode-schedule')), findsOneWidget);
  expect(mqtt.calls, hasLength(1));
  expect(mqtt.calls.single.deviceId, 'boiler-01');
  expect(mqtt.calls.single.mode, 'schedule');
});
```

여기서 `pump()`만 호출하면 기기나 실행 환경에 따라 `manual` 페이지가 남아 있을 수 있다. 반대로 무조건 `pumpAndSettle()`만 믿으면 MQTT Fake가 즉시 완료되지 않는 구조에서 테스트가 멈출 수 있다. 네트워크 Fake의 Future는 반드시 완료 가능한 상태로 만들고, 실제 타이머가 필요한 재연결 테스트와 분리해야 한다.

## 실패했던 가정과 체크리스트

스와이프 방향을 반대로 넣는 실수가 가장 흔했다. `PageView`에서 왼쪽으로 넘기려면 x 오프셋이 음수여야 한다. 또 페이지 폭보다 작은 값을 주면 다음 페이지로 정착하지 않을 수 있다. 테스트 실패 시 애니메이션 문제인지, 방향 문제인지, 콜백 중복인지 로그를 나눠서 봐야 한다.

- 페이지 이동은 `find.byKey`로 찾는다.
- 텍스트는 다국어 변경에 대비해 상태 검증의 유일한 기준으로 쓰지 않는다.
- `onPageChanged` 호출 횟수와 MQTT 발행 횟수를 함께 검사한다.
- `pumpAndSettle` 뒤에 최종 UI와 Fake 기록을 검증한다.
- 브로커 연결, 재시도, payload 검증은 Widget Test 밖에서 다룬다.

솔직하게 정리하면 `PageView` 테스트의 어려움은 스와이프 자체보다 이벤트 경계에 있다. 화면 전환은 위젯 테스트로 빠르게 잡고, MQTT 전송 정책은 Fake 호출 기록으로 고정하니 테스트가 안정됐다. 애니메이션 시간을 늘려서 통과시키는 방식은 잠깐 편하지만, 실행 환경이 느려지면 다시 깨진다.

