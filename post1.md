# 메인 스레드 블로킹 트러블슈팅

## 문제 상황

### 증상

- SwiftUI View의 `.task` modifier에서 `async` 함수 호출 시 **UI 프리징 발생**
- 로그 확인 결과, 모든 비동기 작업이 **메인 스레드에서 실행**되고 있었음
```json
⏰ [UserProfile] fetchUserData 시작 [🔴 Main]
⏰ [UserProfile] loadFromCache 완료: 0.62s [🔴 Main]
⏰ [UserProfile] saveToDatabase 완료: 1.43s [🔴 Main]
⏰ [UserProfile] fetchUserData 종료: 1.73s [🔴 Main]
```

**→ 총 1.73초 동안 메인 스레드 블로킹 → UI 멈춤**

---

## 원인 분석

### 1. SwiftUI View는 암시적으로 @MainActor
```swift
struct MyView: View {  // View 프로토콜은 @MainActor로 격리됨

    func someAsyncMethod() async {
        // 이 메서드도 암시적으로 @MainActor!
        // → 항상 메인 스레드에서 실행됨
    }
}
```

### 2. .task modifier도 @MainActor 컨텍스트
```swift
.task {
    // View가 @MainActor이므로 여기도 메인 스레드에서 시작
    await someAsyncMethod()  // 🔴 Main
}
```

### 3. Task.detached를 사용해도 해결 안 됨
```swift
.task {
    Task.detached {
        // ✅ 여기는 백그라운드에서 시작
        print(Thread.isMainThread)  // false

        // ❌ 하지만 @MainActor 메서드 호출 시 메인으로 hop!
        await someAsyncMethod()  // 🔴 Main으로 전환됨
    }
}
```

**핵심:**

Task.detached는 Task 시작 컨텍스트만 분리함. 

호출하는 함수가 @MainActor면 그 함수는 메인에서 실행됨.

### 4. extension도 마찬가지
```swift
extension MyView {
    func helperMethod() async {
        // View의 extension이므로 여전히 @MainActor
        // 🔴 Main
    }
}
```

---

## 해결 방법

### 방법 1: nonisolated 키워드 사용 ✅ (채택)
```swift
extension MyView {
    nonisolated func fetchUserData() async {
        // 🟢 Background에서 실행됨
        let result = await loadData()

        // 상태 변경은 명시적으로 메인에서
        await MainActor.run {
            self.userData = result
        }
    }
}
```

장점:
- 코드가 View 내에 유지됨
- 관련 로직이 한 곳에 모여있음
- 최소한의 변경으로 적용 가능

주의사항:
- @State 등 격리된 프로퍼티 접근 시 await MainActor.run { } 필요

### 방법 2: View 외부 함수로 분리
```swift
// View 스코프 밖에 정의 → @MainActor 아님
func loadDataInBackground(userID: String) async -> UserData {
    // 🟢 Background에서 실행
    return await networkCall(userID)
}

// View에서 사용
.task {
    let result = await loadDataInBackground(userID: currentUserID)  // 🟢 BG
    self.userData = result  // 🔴 Main (View context)
}
```

### 방법 3: UseCase로 분리 (Clean Architecture)
```swift
final class FetchUserDataUseCase {
    func execute(userID: String) async -> UserData {
        // 🟢 Background에서 실행
        // 비즈니스 로직 처리
    }
}
```

장점: 테스트 용이, 재사용성
단점: 상태 변경 로직이 섞여 있으면 분리가 복잡함

---

## 정리: 함수 위치에 따른 실행 스레드

| 위치                 | @MainActor 격리? | 실행 스레드        |
|--------------------|----------------|---------------|
| View 내부 메서드        | ✅ 암시적          | 🔴 Main       |
| View extension 메서드 | ✅ 암시적          | 🔴 Main       |
| nonisolated 메서드    | ❌              | 🟢 Background |
| View 외부 일반 함수      | ❌              | 🟢 Background |
| Actor 메서드          | 해당 Actor       | Actor에 따라     |

---

## Task.detached와 priority
```swift
Task.detached(priority: .background) {
    // priority는 "우선순위"일 뿐
    // 실행 스레드를 강제하지 않음

    await someMainActorMethod()  // 여전히 🔴 Main
}
```

priority 옵션 (.background, .low, .medium, .high, .userInitiated)은 스케줄링 우선순위를 설정하는 것이지, 실행 스레드를 결정하지 않음.

---

### 디버깅 팁: 스레드 확인 로그
```swift
func threadInfo() -> String {
    Thread.isMainThread ? "🔴 Main" : "🟢 BG(\(Thread.current.name ?? "unnamed"))"
}

print("현재 스레드: \(threadInfo())")
```

---

## 결론

1. SwiftUI View의 모든 메서드는 암시적으로 @MainActor
2. Task.detached로도 @MainActor 메서드는 메인에서 실행됨
3. 해결책: nonisolated 또는 View 외부 함수로 분리
4. 상태 변경은 await MainActor.run { } 으로 명시적 처리

---

## 참고

[View | Apple Developer Documentation](https://developer.apple.com/documentation/swiftui/view)

[Protect mutable state with Swift actors - WWDC21 - Videos - Apple Developer](https://developer.apple.com/videos/play/wwdc2021/10133/)

https://github.com/swiftlang/swift-evolution/blob/main/proposals/0316-global-actors.md
