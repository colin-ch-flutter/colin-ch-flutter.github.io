---
layout: post
title: "Flutter 3.50 WidgetSpan RTL 오류 - 뒤집힌 인라인 위젯과 golden 테스트 고치기"
description: "Flutter 3.50에서 아랍어·히브리어 RTL 텍스트 안의 WidgetSpan 순서가 바뀌는 이유와 기존 보정 코드 제거, golden 테스트 업데이트 기준을 정리한다."
date: 2026-11-14
tags: [Flutter, Dart, 테스트, 접근성, RTL]
comments: true
share: true
---

![Flutter 3.50 RTL WidgetSpan 레이아웃과 golden 테스트 점검](https://images.unsplash.com/photo-1551650975-87deedd944c3?auto=format&fit=crop&w=1600&q=80)

Flutter 3.50으로 올린 뒤 아랍어·히브리어 화면의 배지, 아이콘, 링크가 반대 위치로 이동했다면 `WidgetSpan`을 다시 뒤집을 문제가 아니다. 기존에 RTL 순서를 보정하던 `Transform.translate`, span 배열 역순 처리, placeholder 좌표 반전 코드를 제거하고 논리적인 읽기 순서로 되돌리는 선택이 맞다. `WidgetSpan`을 쓰지 않거나 RTL 보정 코드를 넣은 적이 없는 앱은 이 변경으로 수정할 필요가 없다.

## 문제가 생기는 조건

Flutter 3.50부터 텍스트 안의 `WidgetSpan` 위치가 소스 순서가 아니라 Unicode 양방향 알고리즘의 시각 순서를 따른다. 이전 엔진은 아랍어 문장 안에서도 placeholder를 Dart 코드에 적은 순서로 배치했다. 그래서 개발자가 위젯을 역순으로 넣거나 x 좌표를 수동으로 옮겨 화면을 맞추는 경우가 있었다.

이제 엔진이 올바른 위치를 계산한다. 예전 우회 코드를 그대로 두면 보정이 두 번 적용되어 아이콘이 다시 반대쪽으로 간다.

| 코드 상태 | Flutter 3.49 이하 | Flutter 3.50 | 조치 |
|---|---|---|---|
| WidgetSpan만 사용 | 잘못된 RTL 위치 가능 | 시각 순서로 배치 | 코드 유지 |
| span 배열을 역순으로 생성 | 임시로 화면 보정 | 순서가 다시 뒤집힘 | 논리 순서로 복원 |
| Transform으로 x 좌표 보정 | 임시 위치 보정 | 이중 보정 | Transform 제거 |
| golden이 기존 화면 캡처 | 기존 결과 통과 | 새 위치에서 실패 | RTL golden 갱신 |

## 보정 코드를 논리 순서로 바꾸기

인라인 아이콘 두 개를 아랍어 문장에 넣는 예시다. 실제 프로젝트에서는 문자열을 하드코딩하지 말고 번역 리소스와 span 조합을 같은 규칙으로 관리한다.

```dart
Text.rich(
  TextSpan(
    text: 'حالة الجهاز ',
    children: [
      WidgetSpan(
        alignment: PlaceholderAlignment.middle,
        child: const Icon(Icons.wifi, size: 16),
      ),
      const TextSpan(text: ' متصل '),
      WidgetSpan(
        alignment: PlaceholderAlignment.middle,
        child: const Icon(Icons.check_circle, size: 16),
      ),
    ],
  ),
  textDirection: TextDirection.rtl,
)
```

여기서 `WidgetSpan`의 배열을 RTL이라는 이유로 뒤집지 않는다. `Transform.translate`로 아이콘을 이동시키거나 `GlobalKey`로 placeholder의 x 좌표를 다시 계산하는 코드도 같은 기준으로 제거한다. Flutter 공식 문서의 변경 설명처럼 Dart 코드에는 콘텐츠의 논리 순서를 남기고, 레이아웃이 반환한 위치를 그대로 사용해야 한다.

커스텀 렌더러가 `TextPainter`의 placeholder 좌표를 직접 소비한다면 좌표를 좌우 반전하는 부분을 특히 확인한다. 좌표 배열을 별도 캐시에 저장하고 있다면 캐시 키에 Flutter SDK 버전을 넣을 문제가 아니라, 보정 계산 자체를 없애야 한다.

## golden 테스트는 두 방향을 분리한다

실제 golden 파일을 실행했다고 가정하지 않고, 출시 전에는 아래 조건을 별도 케이스로 추가한다. 한 줄 텍스트만 보면 위치 오류를 놓치므로 줄바꿈과 LTR 숫자가 섞인 문장을 포함해야 한다.

```dart
testWidgets('RTL WidgetSpan keeps logical span order', (tester) async {
  await tester.pumpWidget(
    const Directionality(
      textDirection: TextDirection.rtl,
      child: Material(
        child: Center(child: DeviceStatusLabel()),
      ),
    ),
  );

  await expectLater(
    find.byType(DeviceStatusLabel),
    matchesGoldenFile('goldens/device-status-rtl.png'),
  );
});
```

확인 기준은 `WidgetSpan`이 들어간 화면을 LTR·RTL·혼합 방향으로 나누는 것이다.

- 아랍어 또는 히브리어 문장 안에 아이콘 2개 이상
- 숫자 ID, 영문 제품명, 괄호가 섞인 문장
- 한 줄을 넘기는 좁은 폭
- Flutter 3.49 기준 golden과 3.50 기준 golden의 차이 검토

golden을 기계적으로 전부 덮어쓰면 안 된다. 변경된 픽셀이 placeholder 위치뿐인지 확인하고, 텍스트 자체의 줄바꿈이나 아이콘 크기까지 달라졌다면 별도 회귀로 분리한다. `WidgetSpan`을 사용하지 않는 LTR 화면의 golden까지 함께 갱신할 이유도 없다.

## 출시 전 판단표

`flutter --version`이 3.50이고 다음 검색 결과가 나온다면 이 글의 대상이다.

```text
rg "Transform\.translate|WidgetSpan|placeholder|TextDirection\.rtl" lib test
```

| 발견 내용 | 판단 |
|---|---|
| WidgetSpan만 있고 수동 보정 없음 | RTL 실기기·golden만 확인 |
| RTL 전용 역순 생성 있음 | 논리 순서로 되돌리고 golden 갱신 |
| placeholder 좌표를 직접 반전 | 반전 로직 제거 후 줄바꿈 검증 |
| 앱이 RTL을 지원하지 않음 | 해당 변경보다 일반 3.50 업그레이드 점검 우선 |

Flutter 공식 변경 문서 기준으로 이 동작은 3.50에 포함됐고, 공개 API 시그니처가 바뀐 변경은 아니다. 따라서 컴파일 오류보다 “업그레이드 후 화면이 조용히 이동하는” 유형에 가깝다. 수정의 핵심은 새 API를 추가하는 것이 아니라 과거 엔진의 잘못된 위치를 맞추려고 넣었던 코드를 걷어내는 데 있다.

참고 문서: [Flutter WidgetSpan RTL 양방향 순서 변경](https://docs.flutter.dev/release/breaking-changes/widgetspan-rtl-bidi-order), [Flutter 3.50 변경 목록](https://docs.flutter.dev/release/breaking-changes), [WidgetSpan API](https://api.flutter.dev/flutter/widgets/WidgetSpan-class.html)
