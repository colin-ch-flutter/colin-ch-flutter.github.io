---
layout: post
title: "Flutter 적응형 내비게이션 - NavigationBar와 NavigationRail로 스마트홈 화면 만들기"
description: "Flutter LayoutBuilder로 스마트홈 앱의 좁은 화면은 NavigationBar, 넓은 화면은 NavigationRail로 자동 전환하고 탭 상태를 유지하는 구현 방법을 정리했다."
date: 2026-09-03
tags: [Flutter, Dart, IoT, 스마트홈, UX, Material 3]
comments: true
share: true
---

![Flutter NavigationBar와 NavigationRail을 적용한 스마트홈 대시보드](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

스마트폰에서는 `NavigationBar`, 태블릿과 데스크톱에서는 `NavigationRail`을 보여주는 것이 Flutter 적응형 내비게이션의 기본 해법이다. 화면 폭을 `LayoutBuilder`로 판단하면 폴더블과 가로 모드에서도 같은 위젯을 재사용할 수 있다.

## 문제는 화면보다 상태였다

탭 인덱스는 상위 상태 하나에서 관리한다. 내비게이션 UI만 바뀌고 페이지 본문은 그대로 남는 구조다.

| 화면 폭 | 내비게이션 | 사용성 판단 |
|---|---|---|
| 600 미만 | `NavigationBar` | 엄지 조작에 유리하다 |
| 600 이상 | `NavigationRail` | 대시보드와 목록을 함께 보기 좋다 |

## LayoutBuilder로 분기하기

탭 인덱스를 상위에서 관리하고, 폭에 따라 내비게이션 위젯만 바꾸는 구조다.

```dart
class _HomeShellState extends State<HomeShell> {
  int selectedIndex = 0;

  static const labels = ['홈', '기기', '자동화', '설정'];
  static const icons = [Icons.home_outlined, Icons.devices_outlined,
    Icons.schedule_outlined, Icons.settings_outlined];
  static const pages = [HomePage(), DevicePage(), AutomationPage(), SettingsPage()];

  void select(int index) => setState(() => selectedIndex = index);

  @override
  Widget build(BuildContext context) {
    return LayoutBuilder(
      builder: (context, constraints) {
        final isWide = constraints.maxWidth >= 600;
        final body = IndexedStack(index: selectedIndex, children: pages);

        if (!isWide) {
          return Scaffold(
            body: body,
            bottomNavigationBar: NavigationBar(
              selectedIndex: selectedIndex,
              onDestinationSelected: select,
              destinations: [for (var i = 0; i < labels.length; i++)
                NavigationDestination(icon: Icon(icons[i]), label: labels[i])],
            ),
          );
        }

        return Scaffold(
          body: Row(
            children: [
              NavigationRail(
                selectedIndex: selectedIndex,
                onDestinationSelected: select,
                labelType: NavigationRailLabelType.all,
                destinations: [for (var i = 0; i < labels.length; i++)
                  NavigationRailDestination(icon: Icon(icons[i]), label: Text(labels[i]))],
              ),
              const VerticalDivider(width: 1),
              Expanded(child: body),
            ],
          ),
        );
      },
    );
  }
}
```

`IndexedStack`은 탭을 바꿔도 각 페이지의 스크롤 위치와 입력 상태를 유지한다. MQTT 스트림을 페이지 내부에서 구독한다면 매번 페이지를 새로 만드는 방식보다 안전하다.

## 구현 중에 걸린 함정

`NavigationRailDestination`과 `NavigationDestination`은 다른 타입이라 각각 만들어야 한다. `Row` 안의 본문에는 `Expanded`도 필요하다.

600은 고정 법칙이 아니다. 카드 최소 너비를 보고 조정하며, 599↔600 경계와 가로 모드에서 탭·MQTT 구독을 확인한다.

## 짧게 정리하면

- 좁은 화면은 `NavigationBar`, 넓은 화면은 `NavigationRail`로 나눈다.
- 폭 판단은 `LayoutBuilder`, 탭 상태는 상위 `State` 하나에 둔다.
- `IndexedStack`으로 IoT 카드 상태를 보존하고 폭 경계를 테스트한다.
