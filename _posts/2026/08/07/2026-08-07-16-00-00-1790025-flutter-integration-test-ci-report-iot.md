---
layout: post
title: "Flutter integration_test CI 리포트 - GitHub Actions에서 실패 원인 남기기"
description: "Flutter integration_test 결과를 flutter test --machine으로 수집하고 GitHub Actions Summary와 실패 아티팩트로 남기는 방법을 IoT 앱 기준으로 정리했다."
date: 2026-08-07
tags: [Flutter, integration_test, IoT, CI/CD, GitHub Actions]
comments: true
share: true
---

![Flutter integration_test CI 테스트 리포트와 IoT 앱](/assets/images/flutter-integration-test-ci-report-iot.png)

Flutter integration_test를 CI에서 돌리는 것만으로는 부족하다. 테스트가 실패했을 때 GitHub Actions 로그를 처음부터 뒤져야 한다면, 자동화가 아니라 원격 실행에 가깝다. `flutter test --machine` 이벤트를 파일로 남기고, Summary에는 실패한 테스트만 보여주는 구성이 실제 디버깅 시간을 줄였다.

## 로그 전체 출력이 불편했던 이유

처음에는 workflow에서 아래 명령만 실행했다.

```yaml
- name: Run integration tests
  run: flutter test integration_test -d emulator-5554
```

로컬에서는 괜찮았지만 CI에서는 문제가 달랐다. BLE Fake 서비스에서 타임아웃이 나면 마지막 예외만 보이고, 어느 테스트가 몇 초 동안 멈췄는지는 로그 중간에 묻혔다. 테스트가 20개를 넘으니 재현을 위해 전체 로그를 다시 읽는 시간이 더 길어졌다.

그래서 사람이 읽는 일반 출력과 기계가 읽는 이벤트 출력을 분리했다.

## machine 이벤트를 아티팩트로 저장하기

`--machine`은 테스트 시작, 종료, 성공·실패 상태를 JSON 이벤트 스트림으로 출력한다. 아래처럼 표준 출력을 파일로 저장하고, 명령 자체가 실패해도 후속 분석 단계가 실행되게 한다.

```yaml
- name: Run integration tests
  id: integration
  shell: bash
  run: |
    set +e
    flutter test --machine integration_test \
      -d emulator-5554 > integration-test.json
    echo "exit_code=$?" >> "$GITHUB_OUTPUT"
    exit 0

- name: Upload test event stream
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: integration-test-events
    path: integration-test.json
    if-no-files-found: error
```

여기서 `set +e`를 빼면 테스트 실패 즉시 step이 끝나서 JSON 파일은 남아도 Summary 생성 단계가 실행되지 않는다. 대신 마지막에 원래 종료 코드를 별도로 보관하고, 리포트가 만들어진 뒤 workflow를 실패 처리한다.

## Summary에는 실패 목록만 남긴다

전체 JSON을 그대로 출력하면 Summary도 읽기 어려워진다. 이벤트의 `testDone` 중 `error`가 있는 항목만 추려서 링크와 함께 기록한다. `jq`는 GitHub-hosted runner에 기본 설치돼 있다.

```yaml
- name: Write GitHub Actions summary
  if: always()
  shell: bash
  run: |
    {
      echo "## Flutter integration_test 결과"
      echo
      echo "| 테스트 | 결과 | 오류 |"
      echo "|---|---|---|"
      jq -r '
        select(.type == "testDone") |
        "| `\(.testID // "unknown")` | \(if .error then "❌ 실패" else "✅ 통과" end) | \(.error // "-") |"
      ' integration-test.json
      echo
      echo "> 원본 이벤트와 스크린샷은 workflow artifact에서 확인한다."
    } >> "$GITHUB_STEP_SUMMARY"

- name: Fail workflow when tests failed
  if: always() && steps.integration.outputs.exit_code != '0'
  run: exit 1
```

실제 JSON에서는 `testDone` 이벤트의 필드가 Flutter 버전에 따라 조금 달라질 수 있다. 처음 적용할 때는 `jq` 필터를 고정하기 전에 `head -n 20 integration-test.json`으로 이벤트 모양을 확인하는 편이 안전하다.

## JUnit 리포트는 기존 CI 화면에 연결한다

팀에서 이미 JUnit 리포트를 읽고 있다면 machine JSON을 JUnit XML로 변환해 `test-reporter` 같은 액션에 넘길 수 있다. 중요한 건 변환 도구보다 파일의 생존 조건이다.

| 파일 | 용도 | 보관 조건 |
|---|---|---|
| `integration-test.json` | 원본 이벤트, 상세 디버깅 | 항상 업로드 |
| `test-results.xml` | PR 테스트 요약 | 변환 성공 여부 확인 |
| `screenshots/**` | 실패 화면과 상태 | `if: always()` |

내 경우에는 처음에 `if: failure()`를 붙였다가, 테스트 명령이 너무 일찍 종료된 경우 스크린샷 수집 step까지 건너뛰는 실수를 했다. 테스트 관련 수집 단계는 성공·실패와 상관없이 실행해야 한다. 리포트가 없어도 실패 원인을 추적할 수 있도록 원본 JSON을 최후의 증거로 남기는 게 핵심이다.

## 짧게 정리하면

- `flutter test --machine` 결과를 JSON으로 저장한다.
- 테스트 실패와 별개로 Summary·스크린샷·원본 파일을 수집한다.
- Summary에는 실패 목록만, artifact에는 전체 이벤트를 남긴다.
- 마지막 단계에서 원래 테스트 종료 코드로 workflow를 실패 처리한다.

CI의 목적은 테스트를 대신 눌러주는 데 있지 않다. 실패했을 때 “무슨 테스트가, 어느 단계에서, 어떤 상태로 끝났는지”를 바로 알려주는 데 있다.
