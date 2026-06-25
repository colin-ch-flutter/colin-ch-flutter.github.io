---
layout: post
title: "다크모드 완벽 지원 - ThemeData 설계와 동적 전환"
description: " "
date: 2026-05-26
tags: [Flutter, 다크모드, ThemeData, UI, 테마]
comments: true
share: true
---
![다크 테마 UI](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=800&q=80)

## ThemeData 구조 설계

다크모드를 나중에 추가하면 엄청난 수정이 필요하다. 처음부터 테마 시스템을 잘 설계해두는 게 중요하다.

```dart
// presentation/core/theme/app_theme.dart
class AppTheme {
  static ThemeData get light => ThemeData(
    colorScheme: ColorScheme.fromSeed(
      seedColor: const Color(0xFF0066CC),
      brightness: Brightness.light,
    ),
    useMaterial3: true,
    fontFamily: 'SpoqaHanSansNeo',
    textTheme: _textTheme(Brightness.light),
    appBarTheme: const AppBarTheme(
      backgroundColor: Colors.white,
      foregroundColor: Color(0xFF1A1A1A),
      elevation: 0,
    ),
  );

  static ThemeData get dark => ThemeData(
    colorScheme: ColorScheme.fromSeed(
      seedColor: const Color(0xFF0066CC),
      brightness: Brightness.dark,
    ),
    useMaterial3: true,
    fontFamily: 'SpoqaHanSansNeo',
    textTheme: _textTheme(Brightness.dark),
    appBarTheme: const AppBarTheme(
      backgroundColor: Color(0xFF1C1C1E),
      foregroundColor: Colors.white,
      elevation: 0,
    ),
  );

  static TextTheme _textTheme(Brightness brightness) {
    final color = brightness == Brightness.dark ? Colors.white : const Color(0xFF1A1A1A);
    return TextTheme(
      displayLarge: TextStyle(
        fontSize: 48,
        fontWeight: FontWeight.bold,
        color: color,
      ),
      titleLarge: TextStyle(
        fontSize: 20,
        fontWeight: FontWeight.w600,
        color: color,
      ),
      bodyMedium: TextStyle(fontSize: 14, color: color),
    );
  }
}
```

## GetMaterialApp에 테마 적용

```dart
GetMaterialApp(
  theme: AppTheme.light,
  darkTheme: AppTheme.dark,
  themeMode: ThemeMode.system, // 시스템 설정 따름
  // ...
)
```

`ThemeMode.system`으로 설정하면 기기의 다크모드 설정을 자동으로 따른다.

## 테마에 따른 이미지 에셋 교체

아이콘이나 이미지가 라이트/다크 모드에 따라 달라야 할 때:

```dart
// presentation/core/theme/theme_util.dart
class ThemeUtil {
  static String getThemedAsset(BuildContext context, String assetName) {
    final isDark = Theme.of(context).brightness == Brightness.dark;
    final suffix = isDark ? '_dark' : '_light';
    return 'assets/images/$assetName$suffix.png';
  }

  // 앱 시작 시 미리 캐싱
  static Future<void> precacheThemedAssets(
    List<String> assetNames,
  ) async {
    // main.dart에서 호출됨
    // 테마에 따른 이미지를 미리 로드해서 첫 표시 시 깜빡임 방지
  }
}
```

실제 앱에서는 `main.dart`에서 자주 쓰는 이미지를 미리 캐시한다:

```dart
await ThemeUtil.precacheThemedAssets([
  'ic_more_28',
  'img_background',
  'ic_check_default_24',
  // ...
]);
```

## 동적 테마 전환

사용자가 앱 안에서 라이트/다크를 수동 전환할 때:

```dart
class ThemeController extends GetxController {
  final Rx<ThemeMode> themeMode = ThemeMode.system.obs;

  void setTheme(ThemeMode mode) {
    themeMode.value = mode;
    // 저장
    GetStorage().write('theme_mode', mode.index);
    // GetX로 즉시 적용
    Get.changeThemeMode(mode);
  }

  void init() {
    final savedIndex = GetStorage().read<int>('theme_mode');
    if (savedIndex != null) {
      themeMode.value = ThemeMode.values[savedIndex];
      Get.changeThemeMode(themeMode.value);
    }
  }
}
```

## 다크모드에서 발견한 버그들

다크모드를 나중에 추가하면서 생긴 문제들이다.

**하드코딩된 색상**: `Colors.white`나 `Colors.black`을 직접 쓴 곳이 있으면 다크모드에서 배경과 같아져 안 보인다. `Theme.of(context).colorScheme.surface` 같은 테마 색상으로 바꿔야 한다.

**이미지 배경**: 흰 배경 이미지가 다크모드에서 이상하게 보이는 경우. `ColorFiltered`로 색상을 반전시키거나 다크 버전 이미지를 별도로 준비한다.

**Dialog 배경**: 커스텀 다이얼로그의 배경색을 하드코딩했다가 다크모드에서 눈에 띈 경우가 있었다.

---

다음 편은 온도 스케줄 설정 UI 구현이다.
