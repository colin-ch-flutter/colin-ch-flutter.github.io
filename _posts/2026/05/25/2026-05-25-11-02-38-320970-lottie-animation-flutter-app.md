---
layout: post
title: "Lottie 애니메이션으로 앱에 생동감 불어넣기"
description: " "
date: 2026-05-25
tags: [Flutter, Lottie, 애니메이션, UX]
comments: true
share: true
---
![애니메이션 디자인](https://images.unsplash.com/photo-1626785774625-0b1c2c4eab67?w=800&q=80)

## Lottie가 뭔가

Lottie는 Airbnb에서 만든 애니메이션 포맷이다. Adobe After Effects로 만든 애니메이션을 JSON 파일로 내보내서 앱에서 그대로 재생할 수 있다.

GIF나 비디오보다 파일 크기가 작고, 벡터 기반이라 어떤 크기에서도 선명하다. 색상도 코드에서 동적으로 바꿀 수 있다.

## 패키지 설정

```yaml
dependencies:
  lottie: ^3.3.2

flutter:
  assets:
    - assets/lottie/
```

## 기본 사용법

```dart
// 단순 재생
Lottie.asset('assets/lottie/loading.json')

// 크기와 반복 제어
Lottie.asset(
  'assets/lottie/loading.json',
  width: 100,
  height: 100,
  repeat: true,
  animate: true,
)
```

## 로딩 화면 애니메이션

스마트홈 앱에는 두 가지 로딩 애니메이션이 있다. 1~50%와 51~99%용으로 분리되어 있는 로딩 휠이다:

```dart
// presentation/core/common/loading_widget.dart
class LoadingWidget extends StatelessWidget {
  const LoadingWidget({super.key});

  @override
  Widget build(BuildContext context) {
    return Obx(() {
      final loadingState = Get.find<LoadingController>().state.value;
      if (!loadingState.isVisible) return const SizedBox.shrink();

      return Container(
        color: Colors.black.withOpacity(0.3),
        child: Center(
          child: _LoadingAnimation(progress: loadingState.progress),
        ),
      );
    });
  }
}

class _LoadingAnimation extends StatefulWidget {
  final double progress;
  const _LoadingAnimation({required this.progress});

  @override
  State<_LoadingAnimation> createState() => _LoadingAnimationState();
}

class _LoadingAnimationState extends State<_LoadingAnimation>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this);
    _controller.repeat();
  }

  @override
  Widget build(BuildContext context) {
    // 진행률에 따라 다른 애니메이션 파일 사용
    final assetName = widget.progress <= 50
        ? '10_loadingwheel_1to50'
        : '10_loadingwheel_51to99';

    return Lottie.asset(
      'assets/lottie/$assetName.json',
      controller: _controller,
      width: 80,
      height: 80,
    );
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }
}
```

## 애니메이션 재생 제어

특정 순간에만 애니메이션을 재생하고 완료 후 멈추는 경우:

```dart
class SuccessAnimation extends StatefulWidget {
  final VoidCallback onComplete;
  const SuccessAnimation({super.key, required this.onComplete});

  @override
  State<SuccessAnimation> createState() => _SuccessAnimationState();
}

class _SuccessAnimationState extends State<SuccessAnimation>
    with SingleTickerProviderStateMixin {
  late AnimationController _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this);
    _controller.addStatusListener((status) {
      if (status == AnimationStatus.completed) {
        widget.onComplete();
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return Lottie.asset(
      'assets/lottie/success_check.json',
      controller: _controller,
      onLoaded: (composition) {
        _controller
          ..duration = composition.duration
          ..forward(); // 한 번만 재생
      },
    );
  }
}
```

## Figma에서 Lottie까지

디자이너가 Figma에서 작업하고, 개발자에게 Lottie JSON 파일을 넘겨주는 워크플로우를 사용했다. Figma에서 LottieFiles 플러그인으로 직접 내보낼 수 있다.

파일을 받으면 `assets/lottie/`에 넣고 `pubspec.yaml`에 등록하면 된다. 처음에는 `pubspec.yaml`에 파일을 등록하는 걸 빠뜨려서 "asset not found" 오류를 몇 번 겪었다.

## 다크모드에서 색상 변경

Lottie 파일의 특정 색상을 코드에서 동적으로 바꿀 수 있다:

```dart
Lottie.asset(
  'assets/lottie/loading.json',
  delegates: LottieDelegates(
    values: [
      ValueDelegate.color(
        ['loading_circle', '**'],
        value: Theme.of(context).colorScheme.primary,
      ),
    ],
  ),
)
```

다크모드에서 애니메이션 색상도 테마에 맞게 바꿔야 할 때 유용하다.

---

다음 편은 다크모드를 완벽하게 지원하는 ThemeData 설계 방법이다.
