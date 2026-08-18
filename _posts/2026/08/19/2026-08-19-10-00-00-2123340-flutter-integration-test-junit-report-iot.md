---
layout: post
title: "Flutter integration_test JUnit 리포트 - GitHub Actions에서 IoT 테스트 실패 원인 남기기"
description: "Flutter integration_test 결과를 JUnit XML로 변환하고 GitHub Actions Summary와 아티팩트에 남기는 방법을 정리했다. IoT 테스트의 실패 케이스와 재시도 판단까지 다룬다."
date: 2026-08-19
tags: [Flutter, Dart, integration_test, CI/CD, IoT, GitHub Actions]
comments: true
share: true
---

![Flutter integration_test JUnit 리포트와 GitHub Actions IoT 테스트 결과](/assets/images/flutter-integration-test-parallel-ci-iot.png)

Flutter integration_test를 CI에서 돌릴 때 초록색 또는 빨간색만 남기면 원인 분석이 느리다. 특히 IoT 앱은 BLE 스캔, MQTT ACK, 알림 탭이 섞여 있어서 실패한 테스트 이름과 실행 시간을 함께 봐야 한다. 이번에는 테스트 결과를 JUnit XML로 만들고 GitHub Actions Summary와 아티팩트에 보존했다.

## 실패했는데 로그를 다시 찾고 있었다

처음에는 `flutter test integration_test`의 콘솔 로그만 저장했다. 로컬에서는 충분했지만 CI에서는 문제가 달랐다. 여러 테스트가 병렬로 실행되면 로그 순서가 섞이고, 재실행된 테스트는 어느 시도의 결과인지 알기 어려웠다.

| 출력 | 확인할 수 있는 것 | 한계 |
|---|---|---|
| 콘솔 로그 | 상세 stack trace | 테스트별 요약이 약함 |
| 스크린샷 | 실패 당시 화면 | 테스트명·소요 시간 연결이 번거로움 |
| JUnit XML | 테스트별 성공·실패·시간 | 변환 도구가 필요함 |
| GitHub Summary | PR에서 바로 보는 요약 | 긴 로그는 아티팩트로 분리 |

이 그림에서 봐야 할 부분은 테스트 자체보다 실행 결과를 CI에서 다시 읽을 수 있는 형태로 바꾸는 흐름이다.

## 테스트 실행과 결과 파일 분리

`integration_test`는 기본적으로 JUnit XML을 직접 출력하지 않는다. 그래서 실행 로그를 파일로 저장하고, 작은 Dart 스크립트가 테스트 경계를 읽어 XML을 만든다. 실제 프로젝트에서는 테스트 파일별 프로세스를 나눠 파일이 덮어써지지 않게 했다.

테스트마다 독립적인 로그와 결과 경로를 지정하는 코드다.

```yaml
# .github/workflows/integration-test.yml
- name: Run integration tests
  shell: bash
  run: |
    mkdir -p build/test-results build/test-logs
    flutter test integration_test \
      --device-id emulator-5554 \
      --reporter expanded \
      2>&1 | tee build/test-logs/integration.log
  continue-on-error: true

- name: Convert test result to JUnit XML
  if: always()
  run: dart tool/integration_test_junit.dart
```

여기서 `continue-on-error`는 테스트를 성공으로 처리한다는 뜻이 아니다. 결과 변환과 로그 업로드 단계를 실행하기 위해 셸을 계속 진행시키는 옵션이다. 실제 job의 마지막에 XML 결과를 검사해 실패 상태를 다시 반환해야 한다.

## IoT 테스트용 JUnit 변환 기준

테스트 이름은 `testWidgets`의 설명을 그대로 쓰고, `FAIL` 로그가 포함된 블록은 `<failure>`로 넣었다. 중요한 건 XML을 예쁘게 만드는 일이 아니라 테스트 이름, 실패 메시지, duration을 잃지 않는 것이다.

```dart
// tool/integration_test_junit.dart
import 'dart:io';
import 'package:xml/xml.dart';

void main() {
  final log = File('build/test-logs/integration.log').readAsStringSync();
  final cases = <XmlElement>[];

  for (final block in log.split(RegExp(r'(?=00:00 \+\d+:)'))) {
    final name = RegExp(r'\+\d+: (.+)').firstMatch(block)?.group(1);
    if (name == null) continue;

    final failed = block.contains('Some tests failed') ||
        block.contains('Test failed');
    cases.add(XmlElement(XmlName('testcase'), attributes: [
      XmlAttribute(XmlName('name'), name),
    ], children: failed
        ? [XmlElement(XmlName('failure'), children: [
            XmlText(block.trim()),
          ])]
        : []);
  }

  final document = XmlDocument([
    XmlElement(XmlName('testsuite'), attributes: [
      XmlAttribute(XmlName('name'), 'flutter-integration-test'),
      XmlAttribute(XmlName('tests'), '${cases.length}'),
    ], children: cases),
  ]);
  File('build/test-results/integration.xml')
    ..createSync(recursive: true)
    ..writeAsStringSync(document.toXmlString(pretty: true));
}
```

실제 로그 포맷은 Flutter 버전이나 reporter 설정에 따라 달라진다. 위 정규식을 복사해서 끝내면 안 되고, 프로젝트에서 실패 1건과 성공 1건을 고정 fixture로 만들어 변환 결과를 확인해야 한다. 내가 처음 만든 변환기는 `00:00 +1` 줄만 찾다가 assertion stack trace를 테스트 이름으로 오인했다.

## GitHub Actions에서 사람이 읽는 결과 만들기

XML은 Codecov나 테스트 리포터가 읽기 좋지만, PR을 보는 사람에게는 요약 표가 더 빠르다. `actions/github-script`로 실패 수와 XML 내용을 간단히 출력하고 원본은 아티팩트로 올렸다.

```yaml
- name: Publish test report
  if: always()
  uses: dorny/test-reporter@v1
  with:
    name: Flutter integration_test
    path: build/test-results/integration.xml
    reporter: java-junit
    fail-on-error: false

- name: Upload integration test artifacts
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: integration-test-${{ github.run_number }}
    path: |
      build/test-results/
      build/test-logs/
      screenshots/
    retention-days: 14
```

`fail-on-error: false`로 리포터가 XML 파싱에 실패해도 원래 테스트 실패 원인이 가려지지 않게 했다. 대신 별도 셸 단계에서 실제 종료 코드를 검사한다. 테스트가 실패했는데 리포터 오류가 최종 원인처럼 보이면 디버깅 방향이 완전히 틀어진다.

## 재시도는 증거를 남긴 뒤에

IoT integration_test에서 실패한 job을 무조건 재시도하면 flaky 비율만 숨겨진다. JUnit의 duration과 테스트별 실패 횟수를 모아 다음 기준으로 나눴다.

- 같은 테스트가 항상 실패하면 앱 초기화나 Fake 의존성 문제로 분류한다.
- 실행 시간이 비정상적으로 길고 MQTT ACK가 없으면 타임아웃 로그를 확인한다.
- 한 번만 실패해도 스크린샷과 `integration.log`가 있으면 재현 여부를 별도로 판단한다.
- 아티팩트가 없으면 재시도 전에 CI 단계부터 고친다.

짧게 정리하면 Flutter integration_test의 CI 품질은 테스트 개수보다 실패 결과를 얼마나 구조화해서 남기느냐에 달려 있다. JUnit XML은 도구 연동용, Summary는 빠른 판단용, 로그와 스크린샷은 원인 분석용으로 역할을 나누면 된다. 그래야 “CI에서만 실패했다”를 다시 실행하는 데서 끝내지 않고, 어떤 IoT 경계가 흔들렸는지 확인할 수 있다.
