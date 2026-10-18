---
layout: post
title: "Flutter iOS 26 키보드 오류 - BottomSheet 배경이 검게 비칠 때"
description: "Flutter iOS 26에서 반투명 키보드 뒤로 BottomSheet 배경이 검게 보이는 원인과 viewInsets 적용 위치, 임시 출시 우회 조건을 정리한다."
date: 2026-10-18
tags: [Flutter, iOS, Cupertino, UI]
comments: true
share: true
---

![Flutter iOS 26 BottomSheet 키보드 배경 오류 점검](/images/2026-10-18-flutter-ios26-bottomsheet-keyboard.png)

이 문제는 iOS 26에서 텍스트 입력이 있는 `showModalBottomSheet`를 출시하는 앱이 우선 확인해야 한다. `viewInsets.bottom`을 배경 위젯에 직접 적용했다면 키보드 뒤가 검게 비칠 수 있으므로, 배경은 전체 시트에 칠하고 inset은 그 안의 입력 콘텐츠에만 적용하는 방식이 유리하다. iOS 18 이하만 지원하거나 BottomSheet를 쓰지 않는 앱에는 해당되지 않는다.

## 왜 검은 영역이 생기나

iOS 26 키보드는 반투명한 Liquid Glass 형태로 바뀌었다. 키보드가 불투명하던 때에는 `viewInsets.bottom` 아래를 그리지 않아도 문제가 드러나지 않았지만, 이제는 그 영역의 실제 픽셀이 보인다. Flutter 공식 이슈에서도 iOS 18에서는 정상이고, Flutter 3.38·3.39에서 iOS 26 BottomSheet 입력 시 검은 배경이 보이는 조건이 재현됐다.

핵심은 여백과 페인트 영역을 같은 위젯에 두지 않는 것이다.

| 구성 | 키보드 뒤 배경 | 입력 콘텐츠 위치 |
|---|---|---|
| 투명 시트 + 바깥쪽 `viewInsets` | 검은색·기본 배경이 노출될 수 있음 | 올라가지만 배경 경계가 끊김 |
| 불투명 시트 + 안쪽 `viewInsets` | 시트 색상이 계속 그려짐 | 키보드 위로만 이동 |
| `UIDesignRequiresCompatibility` | 구형 렌더링으로 임시 회피 | 장기 해법 아님 |

## viewInsets를 안쪽에 둔다

배경을 먼저 책임지는 `Material` 또는 `ColoredBox`를 만들고, 그 자식에만 키보드 여백을 준다. 아래 코드는 입력창과 버튼은 키보드를 피하면서 시트의 표면색은 키보드 뒤까지 유지한다.

```dart
showModalBottomSheet<void>(
  context: context,
  isScrollControlled: true,
  backgroundColor: const Color(0xfff7f7f8),
  builder: (sheetContext) {
    final keyboardInset = MediaQuery.viewInsetsOf(sheetContext).bottom;

    return Material(
      color: const Color(0xfff7f7f8),
      borderRadius: const BorderRadius.vertical(
        top: Radius.circular(24),
      ),
      clipBehavior: Clip.antiAlias,
      child: Padding(
        padding: EdgeInsets.fromLTRB(20, 20, 20, keyboardInset + 20),
        child: const Column(
          mainAxisSize: MainAxisSize.min,
          children: [
            TextField(),
            SizedBox(height: 16),
            SizedBox(
              width: double.infinity,
              child: ElevatedButton(
                onPressed: null,
                child: Text('저장'),
              ),
            ),
          ],
        ),
      ),
    );
  },
);
```

`backgroundColor: Colors.transparent`를 꼭 써야 한다면, `builder` 안에서 `ColoredBox`를 `Padding`보다 바깥에 둔다. 반대로 `Padding`으로 시트 전체를 밀어 올린 뒤 그 바깥에 색을 칠하면 키보드가 비추는 영역과 시트 표면이 분리된다.

## 출시 전 확인할 조건

- iOS 26 실기기에서 텍스트 필드에 포커스하고 키보드를 연다.
- 키보드가 올라오는 동안 시트를 드래그해 둥근 상단 모서리 주변을 확인한다.
- `showModalBottomSheet`, `Scaffold.bottomSheet`, 커스텀 오버레이를 각각 나눠 확인한다.
- `viewInsets.bottom`을 배경 컨테이너가 아니라 입력 콘텐츠에 적용했는지 확인한다.
- iOS 18에서도 같은 화면을 확인해 조건부 회귀가 없는지 본다.

Flutter 공식 조사 기준은 “배경 그리기는 키보드 뒤까지 확장하고, 상호작용 레이아웃만 inset을 피한다”다. 프레임워크 소유의 persistent BottomSheet는 Flutter 수정이 필요한 영역일 수 있어 앱 코드만으로 완전히 고쳐지지 않는다. 앱이 직접 만든 modal이라면 `backgroundColor`를 지정하고, 위와 같이 여백의 위치를 옮기는 것이 우선이다.

임시로 `ios/Runner/Info.plist`에 `UIDesignRequiresCompatibility`를 `true`로 넣어 Liquid Glass를 끌 수 있지만, Flutter 이슈도 이를 임시 우회책으로만 설명한다. 새 iOS 디자인을 포기하는 설정이므로 장기 배포 플래그로 남기지 않는 편이 안전하다.

공식 기준은 2026년 10월 18일 확인했다. 참고 문서는 [Flutter iOS 26 키보드 조사 이슈](https://github.com/flutter/flutter/issues/179482), [Flutter의 iOS 26 키보드 배경 조사](https://flutter.dev/go/ios-26-keyboard-bleed), [Apple의 UIDesignRequiresCompatibility 문서](https://developer.apple.com/documentation/bundleresources/information-property-list/uidesignrequirescompatibility)다.
