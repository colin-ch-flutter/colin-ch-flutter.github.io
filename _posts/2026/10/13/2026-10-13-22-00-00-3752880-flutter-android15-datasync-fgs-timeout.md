---
layout: post
title: "Flutter Android 15 Foreground Service timeout 오류 - dataSync 6시간 제한과 WorkManager 전환"
description: "Flutter 앱이 Android 15에서 dataSync Foreground Service timeout으로 종료될 때, 6시간 제한의 적용 조건과 재현 명령, 플러그인별 중단·WorkManager 전환 기준을 정리한다."
date: 2026-10-13
tags: [Flutter, Android, 배포·운영, 성능최적화, 테스트]
comments: true
share: true
---

![Flutter Android 15 Foreground Service timeout과 백그라운드 동기화 점검](https://images.unsplash.com/photo-1607252650355-f7fd0460ccdb?auto=format&fit=crop&w=1600&q=80)

Android 15에서 `dataSync` Foreground Service가 오래 살아 있어 `ForegroundServiceDidNotStopInTimeException`이나 `Time limit already exhausted`가 발생한다면, AndroidManifest 권한을 하나 더 넣는 문제가 아니다. 24시간 동안 누적 6시간 제한을 넘긴 작업은 서비스 내부에서 `onTimeout()`을 처리해 스스로 종료하거나, 오래 걸리는 동기화 자체를 WorkManager 같은 시스템 작업으로 바꿔야 한다. 짧은 업로드·다운로드만 수행하는 앱에는 해당하지 않을 수 있지만, Flutter 앱에서 플러그인이 서비스를 대신 띄우고 중단 시점을 숨기는 경우에는 출시 전에 확인해야 한다.

## Android 15에서 갑자기 release만 죽는 이유

Android 공식 문서 기준으로 `dataSync`와 `mediaProcessing` 타입은 Android 15에서 24시간당 총 6시간만 실행할 수 있다. 이 시간은 한 번의 6시간이 아니라 같은 타입 서비스의 누적 시간이다. 제한에 도달하면 시스템이 `Service.onTimeout(int, int)`을 호출하고, 몇 초 안에 `stopSelf()`하지 않으면 프로세스 오류로 이어진다.

| 상황 | 판단 | 대응 |
|---|---|---|
| 앱 화면에서 사용자가 눌러 시작한 짧은 동기화 | 6시간 제한에 닿을 가능성이 낮다 | 작업 완료 즉시 서비스 종료 |
| 백그라운드에서 파일 동기화를 계속 유지 | `dataSync` 제한 대상 | 청크 작업과 재개 지점 설계 |
| 주기적으로 서버를 확인하는 작업 | 상시 서비스가 필요하지 않을 수 있다 | WorkManager 또는 서버 푸시 검토 |
| Android 14의 타입·권한 누락 | 다른 원인 | `foregroundServiceType`과 타입별 권한 점검 |

여기서 Android 14의 `MissingForegroundServiceTypeException`과 Android 15의 timeout을 섞으면 안 된다. 전자는 서비스를 시작하는 순간의 선언·권한 오류이고, 후자는 정상적으로 시작한 서비스가 오래 실행된 뒤의 수명 제한이다.

## Flutter에서 재현 가능한 확인 방법

실제 6시간을 기다리지 않고 Android 15 기기 또는 에뮬레이터에서 제한을 짧게 줄여 종료 경로를 확인할 수 있다. 아래 명령은 패키지 ID를 실제 앱 ID로 바꿔 실행한다.

```bash
# Android 15 기기에서 dataSync timeout 기능을 켠다.
adb shell am compat enable FGS_INTRODUCE_TIME_LIMITS com.example.app

# 테스트 시간을 60초로 줄인다.
adb shell device_config put activity_manager \
  data_sync_fgs_timeout_duration 60000

# 이후 앱에서 dataSync 서비스를 시작하고 백그라운드로 보낸다.
adb logcat -v color | rg "ForegroundService|dataSync|onTimeout|DidNotStop"
```

이 검증은 Flutter의 Dart 타이머가 아니라 Android 서비스 수명 제한을 확인하는 절차다. 따라서 `Timer.periodic`으로 Dart 코드를 정리하는 것만으로는 해결되지 않는다. `flutter_foreground_task`, 다운로드·동기화 플러그인처럼 네이티브 Service를 내부에서 사용하는 경우에는 플러그인이 `onTimeout()`에서 중단하도록 지원하는지 확인해야 한다.

직접 Android Service를 유지하는 앱이라면 timeout 콜백에서 저장할 상태를 남기고 즉시 종료한다. 아래 코드는 “남은 파일 목록을 저장한 뒤 서비스 종료”라는 최소 형태다.

```kotlin
override fun onTimeout(startId: Int, fgsType: Int) {
    // 다음 실행에서 이어 받을 파일·offset을 저장한다.
    syncRepository.saveCheckpoint()
    stopSelf()
}
```

플러그인 소스에 이 콜백이 없으면 앱의 `MainActivity`에서 임의로 가로채기 어렵다. 이 경우에는 플러그인 업데이트 여부를 확인하고, 서비스가 계속 살아 있어야 하는지부터 다시 판단한다. 네트워크가 끊겨도 재개 가능한 파일 전송이라면 한 번의 긴 서비스 대신 작은 작업 단위로 나누는 편이 안전하다.

## WorkManager로 바꿔야 하는 기준

화면이 닫혀도 계속 실행해야 한다는 요구만으로 Foreground Service를 선택하면 안 된다. 백그라운드 동기화가 즉시 완료될 필요가 없다면 WorkManager에 작업을 등록하고, 네트워크 조건과 재시도 정책을 맡기는 편이 Android의 제한과 더 잘 맞는다.

반대로 음악 재생, 사용자가 보고 있는 위치 추적처럼 진행 중인 알림과 지속 실행이 기능 그 자체라면 Foreground Service가 필요할 수 있다. 그래도 `dataSync` 타입으로 분류할 수 없는 작업을 억지로 `dataSync`에 넣지 말고, 실제 기능에 맞는 타입과 정책을 확인해야 한다.

출시 전에는 아래 항목을 체크한다.

- [ ] Android 15 기기에서 `targetSdk 35` release 빌드를 실행했다.
- [ ] 서비스가 `dataSync`인지, 다른 타입인지 실제 작업과 맞춰 설명할 수 있다.
- [ ] 서비스 시작 후 정상 완료·네트워크 실패·앱 백그라운드 전환 때 모두 종료된다.
- [ ] `onTimeout()` 또는 플러그인에 대응하는 중단 경로가 있다.
- [ ] 60초로 줄인 테스트에서 checkpoint 저장 후 재시작이 되는지 확인했다.
- [ ] 상시 실행이 필요 없는 작업은 WorkManager로 분리했다.

짧게 정리하면 `Android 14 선언 오류`와 `Android 15 dataSync 누적 timeout`은 별개다. Flutter 플러그인이 서비스를 대신 관리하더라도 최종 책임은 앱의 출시 조건에 있다. 2026년 10월 13일 확인 기준으로 Android 공식 문서는 6시간 제한, `onTimeout()` 처리, WorkManager 대안을 모두 안내하므로, 로그에 timeout이 보이면 권한을 반복해서 추가하기보다 서비스 수명과 작업 모델부터 바꿔야 한다.

참고 문서: [Android Developers - Foreground service timeouts](https://developer.android.com/develop/background-work/services/fgs/timeout), [Android 15 behavior changes](https://developer.android.com/about/versions/15/behavior-changes-15), [Flutter Workmanager 문서](https://github.com/fluttercommunity/flutter_workmanager/blob/main/docs/quickstart.mdx)
