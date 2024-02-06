# 1. 간단한 GCD / Operation 소개

GCD / Operation

- 직접적으로 쓰레드를 관리하지 않고 (쓰레드 1 생성해! 2 생성해! 이러지 않고) 큐에 작업을 분산 처리
- 그래서 GCD / Operation을 사용하면 시스템에서 알아서 쓰레드의 숫자를 관리함
- 쓰레드보다 더 높은 레벨에서 일을 한다고 보면 됨
- 오래걸리는 작업들을 비동기적으로 동작하도록 만들어줌

어떻게 대기열, 큐로 보낼까?

```swift
DispatchQueue -> Dispatch(보내다) Queue(큐에)

큐에 보낼거야 . 글로벌큐에 . 비동기적으로
DispatchQueue.global().async {
		// 여기의 작업을
}

==

let queue = DispatchQueue.global()
queue.async {}

```

DispatchQueue의 클로저는 작업 하나의 단위

```swift
DispatchQueue.global().async { 
	// 작업의 한 단위 <- task 1
}
DispatchQueue.global().async { 
	// 작업의 한 단위 <- task 2
}
```

하나의 작업이기 때문에

```swift
DispatchQueue.global().async { 
  print("1")
  print("2")
  print("3")
  print("4")
  print("5")
}

// 결과
1
2
3
4
5
```

순서대로 나온다.

중요함 이거. **클로저 = 작업하나** 이다.

---

GCD와 Operation의 차이점?

GCD

- 간단한 일
- 함수를 사용하는 작업(메서드 위주)

Operation

- 복잡한 일
- 데이터와 기능을 캡슐화한 객체

Operation은 GCD를 기반으로 여러가지 기능을 덧붙여서 나온 개념이다~

예를 들어 취소 / 순서지정 / 일시정지 (상태추적) 기능이 있다~

Operation은 클래스로 만들어진 객체이기 때문에 재사용성에 있어서 이점이 있음

근데 무조건 뭐를 써야한다! 이런건 없음 상황에 맞게 쓰는게 제일 중요함