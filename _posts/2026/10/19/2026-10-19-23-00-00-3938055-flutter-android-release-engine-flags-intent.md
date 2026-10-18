---
layout: post
title: "Flutter Android release 오류 - Intent extras 엔진 플래그가 무시될 때"
description: "Flutter Android release에서 Intent extras로 넘긴 dart-flags와 route가 적용되지 않는 조건을 구분하고, AndroidManifest와 profile 빌드로 CI를 고치는 방법을 정리한다."
date: 2026-10-19
tags: [Flutter, Android, CI/CD, 테스트]
comments: true
share: true
---

![Flutter Android release 빌드의 엔진 플래그와 Intent 점검](https://images.unsplash.com/photo-1607252650355-f7fd0460ccdb?auto=format&fit=crop&w=1600&q=80)

이 그림에서 볼 부분은 앱 실행 뒤에 전달하는 값과 빌드 때 앱에 박아 넣는 설정이 서로 다른 경로라는 점이다.

Flutter Android release 앱을 `adb shell am start`나 자체 테스트 러너로 실행하면서 `Intent extras`에 `--dart-flags`, `--route` 같은 엔진 플래그를 넣었다면, 값이 적용되지 않는 것이 정상일 수 있다. 이 방식은 debug/profile에서는 쓸 수 있지만 release에서는 무시된다. 특히 `flutter run --release --use-application-binary`에 플래그를 붙이는 CI라면 오류를 숨기지 말고 빌드 단계 설정으로 옮겨야 한다. 일반적인 `flutter build appbundle`만 쓰고 엔진 플래그를 전달하지 않는 앱에는 해당하지 않는다.

## release에서 갑자기 route와 플래그가 사라지는 이유

Flutter Android 엔진은 release 실행 시 외부 `Intent`가 전달한 엔진 설정을 신뢰하지 않는다. Flutter 공식 문서가 설명하는 이유는 임의 앱이 실행 인자를 주입할 수 있는 보안 경계 때문이다. 따라서 다음처럼 debug에서 잘 되던 명령은 release에서 같은 결과를 보장하지 않는다.

```bash
adb shell am start \
  -n com.example.app/.MainActivity \
  --es dart-flags "--enable-asserts" \
  --es route "/diagnostic"
```

앱이 열리지만 진단 화면 대신 기본 route가 나오거나, `dart-flags`에 의존한 테스트가 조용히 건너뛰는 식으로 나타난다. 반면 Flutter CLI가 직접 Gradle release 빌드를 수행하면 필요한 값은 컴파일 시 `AndroidManifest.xml`에 주입할 수 있다.

| 실행 조건 | Intent extras | 대응 |
|---|---|---|
| debug/profile 앱 | 엔진 플래그 전달 가능 | 로컬 테스트·성능 측정에 사용 |
| 일반 release 빌드 | release 정책에 맞는 설정만 적용 | manifest 또는 코드에서 고정 |
| `--use-application-binary` release | 이미 빌드된 manifest를 바꿀 수 없음 | 플래그 제거 후 재빌드 |
| native `FlutterActivity` 직접 실행 | 외부 Intent 의존 금지 | 엔진 초기화 전 programmatic 설정 |

## 출시용 설정은 Manifest에 고정한다

flavor마다 같은 엔진 플래그를 재현해야 한다면 `application` 아래에 Flutter 엔진 메타데이터를 둔다. 아래 예시는 실제 패키지명과 플래그 이름을 프로젝트 요구에 맞게 바꿔야 한다.

```xml
<application
    android:name="${applicationName}"
    android:label="Example">

    <meta-data
        android:name="io.flutter.embedding.android.EnableImpeller"
        android:value="true" />
</application>
```

모든 플래그가 release에서 허용되는 것은 아니다. Flutter 엔진 플래그 문서의 `allowedInRelease` 정책에 걸리는 값은 manifest에 적어도 무시될 수 있다. 즉 “manifest에 넣었는데도 안 된다”면 XML 위치보다 해당 플래그가 release 허용 목록에 있는지부터 확인해야 한다.

route처럼 앱의 진입 화면을 바꾸려는 목적이라면 엔진 플래그에 기대지 않는 편이 안전하다. flavor의 `--dart-define=APP_ENV=staging`처럼 빌드 시 값을 넣고 Dart에서 읽거나, 테스트 전용 entrypoint를 별도 빌드 대상으로 나누는 방식이 release와 잘 맞는다.

```bash
# 값이 필요한 flavor를 빌드 시점에 고정한다.
flutter build appbundle --release \
  --flavor staging \
  --dart-define=APP_ENV=staging
```

## CI에서 놓치기 쉬운 확인 항목

`--use-application-binary`는 이미 만들어진 APK를 실행하는 옵션이다. 따라서 그 뒤에 붙인 엔진 플래그를 Flutter가 manifest에 다시 쓸 수 없다. 이 경우 성능 검증 목적이면 profile 빌드로 바꾸고, 실제 출시 산출물 검증이면 플래그 없이 서명된 binary를 실행한다.

```bash
# 동적 플래그가 필요한 자동화 검증
flutter build apk --profile
flutter run --profile --use-application-binary build/app/outputs/flutter-apk/app-profile.apk \
  --trace-startup

# 출시 산출물 검증: 엔진 플래그를 실행 시 전달하지 않는다.
flutter build appbundle --release --flavor production
```

체크리스트는 세 줄이면 충분하다.

- [ ] release 실행 스크립트가 `Intent.putExtra`나 `--es dart-flags`에 의존하지 않는다.
- [ ] flavor별 고정값은 `--dart-define` 또는 manifest merge로 빌드 시 기록된다.
- [ ] `--use-application-binary`를 쓰는 단계에는 동적 엔진 플래그가 없다.
- [ ] 동적 플래그가 꼭 필요한 테스트는 profile에서 수행한다.

핵심은 release 앱이 열리는지와 release 설정이 적용되는지를 따로 보는 것이다. Flutter 공식 기준으로 release는 Intent extras를 설정 채널로 사용할 수 없으며, profile은 동적 플래그를 유지한다. CI가 두 모드를 같은 실행 명령으로 취급하면 “빌드는 성공했지만 진단 화면이 안 열린다”는 형태의 운영 오류가 남는다. 설정을 빌드 시점에 고정하고, 동적 검증은 profile로 분리하면 이 혼동을 줄일 수 있다.

참고 문서: [Flutter release 엔진 플래그 제한](https://docs.flutter.dev/release/breaking-changes/restrict-android-engine-flags-release-mode), [Flutter Android 엔진 플래그 설정](https://github.com/flutter/flutter/blob/master/docs/engine/Flutter-Android-Engine-Flags.md), [FlutterEngineFlags API](https://api.flutter.dev/javadoc/io/flutter/embedding/engine/FlutterEngineFlags.Flag.html)
