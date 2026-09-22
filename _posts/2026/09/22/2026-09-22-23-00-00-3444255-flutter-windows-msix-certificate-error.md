---
layout: post
title: "Flutter Windows MSIX 설치 오류 - 0x800B0109 인증서와 배포 방식 선택"
description: "Flutter Windows 앱을 MSIX로 배포할 때 0x800B0109 인증서 오류가 나는 이유와 Microsoft Store·웹 자체 배포별 인증서 조건, 검증 절차를 정리한다."
date: 2026-09-22
tags: [Flutter, Windows, 배포, CI/CD]
comments: true
share: true
---

![Flutter Windows MSIX 인증서와 배포 경로](/images/2026-09-22-flutter-windows-msix-certificate.png)

이 글은 Flutter Windows 앱을 사내 PC나 고객 PC에 직접 설치할 때 MSIX가 `0x800B0109`로 거부되는 경우를 위한 것이다. Microsoft Store에 올릴 앱이면 Store 서명을 선택하고, 웹에서 내려받게 할 앱이면 신뢰할 수 있는 CA 인증서를 준비해야 한다. 개발 PC에서만 시험한다면 자체 서명 인증서를 신뢰 저장소에 넣는 것으로 충분하며, 이때 유료 인증서를 바로 살 필요는 없다.

## 오류가 생기는 이유

MSIX는 실행 파일 묶음이 아니라 서명된 패키지다. Windows가 서명자의 인증서 체인을 신뢰하지 못하면 다음 설치 오류가 난다.

`0x800B0109: A certificate chain processed, but terminated in a root certificate which is not trusted by the trust provider.`

Flutter 공식 문서 기준으로 Store 배포는 인증서를 직접 만들 필요가 없고, 웹 자체 배포는 Windows가 신뢰하는 인증기관의 인증서가 필요하다. 자체 서명 인증서는 로컬 테스트용으로만 쓴다.

| 배포 방식 | 서명 조건 | 적합한 상황 |
| --- | --- | --- |
| Microsoft Store | 제출 후 Store가 관리 | 공개 배포, 자동 업데이트 |
| 웹·사내 서버 | 신뢰된 CA의 `.pfx` | 고객·사내 배포, Store 밖 유통 |
| 로컬 테스트 | 자체 서명 `.pfx` + PC 신뢰 등록 | 개발자 PC와 테스트 장비 |

## Flutter에서 MSIX 만들기

`msix` 패키지는 Flutter의 Release 산출물을 패키징하므로 먼저 Windows 빌드가 성공해야 한다.

```yaml
dev_dependencies:
  msix: ^3.16.7

msix_config:
  display_name: Sample Flutter App
  publisher_display_name: Sample Team
  identity_name: SampleTeam.SampleFlutterApp
  msix_version: 1.0.0.0
  certificate_path: C:\\certs\\sample.pfx
  certificate_password: your-password
```

```powershell
flutter build windows --release
dart run msix:create
```

`msix_version`의 네 번째 숫자는 0이어야 한다. Microsoft Store는 revision 값이 0이 아닌 버전을 허용하지 않으므로 `1.0.0.1` 같은 값은 제출 전에 고쳐야 한다.

## 0x800B0109를 확인하는 순서

1. 패키지 서명자와 인증서의 `Subject`, 유효기간을 확인한다.
2. 자체 서명 인증서라면 `.cer` 공개 인증서를 내보낸다.
3. 테스트 PC의 `로컬 컴퓨터 → 신뢰할 수 있는 사용자` 저장소에 인증서를 가져온다. 현재 사용자 저장소만 등록하면 App Installer가 계속 거부할 수 있다.
4. 기존 앱을 제거한 뒤 새 MSIX를 설치한다. 이전 패키지와 새 패키지의 게시자 정보가 다르면 업그레이드가 아니라 새 설치로 판단될 수 있다.

설치 후에는 일반 사용자 계정에서 파일 접근과 업데이트를 확인한다. Store로 전환할 때는 테스트용 자체 서명 인증서를 재사용하지 말고 Partner Center의 제품 ID·Publisher 정보를 MSIX 설정과 맞춘다.

## 출시 전 체크리스트

- [ ] `flutter build windows --release` 결과를 MSIX에 포함했다.
- [ ] Store인지 웹 배포인지 서명 방식을 먼저 결정했다.
- [ ] 웹 배포라면 CA 인증서와 만료일 갱신 담당자를 정했다.
- [ ] 자체 서명 테스트는 각 테스트 PC의 로컬 컴퓨터 신뢰 저장소에 등록했다.
- [ ] Windows App Certification Kit으로 패키지를 검사했다.

핵심은 MSIX 생성 성공과 설치 신뢰가 별개라는 점이다. Store 밖에서 배포한다면 인증서 구매보다 신뢰 체인과 갱신 비용을 먼저 계산해야 한다.

공식 기준은 [Flutter Windows 배포 문서](https://docs.flutter.dev/platform-integration/windows/building), [Flutter Windows Store 출시 문서](https://docs.flutter.dev/deployment/windows), [Microsoft MSIX 문제 해결 가이드](https://learn.microsoft.com/en-us/windows/msix/msix-troubleshooting-guide)에서 확인할 수 있다.
