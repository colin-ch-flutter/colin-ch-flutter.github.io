---
layout: post
title: "Flutter Unit Test 실전 시리즈 회고 — IoT 앱에서 배운 테스트 격리의 실제 가치"
description: "BLE·MQTT·Firebase까지 IoT 앱의 모든 레이어를 단위 테스트로 격리하며 배운 것들. 효과 있었던 Fake 패턴, 솔직히 과했던 테스트, IoT 앱만의 특이점을 정리했다."
date: 2026-06-29
tags: [Flutter, Dart, 회고, IoT, GetX]
comments: true
share: true
---

![Flutter 테스트 코드 회고](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

Flutter IoT 스마트홈 시리즈 50편을 마무리하면서 한 가지 아쉬움이 남았다. 테스트 코드가 없었다. Repository 패턴 덕분에 구조는 잡혀 있었는데, 정작 그 구조를 활용한 테스트가 단 한 줄도 없었다. 그래서 바로 이어서 시작한 게 Flutter Unit Test 실전 시리즈였다.

BLE 격리부터 시작해서 MQTT, Firebase, Platform Channel, Realm, Dio, GetX 라우팅, 애니메이션 로케일까지. 뭔가를 쓸 때마다 테스트하기 까다롭다는 걸 새삼 깨달았다. IoT 앱은 네이티브 플러그인에 의존하는 게 많아서 단위 테스트 격리가 특히 성가시다.

## 왜 처음부터 테스트를 안 썼나

솔직히 말하면 IoT 프로젝트 초반에는 "실기기에서 동작하면 된다"는 생각이 강했다. BLE 연결이 되냐 안 되냐는 unit test로 확인하기 어렵고, MQTT 메시지가 정말 오가는지도 브로커 없이는 테스트가 불가능하다.

근데 이게 함정이었다. **실기기 테스트는 회귀 검증이 안 된다.** 기기 연결해서 UI 확인하는 건 매번 손으로 해야 하고, 로직 수정 이후 "이전 동작이 깨졌는지"를 알 방법이 없다. 버그는 항상 관련 없어 보이는 수정 이후에 튀어나왔다.

## 시리즈 전체를 관통한 핵심 패턴 하나

시리즈를 다 쓰고 나서 보니, 결국 모든 격리는 같은 패턴으로 귀결됐다.

```dart
// 1. 추상 인터페이스 정의
abstract class IXxxService {
  // 테스트하려는 동작 선언
}

// 2. 실제 구현 (네이티브/외부 의존)
class RealXxxService implements IXxxService { ... }

// 3. Fake 구현 (인메모리, 제어 가능)
class FakeXxxService implements IXxxService { ... }

// 4. Controller/Repository는 인터페이스만 알고 있음
class SomeController {
  SomeController(this._xxx);
  final IXxxService _xxx;
}
```

BLE든 MQTT든 Firebase든 SharedPreferences든 Platform Channel이든, 이 구조를 벗어난 경우가 없었다. 처음엔 "이걸 위해 인터페이스 하나 더 만들어야 해?" 싶었는데, 반복하다 보니 이게 정석이라는 게 몸으로 느껴졌다.

## 실제로 효과 있었던 것들

**BLE Controller 테스트 — 예상보다 훨씬 유용했다.**

`flutter_blue_plus`를 직접 쓰던 코드를 `IBleService` 뒤로 숨기고 나서, 재연결 로직 테스트가 가능해졌다. "5초 안에 재연결 시도를 3번 하는가", "3번 실패하면 에러 상태로 전환되는가" — 이런 게 실기기 없이 검증됐다. 타이밍 테스트는 `fakeAsync`로 해결했다.

**MQTT 메시지 흐름 테스트 — 버그를 두 번 잡았다.**

FakeMqttService를 만들고 나서 "토픽 구독이 제대로 됐을 때만 메시지 콜백이 실행되는가"를 테스트하다가, 구독 전에 publish가 먼저 도는 엣지 케이스를 발견했다. 실기기 테스트에서는 타이밍 때문에 재현이 어려운 케이스였다.

**Dio 인터셉터 테스트 — 리팩토링에서 진가 발휘.**

JWT 자동갱신 인터셉터를 수정할 때마다 손으로 로그인 토큰 만료 기다리기를 반복했었다. 테스트 작성 이후로 그 짓을 안 해도 된다. 만료된 토큰 주입 → 401 응답 → 갱신 요청 → 재시도 성공 전체 흐름이 3초 안에 돈다.

![Flutter IoT 테스트 격리 구조](https://images.unsplash.com/photo-1504639725590-34d0984388bd?w=800&q=80)

## 솔직히 과했던 것들

**Golden Test — 유지 비용이 너무 높다.**

골든 파일이 CI 환경의 폰트·DPR 차이로 계속 깨졌다. `alchemist`를 쓰면 어느 정도 안정화되지만, 사소한 UI 수정마다 골든 파일을 재생성해야 하는 건 여전히 번거롭다. IoT 제어 카드처럼 자주 바뀌는 UI에는 golden test보다 widget test가 낫다.

**E2E + Native (Patrol) — 설정 비용 대비 효과가 불분명했다.**

BLE 권한 다이얼로그 자동화는 멋져 보였는데, 실기기에서 돌리기 위한 CI 설정이 복잡했다. Emulator에서는 BLE 자체가 안 되니까. 테스트 가치는 분명히 있지만, 프로젝트 초반에 붙이기에는 진입 장벽이 높다.

**every single plugin에 Fake 만들기 — 선택과 집중이 필요했다.**

`easy_localization`, `wakelock_plus` 같은 단순한 플러그인까지 Fake를 만들었는데, 솔직히 `setMockInitialValues`나 `TestDefaultBinaryMessengerBinding`으로 충분한 경우가 많았다. Fake 계층이 많아질수록 관리 부담도 늘어난다.

## IoT 앱 테스트만의 특이점

일반 CRUD 앱이랑 IoT 앱의 테스트가 다른 점이 몇 가지 있다.

**1. 스트림이 많다.**

BLE 특성 변화, MQTT 메시지 수신, 네트워크 상태 변화가 전부 스트림이다. `Fake`를 만들 때 `StreamController`를 노출해서 테스트에서 직접 이벤트를 주입하는 패턴을 반복적으로 썼다. [fakeAsync와 조합하면]({% post_url 2026-06-26-20-00-00-814770-flutter-firebase-test-isolation-auth-remote-config-fake %}) 타이밍 제어까지 가능해진다.

**2. 플랫폼 채널 의존이 유독 많다.**

BLE, WiFi, 카메라(QR), 로컬 알림, 보안 저장소가 전부 네이티브 채널을 쓴다. [MethodChannel mock]({% post_url 2026-06-27-10-00-00-827115-flutter-platform-channel-mock-test-methodchannel %})을 알고 나서야 플러그인 초기화 오류 없이 테스트를 돌릴 수 있었다.

**3. 기기 상태가 복잡한 상태 머신이다.**

보일러 기기 하나가 "연결 중 → 연결됨 → 제어 중 → 에러 → 재연결 중" 같은 상태를 가진다. [테스트 안티패턴]({% post_url 2026-06-28-18-00-00-925875-flutter-test-antipatterns-code-quality %})에서 다뤘지만, 상태 전이마다 별도 테스트를 작성하는 게 버그를 사전에 잡는 데 확실히 도움이 됐다.

## 커버리지 수치에 대한 생각

시리즈 초반에 "80% 커버리지 달성"을 목표로 잡은 적이 있다. 근데 막상 달성하고 나니 숫자가 의미 없다는 걸 느꼈다. Repository의 `throw Exception` 경로에 테스트를 붙여서 커버리지를 채웠는데, 그게 실제로 유용한 테스트냐 하면 그건 아니었다.

지금은 커버리지보다 **"이 테스트가 없으면 어떤 회귀가 잡히지 않을까?"**를 기준으로 쓴다. BLE 재연결 로직, JWT 갱신 흐름, MQTT 구독 등록 순서처럼 "한 번 틀리면 장애로 직결되는 곳"을 먼저 커버한다.

## 다음에 이 구조를 다시 쓴다면

시리즈를 통해 쌓인 패턴을 처음부터 쓴다면 이렇게 할 것 같다.

1. **프로젝트 초기부터 인터페이스 + Fake 세트로 플러그인을 감싼다.** 나중에 추가하면 기존 코드를 다 수정해야 해서 피곤하다.
2. **비동기·스트림 로직에만 집중 투자한다.** 동기 getter/setter에 테스트 붙이는 시간을 아껴서 복잡한 흐름을 더 촘촘하게 커버한다.
3. **CI는 unit + widget까지만 자동화한다.** Integration test와 Patrol은 PR 머지 전 수동 확인 + 주기적 배치 실행으로 나눈다.

---

시리즈를 끝까지 읽어주신 분들께 감사하다. IoT 앱 특성상 "실기기에서 확인하면 되지"라는 생각이 강하게 드는데, 막상 테스트를 작성하고 나면 생각보다 많은 걸 잡아낸다는 걸 느꼈다. 다음 시리즈 주제를 고민 중이다.

**참고 링크**
- [Flutter 공식 테스트 문서](https://docs.flutter.dev/testing)
- [mocktail 패키지](https://pub.dev/packages/mocktail)
- [alchemist (golden test)](https://pub.dev/packages/alchemist)
- [patrol (E2E + native)](https://pub.dev/packages/patrol)
