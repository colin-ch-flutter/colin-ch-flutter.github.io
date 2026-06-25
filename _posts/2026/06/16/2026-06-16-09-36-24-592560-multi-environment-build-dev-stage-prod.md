---
layout: post
title: "멀티 환경 빌드 설정 - dev/stage/prod 완전 분리"
description: " "
date: 2026-06-16
tags: [Flutter, 빌드설정, 환경분리, CI/CD, dart-define]
comments: true
share: true
---
![빌드 설정](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

## 왜 환경을 분리해야 하는가

개발할 때 실수로 프로덕션 서버에 테스트 데이터를 넣거나, 개발 중 발생한 크래시가 프로덕션 Crashlytics에 쌓이는 일이 생기면 곤란하다.

환경을 분리하면:
- **dev**: 개발 서버, 테스트 계정, 로그 상세
- **stage**: 프로덕션과 동일 환경, 심사/QA용
- **prod**: 실제 서비스

## dart-define으로 환경 주입

Flutter는 빌드 시 `--dart-define`으로 상수를 주입할 수 있다:

```bash
# 개발 환경
flutter run --dart-define=CONFIG=dev --dart-define=BUILD_DATE=2025_01_01T00:00:00

# 스테이지 환경
flutter build apk --dart-define=CONFIG=stage

# 프로덕션 환경
flutter build apk --dart-define=CONFIG=prod
```

코드에서 읽기:
```dart
const String environment = String.fromEnvironment('CONFIG', defaultValue: 'dev');
const String buildDate = String.fromEnvironment('BUILD_DATE', defaultValue: 'unknown');
```

## Config 클래스 설계

각 환경의 설정을 Config 클래스로 관리한다:

```dart
// core/config/config.dart
class Config {
  final String apiBaseUrl;
  final String mqttEndpoint;
  final String mqttRegion;
  final bool isTestMode;
  final bool debugLogging;

  const Config({
    required this.apiBaseUrl,
    required this.mqttEndpoint,
    required this.mqttRegion,
    this.isTestMode = false,
    this.debugLogging = false,
  });
}

Config getConfig(String env, {bool isDemo = false}) {
  return switch (env) {
    'dev' => Config(
        apiBaseUrl: 'https://api-dev.smarthome.com',
        mqttEndpoint: 'xxx-dev.iot.ap-northeast-2.amazonaws.com',
        mqttRegion: 'ap-northeast-2',
        isTestMode: true,
        debugLogging: true,
      ),
    'stage' => Config(
        apiBaseUrl: 'https://api-stage.smarthome.com',
        mqttEndpoint: 'xxx-stage.iot.ap-northeast-2.amazonaws.com',
        mqttRegion: 'ap-northeast-2',
        debugLogging: true,
      ),
    'prod' => Config(
        apiBaseUrl: 'https://api.smarthome.com',
        mqttEndpoint: 'xxx.iot.ap-northeast-2.amazonaws.com',
        mqttRegion: 'ap-northeast-2',
      ),
    'global_prod' => Config(
        apiBaseUrl: 'https://api-global.smarthome.com',
        mqttEndpoint: 'xxx.iot.us-east-1.amazonaws.com',
        mqttRegion: 'us-east-1',
      ),
    _ => throw ArgumentError('알 수 없는 환경: $env'),
  };
}
```

## build_config 디렉터리

`build_config/` 디렉터리에 환경별 설정 파일이 있다:

```
build_config/
├── domestic/
│   └── android_version.txt    # Android 버전 정보
└── global/
    └── android_version.txt
```

CI/CD에서 빌드 시 이 파일들을 읽어서 버전 코드를 설정한다.

## ConfigManager로 전역 접근

```dart
// core/config/config_manager.dart
class ConfigManager extends GetxController {
  late Config _config;

  void setConfig(Config config) {
    _config = config;
  }

  Config get config => _config;
  
  String get apiBaseUrl => _config.apiBaseUrl;
  bool get isTestMode => _config.isTestMode;
  bool get debugLogging => _config.debugLogging;
}
```

어디서든 `Get.find<ConfigManager>().config`로 현재 환경 설정에 접근할 수 있다.

## 환경별 앱 아이콘

환경이 다른 빌드를 같이 설치했을 때 구분하기 위해 아이콘을 다르게 설정한다:

```yaml
# pubspec.yaml
flutter_icons:
  android: true
  ios: true
  image_path: "assets/icon/app_icon.png"
  # 환경별로 다른 아이콘을 쓰려면 build flavor 설정 필요
```

개발팀 내부에서는 dev 빌드에 "DEV" 배너가 붙은 아이콘을 쓰면 혼동을 피할 수 있다.

## CI/CD에서의 환경 분리

GitHub Actions 예시:

```yaml
jobs:
  build-prod:
    steps:
      - name: Build Production APK
        run: |
          flutter build apk \
            --dart-define=CONFIG=prod \
            --dart-define=BUILD_DATE=$(date -u +%Y_%m_%dT%H:%M:%S) \
            --release
```

---

다음 편은 앱 스토어 배포 준비 전 과정이다.
