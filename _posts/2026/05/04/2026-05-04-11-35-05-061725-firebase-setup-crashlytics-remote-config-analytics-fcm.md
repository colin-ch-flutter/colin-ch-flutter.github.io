---
layout: post
title: "Firebase 한 번에 셋업하기 - Crashlytics, Remote Config, Analytics, FCM"
description: " "
date: 2026-05-04
tags: [Flutter, Firebase, FCM, Crashlytics, RemoteConfig]
comments: true
share: true
---
![Firebase 설정](https://images.unsplash.com/photo-1614064641938-3bbee52942c7?w=800&q=80)

## Firebase가 필요한 이유

IoT 앱에서 Firebase는 단순한 백엔드 서비스 이상의 역할을 한다. 크래시 리포트, 원격 설정 변경, 푸시 알림, 사용 분석... 이 모든 걸 Firebase 하나로 해결할 수 있다.

특히 Crashlytics는 필수다. 수백, 수천 명의 사용자가 쓰는 앱에서 특정 기기나 OS 버전에서만 발생하는 크래시를 로컬 테스트로는 발견하기 어렵다. Crashlytics가 실시간으로 크래시를 잡아준다.

## FlutterFire CLI로 빠르게 셋업

예전에는 Firebase 콘솔에서 `google-services.json`과 `GoogleService-Info.plist`를 수동으로 다운받아서 프로젝트에 추가해야 했다. 지금은 FlutterFire CLI를 쓰면 훨씬 편하다:

```bash
dart pub global activate flutterfire_cli
flutterfire configure
```

실행하면 Firebase 프로젝트를 선택하고, 플랫폼을 선택하면 `firebase_options.dart`를 자동으로 생성해준다. 이 파일에 모든 플랫폼 설정이 들어있다.

## main.dart에서 Firebase 초기화

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Firebase 초기화
  await Firebase.initializeApp(
    options: DefaultFirebaseOptions.currentPlatform,
  );
  
  // FCM 백그라운드 핸들러 등록 (반드시 top-level 함수)
  FirebaseMessaging.onBackgroundMessage(firebaseMessagingBackgroundHandler);
  
  runApp(MyApp());
}

// top-level 함수여야 한다
@pragma('vm:entry-point')
Future<void> firebaseMessagingBackgroundHandler(RemoteMessage message) async {
  await Firebase.initializeApp();
  // 백그라운드 메시지 처리
}
```

## Crashlytics 설정

Crashlytics는 설정이 간단하다:

```dart
// FlutterError를 Crashlytics로 전달
FlutterError.onError = FirebaseCrashlytics.instance.recordFlutterFatalError;

// Dart 비동기 에러도 잡기
PlatformDispatcher.instance.onError = (error, stack) {
  FirebaseCrashlytics.instance.recordError(error, stack, fatal: true);
  return true;
};
```

이걸 설정해두면 앱에서 처리되지 않은 모든 예외가 Firebase 콘솔에 보고된다. 실제로 출시 후 특정 Android 버전에서만 발생하는 버그를 Crashlytics로 발견한 경우가 있었다.

개발 중에는 Crashlytics를 비활성화해두는 게 좋다:

```dart
await FirebaseCrashlytics.instance
    .setCrashlyticsCollectionEnabled(!kDebugMode);
```

## Remote Config 활용

Remote Config의 가장 유용한 활용 사례는 **앱 스토어 심사 대응**이었다. 심사 중에는 실제 서버 대신 데모 환경으로 동작하도록 Remote Config로 전환한다:

```dart
final remoteConfig = FirebaseRemoteConfig.instance;

await remoteConfig.setConfigSettings(RemoteConfigSettings(
  fetchTimeout: const Duration(seconds: 3),
  minimumFetchInterval: const Duration(hours: 1),
));

await remoteConfig.setDefaults({'env_app_demo': '0'});
await remoteConfig.fetchAndActivate();

final isDemo = remoteConfig.getString('env_app_demo') == '1';
```

서버 배포 없이 Firebase 콘솔에서 값 하나만 바꾸면 앱 동작이 바뀐다.

## Analytics 이벤트

사용자가 어떤 기능을 얼마나 쓰는지 파악하는 데 Analytics를 쓴다:

```dart
// 기기 제어 이벤트 기록
await FirebaseAnalytics.instance.logEvent(
  name: 'iot_device_power_toggle',
  parameters: {
    'device_id': deviceId,
    'action': isOn ? 'turn_on' : 'turn_off',
  },
);
```

어떤 화면에서 사용자가 이탈하는지, 어떤 기능을 주로 쓰는지 데이터로 볼 수 있다.

## FCM 기본 설정

FCM 초기화와 토큰 획득:

```dart
// PushNotificationService 초기화
final messaging = FirebaseMessaging.instance;

// iOS 권한 요청
await messaging.requestPermission(
  alert: true,
  badge: true,
  sound: true,
);

// FCM 토큰 획득
final token = await messaging.getToken();
// 이 토큰을 서버에 등록해두면 서버에서 특정 기기로 푸시 가능

// 토큰 갱신 이벤트
messaging.onTokenRefresh.listen((newToken) {
  // 서버에 새 토큰 업데이트
});
```

FCM 상세 처리(포그라운드/백그라운드/종료 상태)는 나중에 별도 편에서 자세히 다룬다.

## 멀티 환경 Firebase 설정

개발/스테이징/프로덕션 환경마다 Firebase 프로젝트를 분리하는 게 좋다. `--dart-define`으로 환경을 주입받아서 해당 환경의 Firebase 옵션을 선택한다:

```dart
// firebase_options.dart를 환경별로 분리
FirebaseOptions get currentPlatform {
  switch (environment) {
    case 'prod': return productionOptions;
    case 'stage': return stagingOptions;
    default: return developmentOptions;
  }
}
```

이렇게 하면 개발 중 발생한 크래시가 프로덕션 Crashlytics에 섞이지 않는다.

---

다음 편부터는 BLE 연결 파트가 시작된다. 개인적으로 이 프로젝트에서 가장 어려웠던 부분이다.
