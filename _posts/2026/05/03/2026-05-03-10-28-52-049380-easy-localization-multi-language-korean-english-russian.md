---
layout: post
title: "Easy Localization으로 3개 국어 지원하기 (한국어/영어/외국어)"
description: " "
date: 2026-05-03
tags: [Flutter, i18n, easy_localization, 다국어]
comments: true
share: true
---
![세계 언어](https://images.unsplash.com/photo-1526378800651-2e3b8c7cb34b?w=800&q=80)

## 다국어 지원이 필수인 이유

스마트 IoT 기기는 국내뿐만 아니라 해외 시장에도 나가는 제품이다. 같은 앱을 한국어, 영어, 외국어로 제공해야 했다. 처음에는 "그냥 문자열만 바꾸면 되는 거 아니야?"라고 생각했는데, 실제로 해보니 언어마다 UI 레이아웃에 영향을 주는 경우가 꽤 있었다.

외국어는 단어가 유독 길다. 영어로 "Settings"가 외국어로 "Настройки"가 되는데, 이게 버튼 안에 들어가면 넘치는 경우가 생긴다. 이런 부분을 처리하는 게 생각보다 손이 많이 갔다.

## easy_localization 설정

`pubspec.yaml`에 의존성을 추가하고 assets 경로를 지정한다:

```yaml
dependencies:
  easy_localization: ^3.0.0

flutter:
  assets:
    - assets/lang/
```

`assets/lang/` 디렉터리에 언어별 JSON 파일을 만든다:

```
assets/lang/
├── en_US.json
├── ko_KR.json
└── ru_RU.json
```

## JSON 구조

JSON 파일은 키-값 형태로 번역 문자열을 관리한다:

```json
// ko_KR.json
{
  "common": {
    "confirm": "확인",
    "cancel": "취소",
    "save": "저장",
    "loading": "로딩 중..."
  },
  "iot_device": {
    "power_on": "전원 켜기",
    "power_off": "전원 끄기",
    "temperature": "온도",
    "mode": {
      "heating": "난방",
      "hot_water": "온수",
      "away": "외출"
    }
  },
  "error": {
    "network": "네트워크 연결을 확인해 주세요",
    "device_offline": "기기가 오프라인 상태입니다"
  }
}
```

계층 구조로 키를 관리하면 나중에 찾기 쉽다.

## main.dart 설정

```dart
void main() async {
  await EasyLocalization.ensureInitialized();
  EasyLocalization.logger.enableLevels = []; // 로그 끄기

  runApp(
    EasyLocalization(
      supportedLocales: const [
        Locale('en', 'US'),
        Locale('ko', 'KR'),
        Locale('ru', 'RU'),
      ],
      path: 'assets/lang',
      fallbackLocale: const Locale('en', 'US'),
      startLocale: initialLocale,
      child: MyApp(),
    ),
  );
}
```

`fallbackLocale`을 영어로 설정해두면 번역 키가 없을 때 영어로 폴백된다.

## 코드에서 사용하기

```dart
// 단순 번역
Text('common.confirm'.tr())

// 변수 삽입
Text('iot_device.temperature_label'.tr(args: ['22']))
// JSON: "temperature_label": "현재 온도: {}°C"

// 복수형 처리
Text('device.count'.plural(devices.length))
// JSON: "count": {"one": "기기 1대", "other": "기기 {}대"}
```

## 동적 언어 변경

앱 실행 중에 언어를 변경할 수 있다:

```dart
// 한국어로 변경
await context.setLocale(const Locale('ko', 'KR'));

// 또는 GetX + EasyLocalization 조합
Get.updateLocale(const Locale('ru', 'RU'));
```

언어 변경 후 앱을 재시작하지 않아도 즉시 반영된다.

## 로케일 감지 로직

앱 첫 실행 시 기기의 시스템 언어를 감지해서 적절한 로케일을 자동 선택한다:

```dart
// core/utils/locale_util.dart
Locale getInitialLocale() {
  final systemLocale = WidgetsBinding.instance.platformDispatcher.locale;
  final supported = ['ko', 'en', 'ru'];
  
  if (supported.contains(systemLocale.languageCode)) {
    return systemLocale;
  }
  return const Locale('en', 'US'); // 기본값
}
```

## 외국어에서 겪은 문제들

외국어 텍스트를 처음 적용했을 때 몇 가지 문제가 있었다.

**텍스트 오버플로우**: 외국어 단어가 영어보다 길어서 버튼 레이블이 잘리는 현상이 있었다. `TextOverflow.ellipsis`나 `FittedBox`로 처리했다.

**폰트 지원**: 기본 폰트(SpoqaHanSansNeo)가 키릴 문자를 지원하지 않는 경우가 있었다. 폰트 파일에 키릴 문자 범위가 포함되어 있는지 확인이 필요했다.

**날짜 포맷**: intl 패키지의 DateFormat이 언어에 따라 자동으로 포맷을 바꿔주는데, 외국어 로케일도 정상적으로 동작하는지 확인이 필요했다.

---

다음 편은 Firebase를 한 번에 셋업하는 과정이다.
