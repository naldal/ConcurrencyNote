# 2. 동기 vs 비동기

### 글로벌 큐 - 동기 작동방식

```swift
DispatchQueue.global().sync { 

}
```

원래의 작업이 진행되고 있던 곳(메인스레드)에서

디스패치 글로벌 큐로 보낸 작업을 동기적으로 기다린다.

근데 글로벌 큐에서 다른 스레드로 ‘동기’적으로 보내는게 의미가 없다.

메인스레드에서 일하는거랑 똑같음. 결국 작업이 리턴될 때 까지 기다리게 될테니까.

그래서! 동기적으로 보내는 코드를 짜더라도 OS에서 실질적으로는 메인스레드에서 일한다.

```swift
func checkThread() {
  DispatchQueue.global().sync { [weak self] in
    usleep(500_000)
    print(Thread.current)
  }
}
```

![2-1.png](https://github.com/naldal/ConcurrencyNote/blob/main/Assets/2-1.png?raw=true.png)

### 정리

Async

- 작업을 다른 스레드에서 하도록 시킨 후, 그 작업이 끝나길 “안 기다리고” 다음일을 진행한다 (안 기다려도 다음작업을 생성할 수 있다)

Sync

- 작업을 다른 스레드에서 하도록 시킨 후 그 작업이 끝나길 “기다렸다가” 다음일을 진행한다. (기다렸다가 다음 작업을 생성할 수 있다)

---

근데 비동기 왜 필요함? → 대부분 서버와의 통신 때문.

리스폰이 얼마나 걸릴지 모르니까! 응답 올때까지 기다리고 앉아있을 순 없잖아요 🤢

그래서 네트워크 관련 작업들은 **내부적으로** 비동기적으로 구현되어 있다.

```swift
URLSession(configuration: .ephemeral).dataTask(with: url) { data, response, error in 
	self.image = UIImage(data: data!)
	...
}
```

이미 컴플리션 핸들러 안에서는 다른 스레드에서 일을 하고 있다.