---
layout: post
title: "Flutter Linux Flatpak 파일 선택 오류 - XDG Desktop Portal과 Wayland 조건"
description: "Flutter Linux 앱을 Flatpak으로 배포한 뒤 파일 선택창이 안 열리거나 부모 창에 붙지 않을 때 file_picker, XDG Desktop Portal, X11·Wayland 조건을 점검하는 방법을 정리한다."
date: 2026-10-26
tags: [Flutter, Linux, Flatpak, file_picker, 배포]
comments: true
share: true
---

![Flutter Linux Flatpak 앱에서 XDG Desktop Portal 파일 선택창을 점검하는 화면](https://images.unsplash.com/photo-1629654297299-c8506221ca97?auto=format&fit=crop&w=1600&q=80)

그림에서 확인할 부분은 앱 자체가 파일 시스템을 직접 여는지, 데스크톱 Portal을 통해 사용자가 고른 파일만 받는지다.

Flutter Linux 앱을 Flatpak으로 배포했는데 `file_picker` 호출 뒤 창이 안 뜨거나 선택한 파일을 읽지 못한다면, `--filesystem=home` 권한부터 추가하는 선택은 피하는 편이 좋다. `file_picker_linux`는 XDG Desktop Portal의 FileChooser를 사용하고, Portal은 샌드박스 밖 파일을 사용자가 고른 범위에서 앱에 연결한다. 다만 Portal 서비스가 없거나, 앱이 Wayland에서 창 핸들을 넘기지 못하는 경우는 별도로 나눠야 한다. 일반 Linux 패키지에는 직접 파일 접근이 더 단순할 수 있어 Flatpak이 아닌 배포라면 이 글의 권한 해결책이 그대로 적용되지 않는다.

## 증상별로 원인을 나눈다

| 증상 | 우선 의심할 것 | 확인 기준 |
|---|---|---|
| 선택창 자체가 안 뜸 | `xdg-desktop-portal` 또는 백엔드 | `flatpak run` 터미널 로그 |
| 창은 뜨지만 부모 창에 고정되지 않음 | Wayland 창 핸들 부재 | X11과 Wayland를 나눠 재현 |
| 선택 후 읽기 실패 | 직접 경로 접근 또는 권한 override | `flatpak info --show-permissions` |
| 특정 배포판에서만 실패 | 포털 백엔드·데스크톱 환경 조합 | GNOME, KDE, Sway를 구분 |

`file_picker` 최신 API는 파일을 고른 뒤 `PlatformFile`을 반환한다. 파일 크기와 바이트를 바로 속성으로 읽는 이전 예제를 그대로 복사하면 버전 변경에서 또 막힐 수 있다.

```dart
import 'package:file_picker/file_picker.dart';

Future<PlatformFile?> pickPdf() async {
  final file = await FilePicker.pickFile(
    type: FileType.custom,
    allowedExtensions: ['pdf'],
  );

  if (file == null) return null; // 사용자가 취소한 경우
  final bytes = await file.readAsBytes();
  print('${file.name}: ${bytes.length} bytes');
  return file;
}
```

이 코드는 Linux에서 Portal을 호출하도록 두고, 앱 코드에서는 선택 취소와 실제 읽기 실패를 분리한다. 파일 필터는 사용자를 돕는 힌트일 뿐이므로 확장자 검증을 보안 경계로 사용하면 안 된다.

## Flatpak 권한은 넓히기 전에 확인한다

```bash
flatpak run com.example.MyApp
flatpak info --show-permissions com.example.MyApp
flatpak override --user --show com.example.MyApp
```

첫 명령으로 파일 선택 동작 직전의 오류를 확인하고, 나머지 명령으로 기존에 Flatseal이나 `flatpak override`가 만든 변경을 확인한다. FileChooser Portal을 정상적으로 쓰는 앱이라면 사용자가 선택한 파일에 대한 접근이 Portal을 통해 제공되므로 홈 디렉터리 전체 권한이 필수는 아니다. `filesystem=home`을 무조건 추가하면 문제는 잠깐 사라져도 샌드박스의 의미와 배포 검토 범위가 커진다.

## Wayland에서는 모달 동작을 기대하지 않는다

`file_picker_linux` 문서 기준으로 X11은 창 XID를 `parentWindow`에 넘길 수 있지만, 일반적인 Wayland 세션에서는 Flutter가 `xdg_foreign` 핸들을 노출하지 않아 `lockParentWindow`가 눈에 띄게 동작하지 않을 수 있다. 파일 선택 자체가 실패한 것과 부모 창에 붙지 않은 것은 다른 증상이다.

| 배포 조건 | 현실적인 선택 |
|---|---|
| 파일 선택만 필요 | 기본 Portal 호출, 부모 창 옵션 생략 |
| X11에서 모달 고정 필요 | 런타임 XID를 얻어 `parentWindow` 전달 |
| Wayland 네이티브 세션 | 모달 고정을 필수 UX로 만들지 않음 |
| 포털이 없는 테스트 이미지 | `xdg-desktop-portal`과 데스크톱별 백엔드 설치 |

확인 순서는 `flatpak run` 로그 → Portal 설치·백엔드 상태 → 권한 override → X11/Wayland 차이 순서가 짧다. 실제 배포 전에는 최소한 파일 선택, 취소, 선택 파일 읽기, 저장 대화상자를 각각 확인한다.

## 짧게 정리

- Flatpak 파일 선택 오류는 홈 디렉터리 권한보다 FileChooser Portal 경로를 우선 확인한다.
- 선택창 불가와 부모 창 미고정은 서로 다른 문제다.
- Wayland에서 `lockParentWindow`가 무시된 것처럼 보여도 Portal 호출 자체의 실패는 아닐 수 있다.
- 확인 기준은 2026년 10월 26일이며, [file_picker_linux 문서](https://pub.dev/packages/file_picker_linux), [XDG Desktop Portal FileChooser 명세](https://flatpak.github.io/xdg-desktop-portal/docs/doc-org.freedesktop.portal.FileChooser.html), [Flathub 권한 점검 안내](https://docs.flathub.org/docs/for-users/troubleshooting)를 바탕으로 했다.
