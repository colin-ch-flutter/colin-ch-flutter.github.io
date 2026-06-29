# Colin Flutter 블로그 — 전용 하네스

> 공통 규칙(파일명·frontmatter·SEO·인간 글쓰기·워크플로)은 상위 폴더 `../CLAUDE.md` 참조

---

## 블로그 정보

| 항목 | 값 |
|------|----|
| 사이트 URL | https://colin-ch-flutter.github.io |
| 저자 | Colin Flutter IoT |
| 언어 | 한국어 (주), 영어 (레거시) |
| 주제 | Flutter 앱 개발 실전 — IoT, 스마트홈, BLE, MQTT |
| Jekyll 설정 | `_config.yml` 참조 |

---

## 이 블로그의 SEO 키워드 전략

검색 유입을 노리는 핵심 키워드 그룹. 포스트 title·description·첫 문단에 반드시 포함.

| 그룹 | 주요 키워드 | 의도 |
|------|------------|------|
| Flutter IoT | `Flutter IoT`, `Flutter 스마트홈`, `Flutter BLE` | 정보 탐색 |
| 패키지 실전 | `flutter_blue_plus 사용법`, `mqtt5_client 예제`, `fl_chart 커스텀` | 방법 탐색 |
| 문제 해결 | `Flutter BLE 재연결`, `MQTT 연결 끊김`, `SigV4 403 오류` | 문제 해결 |
| 아키텍처 | `Flutter 클린 아키텍처`, `GetX Repository 패턴` | 학습 탐색 |

**description 작성 예시:**
```
# 좋은 예
"flutter_blue_plus로 BLE 기기를 연결할 때 iOS에서 재연결이 안 되는 문제와
 CBErrorPeripheralDisconnected 해결 방법을 실제 코드로 정리했다."

# 나쁜 예 (공백 또는 너무 일반적)
" "
"Flutter BLE 연결에 대해 알아봅니다."
```

---

## 태그 목록

새 포스트 작성 시 이 목록에서 선택. 첫 번째 태그는 항상 `Flutter`.

```
# 핵심 기술
Flutter, Dart, GetX, CleanArchitecture, Repository패턴

# 통신·프로토콜
BLE, MQTT, AWS IoT, SigV4, BluFi, WiFi프로비저닝

# 패키지 (패키지명 그대로 사용)
flutter_blue_plus, mqtt5_client, fl_chart, Lottie
permission_handler, flutter_secure_storage, easy_localization
realm, mobile_scanner, wakelock_plus

# 기능·도메인
IoT, 스마트홈, 보일러, SCADA, FOTA
인증, JWT, SecureStorage, 회원가입, 이메일인증
오프라인캐시, 다크모드, 다국어, 애니메이션

# 플랫폼·운영
Android, iOS, Firebase, Crashlytics, RemoteConfig, FCM
CI/CD, dart-define, 멀티환경, 앱스토어, GitHub Actions

# 기타
UX, 회고
```

---

## 이 블로그의 인간 글쓰기 톤 기준

기존 포스트에서 뽑은 실제 문체 패턴. 새 포스트도 이 톤을 유지한다.

**이 블로그의 실제 문체 예시:**
```
"처음엔 connect()만 호출하면 되는 줄 알았는데 아니었다."
"단연코 BluFi 프로토콜 구현이었다. Flutter(Dart)로 구현된 레퍼런스가 없어서..."
"GetX 쓰기 편한 건 맞는데, 프로젝트가 커지면 Controller가 비대해지는 경향이 있다."
"솔직하게 정리해보면:"
"해보니 됐다."
"디버깅 환경도 열악해서 Serial 로그와 Wireshark로 패킷을 직접 분석했다."
```

**이 블로그에서 실제로 겪은 구체적 상황 활용:**
- BluFi: Dart 레퍼런스 없어서 Android SDK 소스 역분석
- AWS SigV4: 서명 틀리면 `403 Forbidden`만 나오고 원인 불명
- BLE: iOS 시뮬레이터에서 BLE 불가, 실기기 필수
- GetX: Controller 비대화 문제, Repository 패턴으로 해결

---

## 시리즈 현황

### 시리즈 1 — Flutter IoT 스마트홈 앱 (2026.04 ~ 2026.06, **완결 50편**)

| 범주 | 주요 주제 | 편수 |
|------|-----------|------|
| 아키텍처 | 클린 아키텍처, GetX, Repository 패턴 | 4 |
| BLE | flutter_blue_plus, BluFi, WiFi 프로비저닝 | 6 |
| MQTT | AWS IoT Core, SigV4, 토픽 설계, mqtt5_client | 6 |
| 로컬 DB | Realm DB, 오프라인 캐시 | 3 |
| 인증·보안 | JWT 자동갱신, SecureStorage, 회원가입 6단계 | 5 |
| UI/UX | fl_chart, 다크모드, Lottie, 스케줄, 퀵모드 | 8 |
| 배포·운영 | Firebase 셋업, 멀티환경, 앱스토어, FOTA | 7 |
| 기기연동 | SCADA, 알림, wakelock, 데모모드 | 6 |
| 공간·멤버 | Space 관리, 멤버 초대·권한 | 4 |
| 회고 | 50일 회고 | 1 |

마지막 포스트: `_posts/2026/06/18/2026-06-18-11-50-50-617250-50-days-flutter-iot-app-retrospective-smarthome.md`  
다음 일련번호: `629595`

### 시리즈 2 — Flutter Unit Test 실전 (2026.06 ~ , **진행 중**)

| 범주 | 주요 주제 | 편수 |
|------|-----------|------|
| 기초 패턴 | Fake Repository, Controller Unit Test | 2 |
| BLE 격리 | flutter_blue_plus Wrapper 패턴, FakeBleService | 1 |
| MQTT 격리 | mqtt5_client Fake 서비스로 연결 로직 테스트 | 1 |
| CI 자동화 | GitHub Actions, flutter test --coverage, lcov | 1 |
| Widget Test | IoT 제어 화면 WidgetTester, FakeController, pump vs pumpAndSettle | 1 |
| Widget Test + CI | Unit·Widget 커버리지 합산, lcov merge, Codecov 뱃지 | 1 |
| Golden Test | matchesGoldenFile, alchemist, IoT 카드 UI 스냅샷, CI 환경 폰트·DPR 불일치 해결 | 1 |
| Integration Test | integration_test E2E, 타이밍 이슈, CI 실행 전략 | 1 |
| E2E + Native | Patrol, BLE 권한 다이얼로그 자동화, 네이티브 OS 인터랙션 | 1 |
| TDD 실전 | Red-Green-Refactor, BleController 설계 변화, 레이어별 TDD 적합성 | 1 |
| 테스트 더블 패턴 | Dummy·Stub·Fake·Spy·Mock 비교, IoT 앱 레이어별 선택 기준, mocktail vs mockito | 1 |
| 비동기 테스트 | fakeAsync, Timer·Stream 타이밍 제어, MQTT 재연결·BLE 스캔 타임아웃 테스트 | 1 |
| Firebase 격리 | firebase_core 없이 Auth·RemoteConfig Fake로 단위 테스트 격리, Crashlytics no-op | 1 |

마지막 포스트: `_posts/2026/06/26/2026-06-26-20-00-00-814770-flutter-firebase-test-isolation-auth-remote-config-fake.md`  
다음 일련번호: `827115`

---

## 내부 링크 후보

본문에서 자연스럽게 연결할 수 있는 기존 포스트 목록. 관련 주제 작성 시 활용.

> **⚠️ `{% post_url %}` 사용 전 반드시 두 단계 확인:**
> 1. `find _posts -name "*슬러그*"` — 로컬 존재 확인
> 2. `git ls-files _posts/경로/파일명.md` — git push 여부 확인 (출력 없으면 미push = 빌드 실패)
> 같은 세션에서 방금 만든 포스트는 참조 금지. push 완료 후 다음 포스트에서만 참조 가능.

```
BLE 연결 기초    → _posts/2026/05/05/...-ble-bluetooth-basics-flutter-blue-plus-scanning.md
BLE 재연결 전략  → _posts/2026/05/11/...-ble-connection-stability-reconnect-error-handling.md
MQTT 기초        → _posts/2026/05/12/...-mqtt-protocol-fundamentals-iot-heart.md
AWS IoT 연결     → _posts/2026/05/13/...-aws-iot-core-connection-sigv4-authentication.md
클린 아키텍처    → _posts/2026/05/01/...-clean-architecture-large-flutter-project-structure.md
GetX 상태관리    → _posts/2026/05/02/...-getx-state-management-routing-flutter-iot.md
JWT 자동갱신     → _posts/2026/05/20/...-jwt-token-auto-refresh-session-management.md
```

---

## 기존 포스트 검색

```bash
grep -rl "KEYWORD" _posts/ --include="*.md"
find _posts -name "*.md" | sort | tail -10
```

---

## 글 유형별 참조 파일

| 유형 | 참조 파일 |
|------|-----------|
| 기술 구현형 | `_posts/2026/06/16/.../multi-environment-build-dev-stage-prod.md` |
| 회고·에세이형 | `_posts/2026/06/18/.../50-days-flutter-iot-app-retrospective-smarthome.md` |
| 기능 설계형 | `_posts/2026/06/03/.../space-member-management-family-smarthome.md` |
