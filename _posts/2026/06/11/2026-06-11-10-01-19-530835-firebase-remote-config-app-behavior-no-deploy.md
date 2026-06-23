---
layout: post
title: "Firebase Remote Config로 서버 배포 없이 앱 동작 변경하기"
description: " "
date: 2026-06-11
tags: [Flutter, Firebase, RemoteConfig, 기능플래그, 앱배포]
comments: true
share: true
---

# Firebase Remote Config로 서버 배포 없이 앱 동작 변경하기

![Firebase Remote Config](https://images.unsplash.com/photo-1451187580459-43490279c0fa?w=800&q=80)

## Remote Config의 강력함

Remote Config는 앱을 새로 배포하지 않고도 앱 동작을 바꿀 수 있게 해주는 서비스다. Firebase 콘솔에서 값을 바꾸면 앱이 다음 번에 그 값을 읽어서 동작을 바꾼다.

스마트홈 앱에서 Remote Config를 쓰는 주요 케이스:

1. **앱 스토어 심사 모드**: 심사 중에 데모 환경으로 전환
2. **긴급 기능 OFF**: 특정 기능에 심각한 버그가 있을 때 서버 배포 없이 비활성화
3. **점진적 배포**: 전체 사용자 중 일부에게만 새 기능 노출

## 설정과 초기화

```dart
// main.dart
Future<(String, bool)> checkDemoMode(String environment) async {
  final remoteConfig = FirebaseRemoteConfig.instance;

  await remoteConfig.setConfigSettings(
    RemoteConfigSettings(
      fetchTimeout: const Duration(seconds: 3),
      minimumFetchInterval: const Duration(hours: 1),
    ),
  );

  // 기본값 설정 (fetch 실패 시 사용)
  await remoteConfig.setDefaults({
    'env_app_demo': '0',      // 0: 운영, 1: 데모
    'feature_automation': 'true',
    'feature_iot_dashboard': 'true',
    'max_devices_per_space': '10',
  });

  try {
    await remoteConfig.fetchAndActivate();
  } catch (e) {
    // fetch 실패해도 기본값으로 동작
    return (environment, false);
  }

  final isDemo = remoteConfig.getString('env_app_demo') == '1';
  // ...
}
```

## 값 읽기

```dart
class RemoteConfigService {
  final FirebaseRemoteConfig _config = FirebaseRemoteConfig.instance;

  bool get isAutomationEnabled {
    return _config.getBool('feature_automation');
  }

  bool get isScadaEnabled {
    return _config.getBool('feature_iot_dashboard');
  }

  int get maxDevicesPerSpace {
    return _config.getInt('max_devices_per_space');
  }

  String get serverEnvironment {
    return _config.getString('server_environment');
  }
}
```

## 기능 플래그로 조건부 UI

```dart
// 자동화 탭을 Remote Config로 제어
Obx(() {
  final showAutomation = _remoteConfig.isAutomationEnabled;
  
  return BottomNavigationBar(
    items: [
      const BottomNavigationBarItem(icon: Icon(Icons.home), label: '홈'),
      const BottomNavigationBarItem(icon: Icon(Icons.devices), label: '기기'),
      if (showAutomation)
        const BottomNavigationBarItem(
          icon: Icon(Icons.auto_awesome),
          label: '자동화',
        ),
      const BottomNavigationBarItem(icon: Icon(Icons.menu), label: '메뉴'),
    ],
  );
})
```

## 데모 모드 구현

앱 스토어 심사 시 심사자가 실제 IoT 기기 없이도 앱을 테스트할 수 있도록 데모 모드를 제공한다:

```dart
// main.dart에서 환경 결정
final bool callDemoMode;
if (environment.endsWith('_prod') && buildDate != 'unknown') {
  final buildDateTime = DateTime.tryParse(buildDate.replaceAll('_', 'T'));
  final diff = DateTime.now().difference(buildDateTime!).inDays;
  callDemoMode = diff <= 3;  // 빌드 후 3일 이내에만 데모 체크
} else {
  callDemoMode = false;
}

if (callDemoMode) {
  final (env, isDemo) = await checkDemoMode(environment);
  // isDemo가 true면 Mock 데이터로 동작하는 Stage 환경 사용
}
```

빌드 후 3일 이내의 프로덕션 빌드에서만 Remote Config를 체크한다. 앱 심사 기간(보통 1-3일) 내에 데모 모드가 켜져 있으면 심사 환경으로 전환된다.

## fetch 주기와 캐시

`minimumFetchInterval`은 fetch를 너무 자주 하지 않도록 제한한다. 기본값은 12시간이고, 개발 중에는 0으로 설정해서 즉시 반영할 수 있다:

```dart
await remoteConfig.setConfigSettings(
  RemoteConfigSettings(
    fetchTimeout: const Duration(seconds: 3),
    minimumFetchInterval: kDebugMode
        ? Duration.zero         // 개발 중
        : const Duration(hours: 1),  // 프로덕션
  ),
);
```

## 값 변경 감지

앱이 실행 중일 때 Remote Config 값이 바뀌면 실시간으로 감지할 수 있다:

```dart
remoteConfig.onConfigUpdated.listen((event) async {
  await remoteConfig.activate();
  // 변경된 키들: event.updatedKeys
  print('Remote Config 업데이트: ${event.updatedKeys}');
  // 필요한 경우 UI 갱신
});
```

---

다음 편은 글로벌 서비스를 위한 타임존 처리 전략이다.
