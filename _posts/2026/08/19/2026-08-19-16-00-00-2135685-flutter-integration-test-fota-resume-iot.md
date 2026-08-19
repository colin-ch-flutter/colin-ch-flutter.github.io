---
layout: post
title: "Flutter integration_test FOTA 재개 테스트 - 중단된 IoT 펌웨어 업데이트 검증"
description: "Flutter integration_test에서 FOTA 다운로드가 중단된 뒤 진행률과 재개 요청이 올바르게 이어지는지 Fake 이벤트로 검증하는 방법을 정리했다."
date: 2026-08-19
tags: [Flutter, Dart, integration_test, FOTA, IoT, CI/CD, Android, iOS]
comments: true
share: true
---

![Flutter integration_test FOTA 펌웨어 업데이트 재개 테스트](https://images.unsplash.com/photo-1518770660439-4636190af475?w=1200&q=80)

이 그림에서 볼 부분은 펌웨어를 내려받는 기기와 앱이 서로 다른 상태를 가질 수 있다는 점이다. Flutter integration_test에서 FOTA를 검증할 때 완료 화면만 확인하면 부족하다. 다운로드가 42%에서 끊긴 뒤 앱을 다시 열었을 때, 처음부터 다시 받는지 이어 받는지까지 확인해야 실제 IoT 장애를 잡을 수 있다.

## 진행률 100%만 테스트했던 실수

예전에 FOTA 화면을 만들 때는 `completed` 이벤트를 넣고 완료 문구가 보이는지만 검사했다. 로컬에서는 통과했지만 실제 기기에서 Wi-Fi가 잠깐 끊기자 두 가지 문제가 나왔다.

| 상황 | 잘못된 구현에서 보인 결과 | 기대하는 결과 |
|---|---|---|
| 다운로드 42%에서 연결 끊김 | 진행률이 0%로 초기화됨 | 42%를 보존하고 재개 |
| 앱을 백그라운드로 전환 | 완료 이벤트를 놓침 | 복귀 후 상태 조회 |
| 재개 버튼 중복 탭 | 다운로드 요청이 두 번 전송됨 | 하나의 작업만 유지 |

기존 [FOTA 무선 펌웨어 업데이트 구현]({% post_url 2026-06-08-10-40-40-493800-fota-wireless-firmware-update-ota %})은 화면과 MQTT 진행 이벤트를 연결하는 글이다. 이번 테스트에서는 서버나 실제 기기를 호출하지 않고 상태 전이의 계약만 고정한다.

## FOTA 상태를 이벤트로 분리한다

테스트가 진행률을 직접 바꾸면 UI만 검증하게 된다. 앱이 실제로 받는 것처럼 `FotaEvent`를 주입할 수 있는 경계를 둔다.

```dart
enum FotaStage { idle, downloading, interrupted, resuming, completed, failed }

class FotaEvent {
  const FotaEvent(this.stage, this.percent);

  final FotaStage stage;
  final int percent;
}

abstract interface class FotaGateway {
  Stream<FotaEvent> get events;
  Future<void> resume(String deviceId, {required int fromPercent});
}
```

`Controller`는 MQTT 패킷을 해석하지 않고 이벤트에 따라 화면 상태만 바꾼다. 이 경계가 있어야 integration_test에서 네트워크 지연 대신 원하는 중단 시점을 정확히 만들 수 있다.

```dart
class FakeFotaGateway implements FotaGateway {
  final _events = StreamController<FotaEvent>.broadcast();
  int? resumedFrom;

  @override
  Stream<FotaEvent> get events => _events.stream;

  void emit(FotaEvent event) => _events.add(event);

  @override
  Future<void> resume(String deviceId, {required int fromPercent}) async {
    resumedFrom = fromPercent;
  }

  Future<void> dispose() => _events.close();
}
```

`resumedFrom`을 기록한 이유는 버튼이 눌렸다는 사실보다 “42%부터 재개를 요청했는가”가 더 중요한 계약이기 때문이다. 실제 코드에서는 서버가 재개 가능한 offset을 다시 내려줄 수 있으므로, 앱에 남아 있던 진행률을 무조건 신뢰하지 않는 편이 안전하다.

## integration_test에서 중단과 재개를 재현한다

앱 부트스트랩에서 `FakeFotaGateway`를 주입한 뒤 테스트 이벤트를 순서대로 보낸다. 테스트 전용 진입점에서만 Fake를 허용하고 운영 빌드에서는 생성되지 않게 가드해야 한다.

```dart
testWidgets('FOTA 중단 후 같은 지점에서 재개한다', (tester) async {
  final fake = FakeFotaGateway();
  await launchTestApp(fotaGateway: fake);

  fake.emit(const FotaEvent(FotaStage.downloading, 42));
  await tester.pump();
  expect(find.text('42%'), findsOneWidget);

  fake.emit(const FotaEvent(FotaStage.interrupted, 42));
  await tester.pump();
  expect(find.text('업데이트 재개'), findsOneWidget);

  await tester.tap(find.text('업데이트 재개'));
  expect(fake.resumedFrom, 42);

  fake.emit(const FotaEvent(FotaStage.resuming, 42));
  fake.emit(const FotaEvent(FotaStage.completed, 100));
  await tester.pump();
  expect(find.text('업데이트 완료'), findsOneWidget);

  await fake.dispose();
});
```

여기서 `pumpAndSettle()`을 쓰지 않은 것은 의도적이다. FOTA 스트림은 기기 응답을 기다리는 동안 계속 열려 있을 수 있어서, 모든 비동기 작업이 끝나기를 기다리면 테스트가 영원히 settle되지 않는다. 이벤트 하나를 넣고 한 프레임만 진행하는 방식이 상태 순서를 더 분명하게 보여준다.

## 실제 기기 테스트와 나눌 기준

Fake 테스트가 통과해도 BLE·Wi-Fi·전원 차단까지 보장하는 것은 아니다. 테스트의 역할을 나누면 CI가 느려지지 않으면서 위험한 구간을 남길 수 있다.

| 테스트 | 확인 대상 | 실행 위치 |
|---|---|---|
| Fake integration_test | 중단·재개 상태, 중복 요청 방지 | 모든 PR |
| Android 실기기 | 앱 백그라운드 복귀와 권한 | 일일 빌드 |
| 실제 FOTA 장치 | 다운로드·설치·재부팅 | 릴리스 전 |

실제 장치에서는 42%에서 전원을 끄는 시나리오까지 자동화하기 어렵다. 대신 장치 시뮬레이터가 `interrupted` 이벤트를 발행하도록 만들고, 단 한 번의 수동 릴리스 검증에서 전원·네트워크 차단을 확인했다. 테스트 이름에 `resume`을 넣고 실패 시 마지막 이벤트와 퍼센트를 로그로 남기면 CI에서 원인을 좁히기 쉽다.

짧게 정리하면 Flutter integration_test FOTA 검증의 핵심은 완료 화면이 아니다. 중단 시점의 진행률을 보존하고, 재개 요청이 한 번만 나가며, 앱 복귀 뒤 최신 상태를 다시 받는지를 상태 전이로 고정해야 한다. 실제 MQTT와 기기 통신은 별도 실기기 테스트로 남기고, 반복되는 계약은 Fake 이벤트로 빠르게 돌리는 구성이 현실적이다.
