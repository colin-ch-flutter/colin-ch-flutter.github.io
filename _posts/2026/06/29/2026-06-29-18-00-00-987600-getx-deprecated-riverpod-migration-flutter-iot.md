---
layout: post
title: "GetX 공식 레포가 사라졌다 — 우리 IoT 앱, 지금 Riverpod으로 갈아타야 하나"
description: "2026년 4월 GetX 공식 GitHub 레포가 사라지면서 GetX는 사실상 기술 부채가 됐다. 50편 IoT 스마트홈 앱을 GetX로 만든 입장에서 Riverpod 3.0 마이그레이션을 실제로 고민한 과정을 정리했다."
date: 2026-06-29
tags: [Flutter, GetX, Riverpod, 상태관리, IoT]
comments: true
share: true
---

![Flutter 상태관리 전환 코드 화면](https://images.unsplash.com/photo-1526374965328-7f61d4dc18c5?w=800&q=80)

4월 어느 날, 커뮤니티에서 이상한 글이 올라오기 시작했다. GetX 공식 GitHub 레포지토리가 접근이 안 된다고. 처음엔 GitHub 서버 문제인 줄 알았는데, 며칠이 지나도 복구 소식이 없었다. `jonataslaw/getx` 레포가 그냥 사라진 것이었다.

50편을 GetX로 썼다. [GetX 상태관리·라우팅]({% post_url 2026-05-02-09-21-39-037035-getx-state-management-routing-flutter-iot %})부터 시작해서 Controller·Repository 패턴, 의존성 주입까지 전부 GetX 기반이다. 지금 당장 돌아가는 앱이 있고, 코드 수만 줄이 GetX에 묶여 있는 상황에서 이 소식을 어떻게 받아들여야 하나.

## GetX 레포 소멸 — 정확히 뭔 일이었나

2026년 4월, `jonataslaw/getx` GitHub 레포지토리가 비공개 처리됐다. 정확한 이유는 공개되지 않았다. 작성자가 직접 내린 건지, 다른 이유가 있는지 확인할 방법이 없다.

pub.dev에 패키지는 여전히 남아 있다. 기존 `get: ^4.6.6` 의존성은 아직 설치된다. 근데 문제는 버그 수정이나 신규 Flutter 버전 대응이 막혔다는 것이다. Flutter 3.x 이상에서 생기는 breaking change가 있어도 공식 업데이트가 없다.

현재 커뮤니티 포크들이 있긴 하다. 근데 포크는 포크다. 누가 관리하다 멈추면 그게 끝이다.

2026년 기준 Flutter 상태관리 권고사항은 명확해졌다.

- **신규 프로젝트**: Riverpod 3.0 (compile-time safety, code generation)
- **엔터프라이즈 audit trail 필요**: Bloc 9.0
- **기존 GetX 앱**: 마이그레이션 예산 없으면 현상 유지, 있으면 Riverpod

## 우리 IoT 앱 코드베이스 현황

50편짜리 Flutter IoT 스마트홈 앱이다. [클린 아키텍처]({% post_url 2026-05-01-11-14-26-024690-clean-architecture-large-flutter-project-structure %})를 적용했고, GetX는 다음 역할을 담당한다.

- `GetxController`: BleController, MqttController, DeviceController 등 12개
- `Get.put()` / `Get.find()`: 전역 의존성 관리
- `GetX()` / `Obx()`: UI 반응형 렌더링
- `Get.to()` / `Get.back()`: 네비게이션

규모가 작지 않다. 컨트롤러 12개에 `Obx()` 위젯이 수백 개다. 이걸 한 번에 갈아엎는 건 불가능하고, 사실 하고 싶지도 않다.

## Riverpod 3.0이 뭐가 다른가

Riverpod 3.0의 핵심은 두 가지다. **compile-time safety**와 **자동 재시도**.

```dart
// Riverpod 3.0 — NotifierProvider 기본 패턴
@riverpod
class BleController extends _$BleController {
  @override
  BleState build() => BleState.disconnected;

  Future<void> connect(String deviceId) async {
    state = BleState.connecting;
    try {
      await ref.read(bleServiceProvider).connect(deviceId);
      state = BleState.connected;
    } catch (e) {
      state = BleState.error(e.toString());
    }
  }
}
```

GetX였다면 이렇게 생겼을 코드다.

```dart
// GetX 기존 패턴
class BleController extends GetxController {
  final _bleService = Get.find<IBleService>();
  final bleState = BleState.disconnected.obs;

  Future<void> connect(String deviceId) async {
    bleState.value = BleState.connecting;
    try {
      await _bleService.connect(deviceId);
      bleState.value = BleState.connected;
    } catch (e) {
      bleState.value = BleState.error(e.toString());
    }
  }
}
```

보기엔 비슷하다. 차이는 런타임에서 드러난다.

GetX의 `Get.find<IBleService>()`는 런타임 타입 조회다. 등록 안 된 의존성 찾으면 그냥 터진다. Riverpod는 `ref.read(bleServiceProvider)`가 compile-time에 타입이 확정되어 있어서 빌드 전에 잡힌다.

Riverpod 3.0에서 추가된 자동 재시도는 IoT 앱에서 실제로 쓸 만하다. MQTT 브로커 연결이 끊겼을 때 지수 백오프로 자동 재시도한다.

```dart
@riverpod
Future<MqttClient> mqttConnection(MqttConnectionRef ref) async {
  // autoDispose + keepAlive 조합으로 연결 상태 자동 관리
  // 연결 실패 시 200ms → 400ms → 800ms → ... → 6.4s 로 자동 재시도
  final client = await MqttService().connect();
  ref.onDispose(() => client.disconnect());
  return client;
}
```

MQTT 재연결 로직을 직접 짰던 기억이 있다. BLE도 마찬가지였다. Riverpod이 이걸 프레임워크 레벨에서 처리해준다면 상당한 코드가 없어진다.

![Flutter Riverpod Provider 구조도](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

## IoT 앱에서 실제 마이그레이션하면 어떻게 되나

BleController 하나만 옮겨봤다. 체감 난이도는 이렇다.

**쉬운 것:**
- `RxBool`, `RxString` → `state`로 변경. 구조 자체는 비슷
- Repository 주입: `Get.put()` → `@riverpod` 어노테이션

**번거로운 것:**
- `GetX()` / `Obx()` 위젯 전부 `ConsumerWidget`으로 변경
- `GetBuilder`로 감싼 복잡한 위젯들 분리

**진짜 골치 아픈 것:**
- 네비게이션. `Get.to()`, `Get.off()`가 앱 전체에 퍼져 있다. Riverpod은 라우팅을 안 건드려서 GoRouter나 다른 걸로 별도 교체해야 한다.
- 다이얼로그·스낵바. `Get.snackbar()`, `Get.dialog()` 편의 함수들이 생각보다 많이 쓰인다.

Controller 12개 + 라우팅 전체 교체까지 치면 솔직히 2~3주 작업이다. 기존 테스트도 전부 다시 짜야 한다.

## 지금 당장 마이그레이션해야 하나

결론부터 말하면: **신규 프로젝트면 Riverpod으로 시작. 기존 앱이면 서두를 필요 없다.**

현재 GetX 앱이 Flutter 최신 버전에서 돌아가고 있고, 당장 심각한 버그가 없다면 강제로 갈아엎을 이유는 없다. 패키지는 pub.dev에 남아 있고, Dart 생태계 특성상 과거 버전 호환성은 꽤 오래 유지된다.

다만 6개월~1년 후에는 다르다. Flutter 메이저 업데이트에서 GetX가 따라가지 못하는 breaking change가 생기면 그때는 선택의 여지가 없어진다. 포크 버전에 의존하거나, 아니면 마이그레이션이다.

그래서 지금 해두면 좋은 것 하나: **의존성을 한 곳으로 모아두는 것.** 이미 Repository 패턴을 쓰고 있다면, `Get.put()` 등록을 `Binding` 클래스에 집중시켜 두면 나중에 Riverpod으로 옮길 때 진입점이 명확해진다. 분산된 `Get.put()` 코드를 나중에 찾는 것보다 지금 정리해두는 게 훨씬 낫다.

---

다음에는 Riverpod 3.0을 IoT 앱에서 처음부터 쓴다면 어떻게 구조를 잡을지, Provider 분리 전략과 실제 코드로 정리해볼 생각이다.

**참고 링크**
- [Riverpod 공식 문서](https://riverpod.dev)
- [pub.dev - flutter_riverpod](https://pub.dev/packages/flutter_riverpod)
- [BLoC vs Riverpod vs GetX in 2026](https://ottomancoder.medium.com/bloc-vs-riverpod-vs-getx-in-2026-my-honest-take-after-50-flutter-apps-56242261b14c)
