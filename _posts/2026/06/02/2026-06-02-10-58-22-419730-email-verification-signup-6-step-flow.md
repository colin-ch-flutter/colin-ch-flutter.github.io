---
layout: post
title: "이메일 인증 회원가입 6단계 플로우 구현"
description: " "
date: 2026-06-02
tags: [Flutter, 회원가입, 이메일인증, UX, 플로우]
comments: true
share: true
---
![회원가입 플로우](https://images.unsplash.com/photo-1526378800651-2e3b8c7cb34b?w=800&q=80)

## 회원가입 플로우 설계

스마트홈 앱의 회원가입은 다음 단계로 구성된다:

```
1. 약관 동의 (SignupTermsAgreeView)
   ↓
2. 이메일 입력 (SignupEmailRegisterView)
   ↓
3. 이메일 인증 (SignupEmailVerifyView)
   ↓
4. 닉네임 설정 (SignupNicknameView)
   ↓
5. 완료 (SignupCompleteView)
```

각 단계를 별도 화면으로 분리한 이유는 UX 때문이다. 긴 폼 하나보다 단계별로 나누면 사용자가 덜 부담스럽게 느낀다.

## 약관 동의 화면

서비스 이용약관, 개인정보 처리방침, 마케팅 동의 등을 처리한다. 필수와 선택을 구분해야 한다:

```dart
class SignupTermsAgreeController extends GetxController {
  final terms = [
    TermsItem(
      id: 'service',
      title: '서비스 이용약관',
      required: true,
      url: 'https://...',
    ),
    TermsItem(
      id: 'privacy',
      title: '개인정보 처리방침',
      required: true,
      url: 'https://...',
    ),
    TermsItem(
      id: 'age',
      title: '만 14세 이상 확인',
      required: true,
    ),
    TermsItem(
      id: 'marketing',
      title: '마케팅 정보 수신 동의 (선택)',
      required: false,
    ),
  ];

  final RxMap<String, bool> agreed = <String, bool>{}.obs;

  bool get canProceed => terms
      .where((t) => t.required)
      .every((t) => agreed[t.id] == true);

  void toggleAll(bool value) {
    for (final term in terms) {
      agreed[term.id] = value;
    }
  }

  void toggleTerm(String id, bool value) {
    agreed[id] = value;
  }

  void proceed() {
    if (!canProceed) return;
    Get.toNamed(Routes.signupEmailRegister, arguments: {
      'agreedMarketing': agreed['marketing'] ?? false,
    });
  }
}
```

## 이메일 입력 및 인증 코드 발송

```dart
class SignupEmailRegisterController extends GetxController {
  final TextEditingController emailController = TextEditingController();
  final RxBool isLoading = false.obs;
  final RxString error = ''.obs;

  bool get isValidEmail {
    final email = emailController.text.trim();
    return RegExp(r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$')
        .hasMatch(email);
  }

  Future<void> sendVerificationEmail() async {
    if (!isValidEmail) {
      error.value = '올바른 이메일 주소를 입력해주세요';
      return;
    }

    isLoading.value = true;
    error.value = '';

    final result = await _apiRepo.sendEmailVerification(
      emailController.text.trim(),
    );

    isLoading.value = false;

    result.fold(
      onSuccess: (_) => Get.toNamed(
        Routes.signupEmailVerify,
        arguments: {'email': emailController.text.trim()},
      ),
      onFailure: (msg) => error.value = msg,
    );
  }
}
```

## 이메일 인증 코드 확인

6자리 코드를 입력하고 검증한다. 타이머로 재발송 쿨다운을 구현한다:

```dart
class SignupEmailVerifyController extends GetxController {
  final String email;
  final TextEditingController codeController = TextEditingController();
  
  final RxInt resendCooldown = 0.obs;
  Timer? _timer;

  SignupEmailVerifyController({required this.email});

  @override
  void onInit() {
    super.onInit();
    _startCooldown();
    
    // 6자리 입력 완료 시 자동 확인
    codeController.addListener(() {
      if (codeController.text.length == 6) {
        verifyCode();
      }
    });
  }

  void _startCooldown() {
    resendCooldown.value = 60;
    _timer = Timer.periodic(const Duration(seconds: 1), (timer) {
      if (resendCooldown.value > 0) {
        resendCooldown.value--;
      } else {
        timer.cancel();
      }
    });
  }

  Future<void> resendCode() async {
    if (resendCooldown.value > 0) return;
    await _apiRepo.sendEmailVerification(email);
    _startCooldown();
  }

  Future<void> verifyCode() async {
    final code = codeController.text.trim();
    if (code.length != 6) return;

    final result = await _apiRepo.verifyEmailCode(email: email, code: code);
    result.fold(
      onSuccess: (token) => Get.toNamed(
        Routes.signupNickname,
        arguments: {'verificationToken': token},
      ),
      onFailure: (_) {
        codeController.clear();
        Get.snackbar('오류', '인증 코드가 올바르지 않습니다');
      },
    );
  }

  @override
  void onClose() {
    _timer?.cancel();
    super.onClose();
  }
}
```

## 타이머 UI

```dart
Obx(() {
  final cooldown = controller.resendCooldown.value;
  return TextButton(
    onPressed: cooldown > 0 ? null : controller.resendCode,
    child: Text(
      cooldown > 0 ? '재발송 (${cooldown}초)' : '인증 코드 재발송',
    ),
  );
})
```

## 13세 이상 동의 처리

일부 국가에서는 미성년자 관련 개인정보 규정이 있다. 가입 시 만 14세 이상임을 확인하는 동의를 받고, 이 정보를 서버에 전달한다:

```dart
// 회원가입 API 요청에 포함
class SignupRequest {
  final String email;
  final String password;
  final String nickname;
  final bool agreedMarketing;
  final bool agreedAge;  // 만 14세 이상 동의

  const SignupRequest({
    required this.email,
    required this.password,
    required this.nickname,
    required this.agreedMarketing,
    required this.agreedAge,
  });
}
```

---

다음 편은 공간과 멤버 관리 기능이다.
