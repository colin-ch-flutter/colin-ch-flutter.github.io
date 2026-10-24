---
layout: post
title: "Flutter Windows MSIX 설치 오류 - 0x800B0109 인증서 신뢰 조건"
description: "Flutter Windows 앱을 MSIX로 배포할 때 빌드는 성공했지만 0x800B0109 또는 게시자 인증서 오류로 설치되지 않는 원인을 나누고, 자체 배포와 Microsoft Store의 인증서 조건을 점검한다."
date: 2026-10-25
tags: [Flutter, Windows, MSIX, 배포, CI/CD]
comments: true
share: true
---

![Flutter Windows MSIX 인증서와 앱 설치 오류](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?auto=format&fit=crop&w=1600&q=80)

Flutter Windows 앱을 사내 PC나 웹사이트에서 직접 배포한다면 MSIX 파일보다 **서명 인증서를 설치 대상 PC가 신뢰하는지**가 먼저다. `flutter build windows`와 MSIX 패키징이 성공했는데 다른 PC에서 `0x800B0109`, “publisher could not be verified”가 나온다면 재빌드보다 인증서 체인을 확인해야 한다. Microsoft Store로 배포하는 앱에는 이 자체 서명 인증서를 사용자에게 설치시키는 절차가 필요 없다.

## 빌드 성공과 설치 성공은 다른 단계다

Flutter 공식 문서는 Windows 배포 경로를 Store, 자체 서명 MSIX, ZIP으로 나눈다. `msix` 패키지의 개발용 기본 인증서는 테스트 인증서라서 개발 PC에서는 설치돼도, 인증서를 모르는 사용자 PC에서는 차단될 수 있다.

| 배포 방식 | 패키지 서명 | 사용자 PC에서 할 일 | 운영 판단 |
|---|---|---|---|
| Microsoft Store | Store가 처리 | 별도 인증서 설치 없음 | 일반 사용자 배포에 유리 |
| 사내·테스트 MSIX | 자체 서명 또는 CA 인증서 | 자체 인증서를 신뢰 저장소에 추가 | 통제된 장비에 적합 |
| 웹사이트 공개 MSIX | Windows가 신뢰하는 CA 인증서 | 보통 추가 작업 없음 | 인증서 비용과 갱신 관리 필요 |
| ZIP 배포 | MSIX 서명 없음 | 압축 해제 후 실행 | 자동 업데이트·설치 경험이 약함 |

여기서 `0x800B0109`는 대개 패키지 파일이 깨졌다는 뜻이 아니라 인증서 체인을 신뢰할 수 없다는 뜻이다. 같은 파일이 내 PC에서만 설치되는 이유도 개발 PC에 테스트 인증서가 이미 등록돼 있기 때문이다.

## 자체 배포라면 인증서의 세 가지 값을 맞춘다

MSIX의 `Publisher`와 인증서의 주체, 패키지 정체성이 어긋나면 신뢰 인증서를 추가해도 설치가 실패할 수 있다. `pubspec.yaml`의 `msix_config`를 작성할 때 표시 이름만 바꾸고 인증서 설정을 잊는 경우가 흔하다.

```yaml
# 사내 테스트용 예시다. 비밀번호를 저장소에 커밋하지 않는다.
msix_config:
  display_name: Example Desk
  publisher_display_name: Example Team
  identity_name: com.example.desk
  certificate_path: C:\\secure\\example-desk.pfx
  certificate_password: ${MSIX_CERTIFICATE_PASSWORD}
```

CI에서는 비밀번호를 YAML에 직접 넣지 말고 시크릿으로 주입한다. 공개 배포라면 자체 서명 인증서를 사용자에게 설치하라고 안내하기보다 Windows가 신뢰하는 인증기관의 코드 서명 인증서를 검토한다. Microsoft Store를 목표로 한다면 Store 제출용 unsigned 패키지를 만들고, Store의 서명 흐름을 따른다.

## 설치 PC에서 확인할 순서

인증서를 무턱대고 현재 사용자 저장소에만 넣으면 App Installer가 계속 실패할 수 있다. Microsoft의 문제 해결 문서가 안내하는 자체 서명 인증서의 핵심 위치는 **Local Computer → Trusted People**다.

```powershell
# MSIX에서 서명 인증서를 추출해 확인하는 예시다.
$cert = (Get-AuthenticodeSignature -FilePath .\ExampleDesk.msix).SignerCertificate
$cert | Format-List Subject, Issuer, Thumbprint, NotAfter
Export-Certificate -Cert $cert -FilePath .\ExampleDesk.cer
```

설치 대상 PC에서 다음을 체크한다.

- [ ] MSIX의 `SignerCertificate`가 만료되지 않았고 발급자와 주체를 확인했다.
- [ ] 자체 서명 인증서라면 `.cer`를 `Local Computer\Trusted People`에 추가했다.
- [ ] 인증서의 `NotAfter`가 현재 날짜보다 이후다.
- [ ] 새 인증서로 서명한 패키지라면 이전 버전과 동일한 패키지 정체성을 유지했다.
- [ ] Store 배포가 아닌데 테스트 인증서를 일반 사용자에게 배포하고 있지 않다.
- [ ] 설치 실패 시 이벤트 뷰어의 `Microsoft > Windows > AppxDeployment-Server > Operational` 로그를 확인했다.

자체 서명 인증서 추가는 관리자 권한과 보안 검토가 필요한 운영 변경이다. 신뢰 저장소에 인증서를 넣는 방법만 자동화하고, 인증서의 지문이나 만료일을 검증하지 않으면 악성 패키지도 같은 경로로 신뢰할 수 있다.

## 짧은 요점 정리

Flutter Windows MSIX의 “내 PC에서는 설치되고 다른 PC에서는 안 되는” 문제는 빌드보다 서명 신뢰 조건에서 시작하는 경우가 많다. Store 배포인지, 통제된 사내 배포인지, 공개 웹 배포인지부터 정하고 인증서 정책을 선택해야 한다. 자체 서명 테스트 인증서는 개발·검증 장비에만 사용하고, 운영 배포에서는 인증서 만료·Publisher 일치·갱신 절차까지 출시 체크리스트에 넣는다.

참고 문서: [Flutter Windows 앱 빌드와 배포](https://docs.flutter.dev/platform-integration/windows/building), [Flutter Windows 배포](https://docs.flutter.dev/deployment/windows), [Microsoft MSIX 문제 해결 가이드](https://learn.microsoft.com/en-us/windows/msix/msix-troubleshooting-guide)
