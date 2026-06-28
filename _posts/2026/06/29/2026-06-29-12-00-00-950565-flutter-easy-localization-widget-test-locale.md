---
layout: post
title: "easy_localization 위젯 테스트 — Locale 격리와 다국어 UI 버그 잡기"
description: "easy_localization을 사용하는 Flutter 위젯을 테스트 환경에서 Locale을 주입하는 방법과, ko/en 전환 시 발생하는 UI 버그를 위젯 테스트로 잡는 실전 패턴을 정리했다."
date: 2026-06-29
tags: [Flutter, easy_localization, 다국어, UX, CleanArchitecture]
comments: true
share: true
---

# easy_localization 위젯 테스트 — Locale 격리와 다국어 UI 버그 잡기

![Flutter 다국어 위젯 테스트 Locale 격리](https://images.unsplash.com/photo-1546410531-bb4caa6b424d?w=800&q=80)

`easy_localization`을 쓰는 위젯을 테스트하려고 하면 처음에 이런 오류가 난다. `'package:easy_localization/src/easy_localization_controller.dart': Failed assertion: line 282`. 번역 파일을 못 찾겠다는 건데, 이걸 해결하려고 `flutter_test`의 `rootBundle`을 건드리다가 시간을 날렸다. 핵심은 테스트 전용 `AssetLoader`를 만들어서 실제 json 파일 없이 번역 맵을 메모리에서 직접 제공하는 것이다.

## 문제 상황

스마트홈 앱에서 한국어·영어·러시아어를 지원하는데, 언어 전환 UI가 있다. `SettingsScreen`에서 언어를 바꾸면 앱 전체가 즉시 갱신돼야 한다. 이걸 수동으로 테스트하면 매번 앱 빌드 → 설정 화면 진입 → 언어 변경 → 결과 확인 과정을 반복해야 한다.

실제로 놓쳤던 버그가 있다. 한국어에서는 '보일러 제어' 버튼이 한 줄로 나오는데, 러시아어로 바꾸면 버튼 텍스트가 길어서 두 줄이 되고 레이아웃이 깨졌다. `ko`로만 테스트하니 못 잡았고, 배포 후에 러시아 사용자 피드백으로 알게 됐다.

이런 종류의 버그는 Locale별 위젯 테스트로 사전에 잡을 수 있다.

## 테스트용 AssetLoader 만들기

`easy_localization`이 번역 데이터를 가져오는 방식은 `AssetLoader` 인터페이스를 통해 추상화돼 있다. 프로덕션에서는 `assets/translations/ko.json` 같은 파일을 읽는데, 테스트에서는 메모리 맵으로 대체한다.

```dart
import 'package:easy_localization/easy_localization.dart';

class FakeAssetLoader extends AssetLoader {
  final Map<String, dynamic> translations;

  FakeAssetLoader({required this.translations});

  @override
  Future<Map<String, dynamic>> load(String path, Locale locale) async {
    return translations;
  }
}
```

간단하다. `path`와 `locale`을 무시하고 주입한 맵을 돌려준다. 테스트에서는 필요한 번역 키만 넣으면 된다.

## 위젯 테스트 기본 구조

`easy_localization`이 동작하려면 위젯 트리 상단에 `EasyLocalization` 위젯이 있어야 한다. `tester.pumpWidget()`에 래퍼를 씌우는 방식으로 처리한다.

```dart
Widget buildTestableWidget(Widget child, {Locale locale = const Locale('ko', 'KR')}) {
  return EasyLocalization(
    supportedLocales: const [
      Locale('ko', 'KR'),
      Locale('en', 'US'),
      Locale('ru', 'RU'),
    ],
    path: 'assets/translations',
    assetLoader: FakeAssetLoader(
      translations: {
        'boiler_control': locale.languageCode == 'ko'
            ? '보일러 제어'
            : locale.languageCode == 'ru'
                ? 'Управление котлом'
                : 'Boiler Control',
        'settings': locale.languageCode == 'ko' ? '설정' : 'Settings',
        // 테스트에 필요한 키만 추가
      },
    ),
    startLocale: locale,
    child: MaterialApp(home: child),
  );
}
```

`startLocale`로 테스트할 언어를 지정한다. 테스트마다 `locale`만 바꿔서 같은 시나리오를 여러 언어로 검증할 수 있다.

## 실제 테스트 코드

`setUp`에서 `EasyLocalization.ensureInitialized()`를 반드시 호출해야 한다. 이게 없으면 `EasyLocalization` 위젯이 초기화 전에 번역을 요청해서 예외가 난다.

```dart
void main() {
  setUpAll(() async {
    await EasyLocalization.ensureInitialized();
  });

  group('BoilerControlButton — 다국어 레이아웃', () {
    testWidgets('한국어: 버튼 한 줄 렌더링', (tester) async {
      await tester.pumpWidget(
        buildTestableWidget(
          const BoilerControlButton(),
          locale: const Locale('ko', 'KR'),
        ),
      );
      await tester.pumpAndSettle();

      final button = tester.widget<ElevatedButton>(find.byType(ElevatedButton));
      final textWidget = tester.widget<Text>(find.text('보일러 제어'));
      expect(textWidget.maxLines, isNull); // 줄 수 제한 없는데 실제로 1줄이어야 함
      expect(find.text('보일러 제어'), findsOneWidget);
    });

    testWidgets('러시아어: 버튼 텍스트가 잘리지 않아야 한다', (tester) async {
      tester.binding.window.physicalSizeTestValue = const Size(375, 812);
      tester.binding.window.devicePixelRatioTestValue = 2.0;
      addTearDown(tester.binding.window.clearPhysicalSizeTestValue);

      await tester.pumpWidget(
        buildTestableWidget(
          const BoilerControlButton(),
          locale: const Locale('ru', 'RU'),
        ),
      );
      await tester.pumpAndSettle();

      expect(find.text('Управление котлом'), findsOneWidget);

      // 텍스트가 overflow 없이 렌더링됐는지 확인
      expect(tester.takeException(), isNull);
    });
  });
}
```

화면 크기를 `physicalSizeTestValue`로 고정하는 게 포인트다. 기본 테스트 환경의 화면 크기가 실기기와 달라서 러시아어 버튼이 잘리는 버그를 못 잡는 경우가 있다.

![Flutter 위젯 테스트 다국어 Locale 전환](https://images.unsplash.com/photo-1555099962-4199c345e5dd?w=800&q=80)

## 언어 전환 자체를 테스트하기

런타임에 언어를 바꾸는 동작도 테스트할 수 있다. `context.setLocale()`을 호출한 뒤 `pumpAndSettle()`로 리빌드를 기다린다.

```dart
testWidgets('언어 전환 시 텍스트 즉시 변경', (tester) async {
  late BuildContext capturedContext;

  await tester.pumpWidget(
    buildTestableWidget(
      Builder(
        builder: (context) {
          capturedContext = context;
          return const SettingsLanguageSelector();
        },
      ),
      locale: const Locale('ko', 'KR'),
    ),
  );
  await tester.pumpAndSettle();

  expect(find.text('설정'), findsOneWidget);

  // 영어로 전환
  capturedContext.setLocale(const Locale('en', 'US'));
  await tester.pumpAndSettle();

  expect(find.text('Settings'), findsOneWidget);
  expect(find.text('설정'), findsNothing);
});
```

`Builder`로 `BuildContext`를 꺼내는 게 좀 번거롭긴 한데, `EasyLocalization` 컨텍스트 안에 있는 `context`여야 `setLocale()`이 동작한다. `tester.element(find.byType(MaterialApp))` 같은 방식으로 꺼내도 되는데, `Builder` 쪽이 의도가 더 명확하다.

## 삽질했던 부분

**번역 키가 없을 때 오류 vs 키 이름 반환**: 기본 설정에서 `easy_localization`은 번역 키가 없으면 키 이름 자체를 반환한다. `FakeAssetLoader`에 키를 빠뜨리면 위젯에 `'boiler_control'` 문자열이 그대로 표시된다. 테스트는 통과하는데 실제로는 깨진 UI다. `FakeAssetLoader`에 쓸 키 목록을 위젯 테스트 대상 화면 기준으로 꼼꼼히 작성해야 한다.

**`pumpAndSettle()`이 타임아웃 나는 경우**: `EasyLocalization`이 초기화하면서 내부에 애니메이션이나 Future가 있으면 `pumpAndSettle()`이 타임아웃(기본 100ms × 1000회)을 초과할 수 있다. 이럴 때는 `pump(const Duration(seconds: 1))`를 몇 번 호출하는 식으로 수동으로 시간을 흘려보내는 게 낫다.

**Golden Test와 폰트 불일치**: 다국어 Golden Test는 시스템 폰트에 따라 렌더링이 달라진다. `flutter_test` 환경의 기본 폰트와 실기기 폰트가 달라서 픽셀 단위로 비교하면 CI에서 실패한다. 이 문제는 [Golden Test 편]({% post_url 2026-06-25-16-00-00-716010-flutter-golden-test-ui-snapshot-iot %})에서 다뤘던 `alchemist` + `matchesGoldenFile` 설정으로 폰트를 고정하면 해결된다.

## 번역 맵을 헬퍼로 추출하기

테스트가 늘어나면 각 테스트마다 번역 맵을 중복 작성하게 된다. 공용 `TestTranslations` 클래스로 뽑아두는 게 유지보수에 낫다.

```dart
// test/helpers/test_translations.dart
class TestTranslations {
  static Map<String, dynamic> korean = {
    'boiler_control': '보일러 제어',
    'settings': '설정',
    'connect': '연결',
    'disconnect': '연결 해제',
    'device_not_found': '기기를 찾을 수 없습니다',
  };

  static Map<String, dynamic> english = {
    'boiler_control': 'Boiler Control',
    'settings': 'Settings',
    'connect': 'Connect',
    'disconnect': 'Disconnect',
    'device_not_found': 'Device not found',
  };

  static Map<String, dynamic> russian = {
    'boiler_control': 'Управление котлом',
    'settings': 'Настройки',
    'connect': 'Подключить',
    'disconnect': 'Отключить',
    'device_not_found': 'Устройство не найдено',
  };

  static Map<String, dynamic> forLocale(Locale locale) {
    switch (locale.languageCode) {
      case 'ru': return russian;
      case 'en': return english;
      default: return korean;
    }
  }
}
```

이렇게 만들어두면 테스트에서 `FakeAssetLoader(translations: TestTranslations.forLocale(locale))` 한 줄로 끝난다. easy_localization 구현 자체는 {% post_url 2026-05-03-10-28-52-049380-easy-localization-multi-language-korean-english-russian %}에서 다뤘다.

---

다음 편은 `fl_chart`로 그린 IoT 센서 데이터 차트를 위젯 테스트로 검증하는 방법이다. 차트 라이브러리는 픽셀 렌더링에 의존하는 부분이 많아서 일반적인 `find` 기반 테스트가 잘 안 된다. 데이터 레이어와 차트 위젯을 분리해서 테스트 가능하게 만드는 설계 패턴을 다룬다.

**참고 링크**
- [easy_localization pub.dev](https://pub.dev/packages/easy_localization)
- [Flutter WidgetTester 공식 문서](https://api.flutter.dev/flutter/flutter_test/WidgetTester-class.html)
- [EasyLocalization AssetLoader 소스](https://github.com/aissat/easy_localization/blob/develop/lib/src/asset_loader.dart)
