---
layout: post
title: "Flutter 3.47 Impeller 오류 - OpenGL 셰이더 이미지가 거꾸로 보일 때"
description: "Flutter 3.47에서 Impeller OpenGL ES render-to-texture 방향이 바뀌어 FragmentShader 이미지가 뒤집힐 때, 구버전 호환 매크로와 출시 검증 방법을 정리한다."
date: 2026-10-21
tags: [Flutter, 성능최적화, Android, 테스트]
comments: true
share: true
---

![Flutter 3.47 Impeller와 OpenGL 셰이더 렌더링 점검](https://images.unsplash.com/photo-1518770660439-4636190af475?auto=format&fit=crop&w=1600&q=80)

Flutter 3.47로 올린 뒤 Android 일부 기기에서 필터·차트·게임 화면이 상하 반전된다면, 위젯 레이아웃보다 `FragmentShader`의 Y축 보정부터 확인하는 편이 맞다. 이번 변경은 Impeller의 OpenGL ES render-to-texture 저장 방향이 기존 bottom-up에서 top-down으로 바뀐 데서 생긴다. 일반 `Container`나 `Image`만 쓰는 앱에는 해당하지 않으며, 직접 렌더 타깃 텍스처를 샘플링하는 코드에만 적용된다.

## 어떤 앱이 영향을 받는가

| 코드 경로 | Flutter 3.47 영향 | 판단 |
| --- | --- | --- |
| 일반 Material·Cupertino 위젯 | 없음 | 셰이더 점검 불필요 |
| `FragmentShader`로 render target 샘플링 | 있음 | 기존 GLES Y-flip 제거 또는 조건부 처리 |
| `flutter_gpu` 실험 기능 | 있음 | Metal·Vulkan과 같은 좌표계로 확인 |
| 단순 이미지 디코딩 | 보통 없음 | render-to-texture 사용 여부만 확인 |

Flutter 공식 breaking change 기준으로 Impeller는 이제 OpenGL ES에서도 Metal·Vulkan과 같은 top-down 방향을 사용한다. 3.46 이하에서 GLES만 보정하려고 `uv.y = 1.0 - uv.y`를 넣었다면, 3.47에서는 그 코드가 오히려 화면을 뒤집는다. Flutter 3.47로 올린 뒤 한 플랫폼에서만 문제가 생기는 이유도 이 조건부 분기 때문이다.

## 새 버전만 지원한다면 Y-flip을 지운다

셰이더가 render target을 직접 읽는 경우에만 아래처럼 보정 코드를 제거한다. 코드가 필요한 이유는 Flutter 위젯의 좌표계를 바꾸는 것이 아니라, 텍스처 샘플링 단계에서 중복된 반전을 없애기 위해서다.

```glsl
#include <flutter/runtime_effect.glsl>

uniform vec2 u_size;
uniform sampler2D u_texture;
out vec4 fragColor;

void main() {
  vec2 uv = FlutterFragCoord().xy / u_size;
  fragColor = texture(u_texture, uv);
}
```

앱이 Flutter 3.46 이하와 3.47 이상을 함께 지원해야 한다면 단순히 flip을 삭제하지 말고 공식 호환 매크로를 사용한다. 이전 엔진에서는 GLES 보정이 필요하고, 새 엔진에서는 필요하지 않은 상황을 한 소스에서 분리하는 방식이다.

```glsl
void main() {
  vec2 uv = FlutterFragCoord().xy / u_size;

#if defined(IMPELLER_TARGET_OPENGLES) && \
    !defined(IMPELLER_OPENGLES_UNFLIPPED_DEPRECATED)
  uv.y = 1.0 - uv.y;
#endif

  fragColor = texture(u_texture, uv);
}
```

## 출시 전에 확인할 항목

문서만 보고 모든 Android 기기에서 같은 결과라고 단정하면 안 된다. Impeller 백엔드와 앱의 실제 셰이더 경로를 각각 확인해야 한다.

- [ ] `FragmentShader` 또는 `flutter_gpu`가 render target 텍스처를 샘플링하는지 검색했다.
- [ ] `IMPELLER_TARGET_OPENGLES`와 `uv.y` 보정 코드를 함께 찾았다.
- [ ] Flutter 3.46 이하와 3.47 이상에서 의도한 방향이 같은지 비교했다.
- [ ] Android OpenGL ES 경로와 iOS Metal 경로를 각각 확인했다.
- [ ] 일반 위젯 화면까지 셰이더 수정 대상으로 잘못 넓히지 않았다.

특히 스크린샷 golden test만 통과했다고 끝내면 안 된다. 이 변경은 백엔드별로 다른 결과를 만들 수 있으므로, 셰이더가 적용된 화면을 Android 실기기 또는 해당 renderer를 사용하는 환경에서 확인해야 한다. 내가 확인한 공식 기준으로 이 변경은 Flutter 3.47부터 적용되며, 그 전 버전과의 호환이 필요할 때만 매크로 분기가 필요하다.

짧게 정리하면, Flutter 3.47에서 이미지가 거꾸로 보이는 경우 `SafeArea`나 위젯 회전부터 고칠 문제가 아니다. 직접 텍스처를 읽는 GLES 셰이더의 오래된 Y-flip을 제거하고, 구버전까지 지원한다면 `IMPELLER_OPENGLES_UNFLIPPED_DEPRECATED` 조건으로 나눠야 한다. 일반 Flutter 2D UI만 사용하는 앱은 이 변경 때문에 코드를 수정할 필요가 없다.

참고 문서: [Flutter OpenGL ES render-to-texture 변경 안내](https://docs.flutter.dev/release/breaking-changes/opengles-render-to-texture-top-down), [Flutter 3.47 릴리스 노트](https://docs.flutter.dev/release/release-notes/release-notes-3.47.0)
