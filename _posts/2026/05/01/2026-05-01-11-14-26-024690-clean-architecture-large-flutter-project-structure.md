---
layout: post
title: "클린 아키텍처 적용하기 - 대규모 Flutter 프로젝트 구조 설계"
description: " "
date: 2026-05-01
tags: [Flutter, CleanArchitecture, 아키텍처, 프로젝트구조]
comments: true
share: true
---

# 클린 아키텍처 적용하기 - 대규모 Flutter 프로젝트 구조 설계

![아키텍처 설계](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

## 왜 클린 아키텍처인가

Flutter 앱을 처음 만들 때 많은 사람이 단순하게 시작한다. 뷰와 로직이 뒤섞인 구조로 빠르게 만들다 보면 어느 순간 파일 하나가 500줄을 넘어가고, 어디서 뭘 수정해야 할지 헷갈리는 상황이 온다.

스마트홈 앱은 처음부터 기능이 많았다. BLE, MQTT, REST API, 로컬 DB, 실시간 상태 관리... 이걸 아무 구조 없이 짜면 유지보수 지옥이 될 게 뻔했다. 그래서 초반에 시간을 좀 투자해서 아키텍처부터 잡고 들어갔다.

## 레이어 구조

프로젝트는 크게 네 개의 레이어로 나뉜다:

```
lib/
├── core/           # 앱 전반에서 사용하는 공통 코드
│   ├── bindings/   # GetX 바인딩, 라우팅
│   ├── config/     # 환경 설정 (dev/stage/prod)
│   ├── constants/  # 상수 정의
│   ├── services/   # 앱 레벨 서비스 (인증, 푸시)
│   └── utils/      # 유틸리티 함수들
│
├── domain/         # 비즈니스 로직 (외부 의존 없음)
│   ├── models/     # 도메인 모델
│   ├── repositories/ # 리포지토리 인터페이스
│   ├── usecases/   # 유스케이스
│   └── ports/      # 포트(캐시, MQTT 컨텍스트 등)
│
├── data/           # 도메인 인터페이스 구현체
│   ├── repositories/ # 리포지토리 구현
│   ├── services/   # 외부 서비스 (API, BLE, MQTT)
│   ├── models/     # 데이터 모델 (JSON 매핑용)
│   ├── mapper/     # 데이터 모델 → 도메인 모델 변환
│   ├── ble/        # BLE 관련 코드
│   └── cache/      # 캐시 구현
│
└── presentation/   # UI
    ├── core/       # 공통 위젯, 테마, 컨트롤러
    ├── signin/     # 로그인/회원가입
    ├── main/       # 메인 탭
    ├── device/     # 기기 연결/제어
    ├── manage/     # 기기/공간/멤버 관리
    ├── smart/      # 퀵모드/자동화
    └── menu/       # 설정/메뉴
```

### Domain 레이어

Domain 레이어는 외부 라이브러리 의존이 없어야 한다. Flutter SDK조차 최대한 쓰지 않는 게 이상적이다. 여기에는 비즈니스 규칙이 담긴다.

Repository 인터페이스 예시:

```dart
// domain/repositories/iot_device_mqtt_repository.dart
abstract class IoT DeviceMqttRepository {
  Future<void> connect(MqttContext context);
  Future<void> disconnect();
  Future<void> publishPowerControl(String deviceId, bool isOn);
  Future<void> publishTemperatureControl(String deviceId, int temperature);
  Stream<IoT DeviceStatus> get statusStream;
}
```

구현은 `data` 레이어에서 하고, `domain`에서는 인터페이스만 정의한다. 덕분에 테스트할 때 Mock 구현체로 쉽게 교체할 수 있다.

### Data 레이어

실제 MQTT 통신, API 호출, BLE 연결 등이 여기에 있다:

```dart
// data/repositories/iot_device_mqtt_repository_impl.dart
class IoT DeviceMqttRepositoryImpl implements IoT DeviceMqttRepository {
  final MqttService _mqttService;

  IoT DeviceMqttRepositoryImpl(this._mqttService);

  @override
  Future<void> publishPowerControl(String deviceId, bool isOn) async {
    final topic = 'device/$deviceId/command/power';
    final payload = jsonEncode({'power': isOn ? 'on' : 'off'});
    await _mqttService.publish(topic, payload);
  }
  // ...
}
```

### Mapper 패턴

API 응답 JSON과 도메인 모델은 분리한다. `data/mapper/` 에서 변환을 담당한다:

```dart
// data/mapper/api/iot_device_mapper.dart
class IoT DeviceMapper {
  static IoT DeviceDevice fromApiModel(IoT DeviceApiModel model) {
    return IoT DeviceDevice(
      id: model.deviceId,
      name: model.nickname ?? model.modelName,
      status: _mapStatus(model.statusCode),
    );
  }
}
```

API 스펙이 바뀌어도 Mapper만 수정하면 된다. Domain 모델은 건드리지 않아도 된다.

### Presentation 레이어

GetX의 Controller가 여기에 있다. View는 Controller를 통해서만 데이터를 가져오고, Controller는 Repository를 통해서만 데이터를 가져온다:

```dart
// presentation/device/control/iot_device/iot_device_control_controller.dart
class IoT DeviceControlController extends GetxController {
  final IoT DeviceMqttRepository _mqttRepo;
  
  final Rx<IoT DeviceStatus?> status = Rx(null);
  
  IoT DeviceControlController(this._mqttRepo);
  
  @override
  void onInit() {
    super.onInit();
    _mqttRepo.statusStream.listen((s) => status.value = s);
  }
  
  Future<void> togglePower() async {
    final current = status.value?.isOn ?? false;
    await _mqttRepo.publishPowerControl(deviceId, !current);
  }
}
```

## 테스트 모드 구현이 깔끔해진다

이 구조의 가장 큰 장점 중 하나가 테스트 모드다. 실제 IoT 기기 기기 없이 개발할 때 `data/test/` 에 Mock 구현체를 두고 교체해서 쓴다:

```dart
// data/test/test_iot_device_impl.dart
class TestIoT DeviceMqttRepositoryImpl implements IoT DeviceMqttRepository {
  final _controller = StreamController<IoT DeviceStatus>.broadcast();

  @override
  Future<void> publishPowerControl(String deviceId, bool isOn) async {
    // 실제 MQTT 대신 로컬에서 상태 변경 시뮬레이션
    await Future.delayed(const Duration(milliseconds: 500));
    _controller.add(IoT DeviceStatus(isOn: isOn, temperature: 22));
  }
  // ...
}
```

GetX Binding에서 환경에 따라 구현체를 교체해주면 된다.

## 처음에 느꼈던 불편함

솔직히 처음에는 이 구조가 너무 과하다고 느꼈다. 간단한 기능 하나 추가하려면 model → repository interface → repository impl → mapper → controller → view 순으로 파일을 여러 개 만들어야 했다.

그런데 프로젝트가 어느 정도 커지고 나서부터는 확실히 관리가 쉬워졌다. 새 기능을 추가할 때 어디에 뭘 써야 할지 명확하고, 버그가 생겼을 때도 어느 레이어 문제인지 좁혀지는 속도가 빨라졌다.

---

다음 편은 GetX로 상태관리와 라우팅을 어떻게 구성했는지 다룬다.
