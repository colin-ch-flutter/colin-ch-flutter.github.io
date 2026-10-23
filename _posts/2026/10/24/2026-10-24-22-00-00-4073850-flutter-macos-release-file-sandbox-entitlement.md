---
layout: post
title: "Flutter macOS release 파일 오류 - file_selector가 Operation not permitted일 때"
description: "Flutter macOS 앱이 debug에서는 파일을 읽지만 release에서 Operation not permitted를 낼 때 App Sandbox entitlement, 파일 선택 권한, 재실행 검증 순서를 정리한다."
date: 2026-10-24
tags: [Flutter, macOS, 배포·운영, CI/CD, file_selector]
comments: true
share: true
---

![Flutter macOS release 파일 접근과 App Sandbox entitlement 점검](https://images.unsplash.com/photo-1517336714739-489689fd1ca8?auto=format&fit=crop&w=1600&q=80)

이 그림에서 볼 부분은 파일 경로가 아니라, macOS 앱이 그 파일에 접근할 수 있다고 서명으로 선언했는지다.

Flutter macOS 앱에서 `file_selector`로 사용자가 고른 파일을 debug에서는 읽는데 `flutter build macos --release` 결과에서 `Operation not permitted`가 난다면 `File` API를 바꿀 문제가 아니다. App Sandbox가 켜진 release entitlement에 `com.apple.security.files.user-selected.read-only` 또는 `read-write`가 빠졌는지 우선 확인할 대상이다. 파일을 고르는 기능이 없는 앱이나 앱 내부 컨테이너만 쓰는 앱에는 이 설정이 필요하지 않다.

## Debug는 되고 Release는 실패하는 이유

Flutter macOS 템플릿은 Debug/Profile과 Release entitlement 파일을 나눠 관리한다. 개발 실행에는 Flutter 도구와 디버거에 필요한 예외가 들어 있지만, 배포 앱은 App Sandbox 규칙을 그대로 적용받는다. 그래서 Xcode에서 capability를 한 번 켰다는 사실만으로 두 빌드 설정이 모두 고쳐졌다고 보면 안 된다.

| 작업 | 필요한 entitlement | 선택 기준 |
| --- | --- | --- |
| 사용자가 고른 파일 읽기 | `com.apple.security.files.user-selected.read-only` | 원본을 수정하지 않음 |
| 사용자가 고른 파일 저장·수정 | `com.apple.security.files.user-selected.read-write` | 파일 내용을 변경함 |
| 앱 컨테이너 내부 저장 | 별도 user-selected 권한 불필요 | `getApplicationSupportDirectory()` 등 사용 |

여기서 `read-write`를 무조건 넣으면 권한 범위가 넓어진다. CSV나 JSON을 가져오기만 하는 앱이라면 read-only가 맞다.

## 두 entitlement 파일에 같은 값을 넣는다

파일 선택 후 장기적으로 다시 열어야 하는 앱이라면 아래처럼 두 파일을 모두 확인한다.

`macos/Runner/DebugProfile.entitlements`와 `macos/Runner/Release.entitlements`에 필요한 키를 추가한다.

```xml
<key>com.apple.security.app-sandbox</key>
<true/>
<key>com.apple.security.files.user-selected.read-only</key>
<true/>
```

실제 파일을 읽는 코드는 선택 결과가 취소된 경우와 권한 오류를 분리한다.

```dart
final XFile? picked = await openFile(
  acceptedTypeGroups: [
    const XTypeGroup(label: 'JSON', extensions: ['json']),
  ],
);

if (picked == null) return; // 사용자가 선택창을 취소함

try {
  final content = await File(picked.path).readAsString();
  // 파싱과 검증
} on FileSystemException catch (error) {
  // entitlement 누락, 삭제된 파일, 접근 거부를 사용자에게 구분해 알림
  debugPrint('macOS file read failed: $error');
}
```

`file_selector`가 경로를 반환했다고 해서 애플리케이션이 그 경로를 영구적으로 읽을 수 있다는 뜻은 아니다. 앱을 종료한 뒤 다시 열어 같은 파일을 읽어야 한다면 security-scoped bookmark 같은 별도 설계가 필요하다. 단순히 경로 문자열만 저장하는 방식은 샌드박스 앱의 재실행 조건을 보장하지 않는다.

## release 앱에서 확인할 체크리스트

- [ ] `DebugProfile.entitlements`와 `Release.entitlements`의 키가 의도대로 일치한다.
- [ ] 읽기만 하는 기능은 `read-only`, 수정 기능은 `read-write`로 최소 권한을 선택했다.
- [ ] `flutter clean && flutter build macos --release`로 이전 산출물을 제거하고 다시 빌드했다.
- [ ] release `.app`에서 파일 선택, 읽기, 앱 종료 후 재실행을 각각 확인했다.
- [ ] `codesign -d --entitlements :- build/macos/Build/Products/Release/*.app`로 실제 서명된 entitlement를 확인했다.

Mac App Store 배포 앱이라면 App Sandbox를 끄는 우회보다 필요한 권한만 선언하는 쪽이 맞다. 반대로 로컬에서만 쓰는 도구라도 사용자가 고른 파일을 다른 실행 세션에서 계속 사용해야 한다면 권한과 bookmark 수명을 따로 설계해야 한다.

짧게 정리하면, Flutter macOS 파일 오류는 `file_selector` 호출보다 최종 `.app`의 서명을 봐야 한다. Debug와 Release 두 entitlement를 맞추고, 읽기·쓰기 권한을 최소화하고, 앱 재실행까지 검증하면 `Operation not permitted`를 경로 문제와 구분할 수 있다.

참고 문서: [Flutter macOS App Sandbox 안내](https://docs.flutter.dev/platform-integration/macos/building), [Apple App Sandbox 설정](https://developer.apple.com/documentation/xcode/configuring-the-macos-app-sandbox), [Apple macOS 파일 접근](https://developer.apple.com/documentation/security/accessing-files-from-the-macos-app-sandbox)
