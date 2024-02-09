# 4. GCD  종류와 특성

1. GCD ← 디스패치 큐
그리고 디스패치 큐에는 세 가지 종류가 있음
    1. Global 메인 큐
    2. Global
    3. Private (커스텀)
2. Operation ← 오퍼레이션 큐

대기열마다 특성이 다르기 때문에 원하는 일처리에 따라 거기에 맞는 대기열로 보내면 된다.

그럼 각자의 큐가 각자 다른 스레드를 생성해서 일을 하게된다!

---

## 메인 큐

유일한 큐이자 직렬로 동작함 = 메인스레드를 의미

:: 1번 스레드는 사실 메인 스레드이자 메인 큐 이기도 함 (유일)

평소에 쓰던  쓰는 코드는 사실 Main.sync 에서 일을 하고 있던 것이었다 (ㄷㄷ)

```swift
print("print something")
```

이게 사실은

```swift
DispatchQueue.main.sync { <-- 사실은 이런 의미가 숨어있었다;
	print("print something") <- 1번 스레드에서 일함
}
```

물론 main.sync를 직접적으로 쓰면 데드락 걸려서 런타임 에러지만, 일반적인 코드를 작성할 때 GCD를 명시적으로 사용하지 않는다면 작업이 처리되는 성질은 메인스레드에서 동기적으로 돌아간다는 말이다.

예를 들어

```swift
override func viewDidLoad() {
	super.viewDidLoad()
  function1() // 버튼의 색상을 바꾸는 작업
  function2() // 1부터 100까지 프린트를 찍는 작업
  function3() // 서버와 네트워크를 처리하는 작업
}
```

이런 코드가 있다고 해보자.

나는 버튼의 색상을 바꾸는 작업은 당연히 async적으로 수행되어야 한다고 생각했다. 

그래서 따로 명시하지 않아도 function1()은 자동으로 DispatchQueue.main.async안에서 돌아간다고 생각했다. function3()도 당연히 자동으로 DispatchQueue.global() 안에서 돌아갈 것이라고 생각했다. 

하지만 완전 틀려먹은 생각!

function1(), function2(), function3()은 모두 **메인스레드에서 동기적으로 돌아간다!**

## 글로벌큐와 QoS의 이해

글로벌큐에는 중요도가 있다. 기본은 `concurrent` 

중요도 순으로

- userInteractive → 거의 즉시
    - 유저의 직접적인 인터렉티브, 애니메이션, UI반응 관련된 어느것이든)
- userInitiated → 몇 초
    - 유저가 즉시 필요하긴 하지만 비동기적으로 처리된 작업(앱 내에서 pdf파일을 여는것 같은)
- 디폴트(concurrent)
    - 일반적인 작업
- utility  → 몇 초에서 몇 분
    - 보통 Progress Indicator와 함께 길게 실행되는 작업, 계산, IO, Networking, 지속적인 데이터 feeds
- background → 속도보다는 에너지 효율성 중시, 몇 분 이상
    - 시간이 별로 안 중요한 작업이나 데이터 미리 가져오기, 데이터베이스 유지
- unspecified
    - 레거시 API에 사용한다고 함(스레드를 서비스 품질에서 제외시키는)

만약, 작업1과 작업2가 메인스레드에서 처리되고 있다가

작업1이 글로벌큐의 디폴트, 작업2가 글로벌큐의 userInitiated로 보내졌다면 어떻게 될까?

```swift
DispatchQueue.global().async { 
	task1()
}

DispatchQueue.global(qos: .userInitiated).async { 
	task2()
}
```

그러면 iOS에서 알아서 우선적으로 userInitiated에 더 많은 스레드를 배치하고 배터리를 더 집중해서 사용하도록 함. 

만약, 큐는 background인데, 작업을 보낼 때 async 파라미터를 더 높은 utility로 실행한다면?

```swift
let queue = DispatchQueue.global(qos .background)
queue.async(qos: .utility) {

}
```

큐가 작업의 영향을 받아 큐의 품질도 utility로 상승한다고 함

## 프라이빗(커스텀) 큐

디폴트는 직렬큐임. 원하면 동시큐로도 만들 수 있고 QoS 설정도 가능.

```swift
let queue = DispatchQueue(label: "com.note.serial")

let queue2 = DispatchQueue(label: "com.note.concurrent", attributes: .concurrent)
```

결론적으로는 작업에 대해서 서비스 품질을 설정해주지 않아도 OS에서 추론을 알아서 해준다.

기본적으로 직렬큐이기 때문에 비동기적으로 테스크들을 호출한다고 해도 순서는 순차적이다.

아래처럼 호출한다고 해보자

```swift
let privateQueue = DispatchQueue(label: "com.note.serial")

privateQueue.async {
  print("task1 시작")
  usleep(200_000)
  print("task1 종료")
}

privateQueue.async {
  print("task2 시작")
  print("task2 종료")
}

privateQueue.async {
  print("task3 시작")
  usleep(500_000)
  print("task3 종료")
}
```

그럼 결과는?

```swift
task1 시작
task1 종료
task2 시작
task2 종료
task3 시작
task3 종료
```

여기서 나는 몇 가지 깨달음을 얻었다.

나는 여태껏 직렬큐에서 async로 호출한 결과들의 호출순서는 순차적일지 몰라도 종료순서는 순차적이지 않다! 라고 생각했다. 

그래서 나는 위 코드의 결과가 아래처럼 나올 줄 알았다.. 비동기?니까 😪

```swift
task1 시작
task2 시작
task3 시작
task2 종료
task1 종료
task3 종료
```

하지만 !큐! Queue!!! 는 FIFO 다. 그리고 직렬큐는 한번에 하나의 작업만 수행된다. (스레드를 하나만 쓸수도 있고 여러개 쓸수도 있지만 이건 OS영역 .하지만 여기서는 스레드 2만 쓴다고 가정해보자)

그렇다면 테스크들이 큐에 쌓일때 ‘직렬적으로’ 쌓이게 될거다.

![4-1.png](https://github.com/naldal/ConcurrencyNote/blob/main/Assets/4-1.png?raw=true.png)

하나의 스레드만 사용하므로 해당 스레드로 작업들이 하나씩 이동하게 될테고 결국 아래 그림처럼 된다.

![4-2.png](https://github.com/naldal/ConcurrencyNote/blob/main/Assets/4-2.png?raw=true.png)

여기가 바로 헷갈리는 지점인데, async인 task1이 시작되고나서 종료될때까지 스레드가 기다릴까?

안기다린다. async니까. 근데 이 안기다린다는게 바로 task2를 실행한다는말이 아니다!!! task2는 task1이 종료된 후에야 실행된다. 

그럼 안 기다린다는게 뭐냐고? 안 기다린다는 말은 “안 중지한다” 라는 말과 같다. 

다시 말해 async 작업은 스레드를 중지하지 않고 task2를 task1 뒷 순서에 배치하는 작업을 수행하고, task3을 task2 뒷 순서에 배치하는 작업을 안 기다리고 수행한다는 말이다.

여기에서 sync와 async의 차이가 두드러진다. sync는 스레드를 “중지”시켜버린다. 

위 예시코드를 sync로 바꿔보자.

```swift
let privateQueue = DispatchQueue(label: "com.note.serial")

privateQueue.sync {
  print("task1 시작")
  usleep(200_000)
  print("task1 종료")
}

privateQueue.sync {
  print("task2 시작")
  print("task2 종료")
}

privateQueue.sync {
  print("task3 시작")
  usleep(500_000)
  print("task3 종료")
}
```

이것의 결과는 async로 했을때의 결과와 같다.

하지만 동작의 차이는 꽤 크다.

이렇게 하면 스레드는 task1이 실행중일때 중지 = 블록(block)된다. task1이 완료될때까지 대기한다는 말과 같다. 결국 task2는 task1이 종료되어 스레드에 제어권이 넘어올때에, 그제서야 스레드에서 실행된다는 것이다.