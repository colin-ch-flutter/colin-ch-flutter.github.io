---
layout: post
title: "Flutter Firebase Test Lab - integration_test를 실기기 Farm에서 돌리기"
description: "Flutter Firebase Test Lab에서 integration_test를 Android 실기기 여러 조합으로 실행하는 방법과 IoT 앱의 BLE·Firebase 의존성을 격리하는 CI 구성을 정리했다."
date: 2026-08-04
tags: [Flutter, Firebase, integration_test, IoT, Android, CI/CD]
comments: true
share: true
---

![Flutter Firebase Test Lab에서 실행하는 IoT integration_test](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

Flutter Firebase Test Lab을 붙이면 integration_test를 내 컴퓨터의 에뮬레이터 하나가 아니라 Google 데이터센터의 여러 Android 기기 조합에서 확인할 수 있다. IoT 앱에서는 권한, 화면 크기, 앱 시작 순서 차이를 잡는 데 특히 효과가 있었다.

## 로컬 테스트와 Test Lab의 차이

로컬에서는 `flutter test integration_test/app_test.dart`가 통과해도 특정 기기에서만 앱이 시작되지 않을 수 있다. 보일러 앱을 Pixel 에뮬레이터에서만 돌렸을 때는 괜찮았는데, 작은 화면 기기에서는 카드가 잘리고 권한 요청 직후 테스트가 시작됐다.

| 대상 | 잘 잡히는 문제 | 한계 |
|---|---|---|
| Unit·Widget Test | 상태·화면 로직 | OS와 실제 앱 시작 흐름 없음 |
| 로컬 integration_test | 전체 화면 흐름 | 기기 조합이 제한됨 |
| Firebase Test Lab | OS·화면·기기별 차이 | 실행 비용과 대기 시간이 생김 |

## Android 빌드 준비

Test Lab은 Dart 테스트 파일을 직접 실행하지 않는다. Flutter 앱 APK와 Android instrumentation APK를 각각 만든 뒤 업로드한다. `androidTest`의 `MainActivityTest.kt`와 `FlutterTestRunner` 설정은 공식 README의 Android Device Testing 절차를 따른다.

테스트 대상은 `integration_test/smarthome_test.dart`라고 가정한다. 아래 명령은 앱 루트에서 실행한다.

```bash
# 앱 APK와 integration_test용 APK 생성
flutter build apk --debug

pushd android
./gradlew app:assembleAndroidTest
./gradlew app:assembleDebug \
  -Ptarget=../integration_test/smarthome_test.dart
popd
```

처음에는 `flutter build apk`만 하면 끝나는 줄 알았다. instrumentation 실행에는 두 APK가 모두 필요하다.

```text
build/app/outputs/apk/debug/app-debug.apk
build/app/outputs/apk/androidTest/debug/app-debug-androidTest.apk
```

## 기기 매트릭스에서 실행하기

`gcloud` 인증과 Firebase 프로젝트 설정이 끝났다면 기기 목록을 확인한다.

```bash
gcloud firebase test android models list
```

CI에서는 결과 버킷과 실행 디렉터리를 고정한다. 모델명과 API 버전은 지원 목록에 맞게 바꾼다.

```bash
gcloud firebase test android run \
  --type instrumentation \
  --app build/app/outputs/apk/debug/app-debug.apk \
  --test build/app/outputs/apk/androidTest/debug/app-debug-androidTest.apk \
  --device model=redfin,version=30,locale=ko_KR,orientation=portrait \
  --device model=panther,version=34,locale=ko_KR,orientation=portrait \
  --timeout 2m \
  --results-bucket=gs://my-flutter-test-results \
  --results-dir=smarthome-${GITHUB_SHA}
```

처음부터 10개 기기를 넣지 않는 편이 낫다. PR에서는 대표 기기 2개, 배포 브랜치에서 화면 크기와 API 조합을 늘리는 편이 비용과 피드백 속도의 균형이 좋았다.

## IoT 테스트에서 BLE를 그대로 연결하면 실패한다

Firebase Test Lab은 BLE 센서와 실제 보일러를 연결해주지 않는다. `BleService`와 `MqttService`를 주입하고 Test Lab에서는 Fake 구현을 사용해야 한다.

```dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  runApp(SmartHomeApp(
    bleService: const FakeBleService(),
    mqttService: FakeMqttService(),
  ));
}
```

이렇게 해야 네트워크 상태나 주변 BLE 장치 유무 때문에 flaky 해지지 않는다. 권한 팝업은 Patrol, 실제 장치 연결은 별도 실기기 테스트로 분리하는 게 맞다.

## CI에 넣을 때 걸린 부분

서비스 계정에 Test Lab 실행 권한과 결과 버킷 접근 권한이 모두 있어야 한다. `--results-dir`를 커밋 SHA와 연결하지 않으면 결과가 섞인다.

정리하면 다음 기준으로 나누면 된다.

- 상태·Repository 로직은 Unit Test
- 화면과 Fake BLE·MQTT 흐름은 Widget·integration_test
- OS, 화면 크기, API 버전 차이는 Firebase Test Lab
- 네이티브 권한 팝업과 실제 BLE 연결은 Patrol·실기기 Farm

Test Lab은 테스트를 대신 설계해주는 도구가 아니다. 테스트 격리와 APK 빌드를 먼저 정리한 뒤 기기 매트릭스를 추가해야, 실패 로그가 실제 제품 문제를 가리키게 된다.

참고:

- [Firebase Test Lab - Integration Testing with Flutter](https://firebase.google.com/docs/test-lab/flutter/integration-testing-with-flutter)
- [Flutter integration_test Android Device Testing](https://github.com/flutter/flutter/tree/main/packages/integration_test#firebase-test-lab)
