---
layout: post
title: "공간(Space)과 멤버 관리 - 가족이 함께 쓰는 스마트홈"
description: " "
date: 2026-06-03
tags: [Flutter, 공간관리, 멤버관리, 스마트홈, UX]
comments: true
share: true
---

# 공간(Space)과 멤버 관리 - 가족이 함께 쓰는 스마트홈

![스마트홈 가족](https://images.unsplash.com/photo-1558618047-3c8c76ca7d13?w=800&q=80)

## 공간(Space) 개념

스마트홈 앱에서 "공간"은 하나의 집, 별장, 사무실 같은 장소를 의미한다. 사용자가 여러 공간을 가질 수 있고, 각 공간에 여러 기기를 등록할 수 있다.

공간의 장점은 멤버 관리다. 공간을 만들고 가족을 초대하면, 같은 공간에 등록된 모든 기기를 공유할 수 있다. 아빠가 밖에서 앱으로 IoT 기기를 켜도 되고, 엄마도 켤 수 있다.

## 데이터 구조

```dart
class Space {
  final String id;
  final String name;
  final List<SpaceMember> members;
  final List<String> deviceIds;
  final bool isOwner;

  const Space({
    required this.id,
    required this.name,
    required this.members,
    required this.deviceIds,
    required this.isOwner,
  });
}

class SpaceMember {
  final String userId;
  final String nickname;
  final MemberRole role;
  final String? profileImageUrl;

  const SpaceMember({
    required this.userId,
    required this.nickname,
    required this.role,
    this.profileImageUrl,
  });
}

enum MemberRole { owner, admin, member }
```

## 공간 생성

```dart
class SpaceAddController extends GetxController {
  final TextEditingController nameController = TextEditingController();
  final RxBool isLoading = false.obs;

  Future<void> createSpace() async {
    final name = nameController.text.trim();
    if (name.isEmpty) {
      Get.snackbar('오류', '공간 이름을 입력해주세요');
      return;
    }

    isLoading.value = true;
    final result = await _apiRepo.createSpace(name: name);
    isLoading.value = false;

    result.fold(
      onSuccess: (space) {
        Get.back(result: space);
        Get.snackbar('완료', '${space.name} 공간이 생성됐습니다');
      },
      onFailure: (msg) => Get.snackbar('오류', msg),
    );
  }
}
```

## 멤버 초대

초대 링크를 생성하고 공유하는 방식으로 멤버를 초대한다:

```dart
class MemberInviteController extends GetxController {
  final String spaceId;
  final RxString inviteLink = ''.obs;
  final RxBool isLoading = false.obs;

  MemberInviteController({required this.spaceId});

  @override
  void onInit() {
    super.onInit();
    _generateInviteLink();
  }

  Future<void> _generateInviteLink() async {
    isLoading.value = true;
    final result = await _apiRepo.generateInviteLink(spaceId: spaceId);
    isLoading.value = false;

    result.fold(
      onSuccess: (link) => inviteLink.value = link,
      onFailure: (msg) => Get.snackbar('오류', msg),
    );
  }

  Future<void> shareLink() async {
    if (inviteLink.value.isEmpty) return;
    await Share.share(
      '스마트홈 공간에 초대합니다!\n${inviteLink.value}',
      subject: '스마트홈 초대',
    );
  }

  Future<void> copyLink() async {
    await Clipboard.setData(ClipboardData(text: inviteLink.value));
    Get.snackbar('복사됨', '초대 링크가 클립보드에 복사됐습니다');
  }
}
```

## 멤버 권한 관리

오너만 다른 멤버를 강제 퇴장시킬 수 있다:

```dart
class MemberManagementController extends GetxController {
  final String spaceId;

  Future<void> removeMember(String userId) async {
    final confirm = await Get.dialog<bool>(
      AlertDialog(
        title: const Text('멤버 삭제'),
        content: const Text('이 멤버를 공간에서 제거하시겠습니까?'),
        actions: [
          TextButton(onPressed: () => Get.back(result: false), child: const Text('취소')),
          TextButton(
            onPressed: () => Get.back(result: true),
            style: TextButton.styleFrom(foregroundColor: Colors.red),
            child: const Text('삭제'),
          ),
        ],
      ),
    );

    if (confirm != true) return;

    final result = await _apiRepo.removeMember(
      spaceId: spaceId,
      userId: userId,
    );
    
    result.fold(
      onSuccess: (_) => members.removeWhere((m) => m.userId == userId),
      onFailure: (msg) => Get.snackbar('오류', msg),
    );
  }
}
```

## 공간 탈퇴

오너는 탈퇴 전에 오너 권한을 다른 멤버에게 넘겨야 한다:

```dart
Future<void> leaveSpace() async {
  final space = currentSpace;
  
  if (space.isOwner && space.members.length > 1) {
    // 다른 멤버에게 오너 권한 이전 필요
    Get.toNamed(Routes.spaceLeave, arguments: space);
    return;
  }
  
  // 마지막 멤버이거나 일반 멤버인 경우 바로 탈퇴
  final result = await _apiRepo.leaveSpace(spaceId: space.id);
  result.fold(
    onSuccess: (_) => Get.offAllNamed(Routes.mainTab),
    onFailure: (msg) => Get.snackbar('오류', msg),
  );
}
```

---

다음 편은 permission_handler로 권한 요청 UX를 개선하는 방법이다.
