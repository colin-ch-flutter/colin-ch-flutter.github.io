---
layout: post
title: "Flutter Unit Test 시작하기 - Repository 레이어부터 짜는 이유"
description: "GetX + Clean Architecture로 만든 Flutter 앱에서 Unit Test를 처음 시작한다면 Repository 레이어가 답이다. Fake 구현체를 만들고 Controller 동작을 검증하는 실제 코드를 정리했다."
date: 2026-06-23
tags: [Flutter, UnitTest, GetX, CleanArchitecture, Repository패턴]
comments: true
share: true
---
![테스트 코드](https://images.unsplash.com/photo-1555949963-ff9fe0c870eb?w=800&q=80)

[50일 회고]({% post_url 2026-06-18-11-50-50-617250-50-days-flutter-iot-app-retrospective-smarthome %})에서 솔직하게 털어놨다. "이번 프로젝트에서 Unit Test를 충분히 못 썼다." 쓰고 나서 찜찜했는데, 그냥 찜찜함으로 끝내기엔 너무 명확한 숙제였다. 지금부터 시작한다.

## 왜 Repository부터인가

처음엔 Controller 테스트를 쓰려다 막혔다. GetX Controller는 `Get.put()`으로 등록하고, 내부에서 API 호출까지 하는데, 이걸 테스트하려면 네트워크 없이도 돌아가야 한다. 근데 막상 보니 이미 준비가 다 돼 있었다.

[Repository 패턴으로 데이터 레이어를 분리]({% post_url 2026-05-19-11-20-20-246900-repository-pattern-clean-data-layer %})해뒀기 때문에 Controller는 `SpaceRepository` 인터페이스만 안다. 실제 구현체가 API를 치든 Fake 데이터를 반환하든 Controller는 모른다. 이게 테스트를 쉽게 만드는 핵심이다.

## Fake Repository 만들기

먼저 `SpaceRepository` 인터페이스의 Fake 구현체를 만든다.

```dart
// test/fake/fake_space_repository.dart
class FakeSpaceRepository implements SpaceRepository {
  final List<Space> _spaces = [];
  bool shouldFail = false;

  @override
  Future<Result<List<Space>>> getSpaces() async {
    if (shouldFail) return Result.failure('서버 오류');
    return Result.success(List.from(_spaces));
  }

  @override
  Future<Result<Space>> createSpace({required String name}) async {
    if (shouldFail) return Result.failure('생성 실패');
    final space = Space(
      id: 'fake-${_spaces.length}',
      name: name,
      members: [],
      deviceIds: [],
      isOwner: true,
    );
    _spaces.add(space);
    return Result.success(space);
  }

  @override
  Future<Result<void>> leaveSpace({required String spaceId}) async {
    _spaces.removeWhere((s) => s.id == spaceId);
    return Result.success(null);
  }
}
```

`shouldFail` 플래그 하나로 성공/실패 시나리오를 전환할 수 있다. 처음엔 이걸 각 메서드마다 따로 만들었는데, 대부분의 케이스에서 "이 Repository 전체가 실패하는 상황"을 테스트하는 게 더 유용했다.

## Controller 테스트 작성

GetX Controller 테스트는 `setUp`에서 Fake를 주입하고, `tearDown`에서 `Get.reset()`으로 정리한다.

```dart
// test/controller/space_add_controller_test.dart
void main() {
  late FakeSpaceRepository fakeRepo;
  late SpaceAddController controller;

  setUp(() {
    fakeRepo = FakeSpaceRepository();
    Get.put<SpaceRepository>(fakeRepo);
    controller = Get.put(SpaceAddController());
  });

  tearDown(Get.reset);

  group('createSpace', () {
    test('이름이 비어있으면 Repository를 호출하지 않는다', () async {
      controller.nameController.text = '';
      await controller.createSpace();

      expect(fakeRepo._spaces, isEmpty);
    });

    test('성공 시 공간이 생성된다', () async {
      controller.nameController.text = '우리집';
      await controller.createSpace();

      expect(fakeRepo._spaces.length, 1);
      expect(fakeRepo._spaces.first.name, '우리집');
    });

    test('서버 실패 시 기존 상태가 유지된다', () async {
      fakeRepo.shouldFail = true;
      controller.nameController.text = '우리집';
      await controller.createSpace();

      expect(fakeRepo._spaces, isEmpty);
    });

    test('로딩 중에는 isLoading이 true다', () async {
      final loadingValues = <bool>[];
      ever(controller.isLoading, loadingValues.add);

      controller.nameController.text = '우리집';
      await controller.createSpace();

      expect(loadingValues, containsAllInOrder([true, false]));
    });
  });
}
```

마지막 케이스가 처음엔 왜 필요한지 몰랐는데, 실제로 이중 탭 버그를 여기서 잡았다. 버튼 두 번 누르면 `isLoading`이 이미 true인데 또 API를 호출했던 것.

## 실행

```bash
flutter test test/controller/space_add_controller_test.dart
```

처음 돌렸을 때 `Get.reset()`을 빠뜨려서 테스트 간 상태가 오염됐다. 두 번째 테스트가 첫 번째에서 등록한 Controller를 그대로 받아서 이상한 값이 나왔다. `tearDown(Get.reset)` 한 줄이 필수다.

## 어디까지 테스트해야 하나

Repository 레이어 자체는 테스트하지 않는다. 실제 API 서버와 통합 테스트가 필요한 영역이고, Unit Test로 커버하면 Mock과 실서버 동작이 달라서 오히려 위험하다. (50일 회고에서 이미 언급한 것처럼, 작년에 이게 문제가 됐다.)

Controller가 Repository 인터페이스를 통해 올바르게 동작하는지, 상태가 예상대로 바뀌는지 — 이것만 검증한다.

---

다음 편은 MemberManagementController 테스트다. 멤버 삭제 확인 다이얼로그처럼 비동기 UI 상호작용이 섞인 테스트는 조금 다른 접근이 필요하다.
