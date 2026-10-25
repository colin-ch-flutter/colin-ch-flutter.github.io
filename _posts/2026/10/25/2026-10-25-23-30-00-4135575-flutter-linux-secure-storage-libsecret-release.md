---
layout: post
title: "Flutter Linux release libsecret 오류 - flutter_secure_storage 실행 조건 점검"
description: "Flutter Linux release에서 flutter_secure_storage가 시작 직후 실패할 때 libsecret 공유 라이브러리, Secret Service 데몬, Snap GLib 충돌을 배포판별로 확인하는 방법을 정리한다."
date: 2026-10-25
tags: [Flutter, Linux, 배포, flutter_secure_storage, CI/CD]
comments: true
share: true
---

![Flutter Linux release의 libsecret과 키링 서비스 점검](https://images.unsplash.com/photo-1629654297299-c8506221ca97?auto=format&fit=crop&w=1600&q=80)

Flutter Linux 앱이 debug에서는 실행되는데 release에서 `libsecret-1.so`를 찾지 못하거나 저장소 초기화 단계에서 멈춘다면, Dart 저장 코드보다 대상 PC의 런타임을 먼저 확인해야 한다. Ubuntu·Fedora처럼 데스크톱 키링을 관리할 수 있는 환경이면 `libsecret`와 Secret Service를 설치하는 선택이 맞다. 서버나 최소 설치 이미지처럼 키링 자체가 없는 환경이라면 이 해결책을 억지로 적용하지 말고 다른 저장 방식이나 배포 형식을 선택해야 한다.

## 빌드 패키지와 실행 패키지는 다르다

`flutter_secure_storage`의 Linux 구현은 컴파일할 때 헤더와 개발 파일이 필요하고, 실행할 때는 공유 라이브러리와 실제 키링 서비스가 필요하다. 패키지 문서도 Ubuntu/Debian은 `libsecret-1-0 libsecret-1-dev`, Fedora 계열은 `libsecret libsecret-devel`로 나누어 안내한다.

| 단계 | Ubuntu/Debian | Fedora/RHEL 계열 | 빠진 경우 |
|---|---|---|---|
| 빌드 | `libsecret-1-dev` | `libsecret-devel` | CMake·링커 오류 |
| 실행 | `libsecret-1-0` | `libsecret` | `.so` 로딩 오류 |
| 키링 | `gnome-keyring` | GNOME Keyring 또는 KWallet | Secret Service 연결 실패 |

실행 PC만 준비하고 CI 빌드 이미지에 개발 패키지를 빠뜨리거나, 반대로 빌드 머신에만 `libsecret`가 있어서 배포 번들에 포함됐다고 착각하는 경우가 흔하다. Linux release 번들은 모든 시스템 라이브러리를 자동으로 휴대하는 AppImage가 아니다.

## 배포판에서 먼저 확인할 명령

아래 명령은 앱을 실행하기 전에 공유 라이브러리와 D-Bus 키링을 분리해 확인하는 최소 점검이다.

```bash
# 실행 파일이 요구하는 libsecret을 찾는지 확인한다.
ldd build/linux/x64/release/bundle/my_app | rg 'secret|not found'

# Secret Service 이름을 제공하는 프로세스가 있는지 확인한다.
busctl --user list | rg 'org.freedesktop.secrets|gnome.keyring'

# 실제 release 번들에서 플러그인과 자산이 빠지지 않았는지 확인한다.
find build/linux/x64/release/bundle -maxdepth 3 \
  \( -name 'libflutter_linux_gtk.so' -o -name 'flutter_assets' \) -print
```

`ldd`에 `not found`가 나오면 패키지 설치 문제다. 라이브러리는 보이지만 Secret Service 이름이 없으면 데스크톱 세션·D-Bus·키링 잠금 상태를 확인해야 한다. 이 둘을 구분하지 않고 `flutter clean`만 반복해도 운영 PC의 환경은 바뀌지 않는다.

## Snap과 headless CI는 별도 조건이다

Flutter를 Ubuntu Snap으로 설치한 환경에서는 `flutter_secure_storage_linux`가 보고한 GLib 버전 충돌도 확인해야 한다. `g_task_set_static_name`이나 `g_once_init_enter_pointer` 같은 undefined reference가 링크 단계에 나오면 애플리케이션 코드보다 Flutter SDK 설치 경로와 시스템 `libsecret`의 조합이 원인일 수 있다. 이 경우 공식 tar archive 설치로 빌드 도구와 시스템 라이브러리의 기준을 맞추는 편이 안전하다.

키링이 없는 CI에서는 GUI가 없다는 사실보다 Secret Service 데몬이 없다는 점이 문제다. 패키지의 Linux 테스트 문서처럼 테스트용 환경에서 D-Bus 세션과 잠금 해제된 `gnome-keyring-daemon`을 별도로 띄워야 한다. 단, CI에서 비밀값을 저장하는 용도로 그대로 사용하지 말고 테스트 후 키링과 워크스페이스를 폐기해야 한다.

출시 전에는 다음 조건을 실제 배포 대상과 같은 이미지에서 확인한다.

- [ ] release 실행 파일의 `ldd`에 `not found`가 없다.
- [ ] GNOME·KDE 등 대상 데스크톱에 Secret Service 제공자가 있다.
- [ ] Snap 빌드라면 GLib undefined reference가 없는지 확인했다.
- [ ] headless CI라면 D-Bus 세션과 임시 키링을 테스트 단계에서만 만들었다.
- [ ] 키링이 없는 서버·컨테이너에는 secure storage를 필수 초기화로 두지 않았다.

Linux에서 `flutter_secure_storage` 오류는 Flutter 플랫폼 채널 하나를 고치는 문제가 아니다. `libsecret`는 빌드 의존성, 공유 라이브러리는 실행 의존성, Secret Service는 사용자 세션 의존성이다. 공식 패키지 문서 기준은 2026년 10월 25일에 확인했으며, 배포판과 설치 방식이 다르면 패키지 이름과 키링 활성화 절차도 달라진다. [Flutter Linux Snap 배포 문서](https://docs.flutter.dev/deployment/linux)와 [`flutter_secure_storage_linux` 실행 조건](https://pub.dev/documentation/flutter_secure_storage_linux/latest/)을 기준으로 대상 환경을 고정하면 release에서만 터지는 저장소 오류를 빌드 재시도로 오해하지 않게 된다.
