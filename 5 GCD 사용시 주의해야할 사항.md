# 5. GCD 사용시 주의해야할 사항

### 반드시 메인큐에서 처리해야 하는 작업

UI 관련 일들은 “메인큐”에서 처리해야 한다.

화면을 표시하는 일은 한개의 스레드에서만 담당해야한다. 당연한 말이지만 스레드 여러개에서 UI작업을 하게되면 간섭이 일어날 가능성이 높음. 원하는 동작이 되지 않을 수 있음!

그래서 아무리 글로벌 큐에서 작업을 수행하고 있어도 UI 작업에 한해서는 반드시 DispatchQueue.main 큐를 호출해서 비동기적으로 처리해야 한다는 것.

### sync 메서드에 대한 주의사항

1. 메인큐에서는 다른큐로 보낼때 sync메서드를 부르면 절대 안된다. 

메인큐는 화면을 담당하는 영역이기 때문에 sync를 호출하면 화면이 버벅이게 된다. 사용성 똥망됨.

메인큐에서 DispatchQueue.global.sync{ .. } 를 호출하게 되면 현재 컨텍스트인 메인스레드가 블로킹되고 global.sync로 호출한 작업이 리턴되기 까지 아무것도 할 수 없기 때문에 UI반응이 멈출 수 밖에 없다.

1. 현재 큐에서 현재 큐로 “동기적”으로 보내면 안된다.

데드락 걸림. 

```swift
DispatchQueue.global().async {  <- task A 시작
	DispatchQueue.global().sync {

  }
} <- task A 종료
```

이때 발생하는 데드락 시나리오.

이때 task1이 시작하는 스레드가 2라고 가정.

1. async 테스크 A 가 글로벌 큐로 보내지고 스레드 2로 보내짐
2. A는 스레드 2에서 비동기적으로 실행됨
3. 다시 A가 sync로써 글로벌 큐로 보내짐
4. A는 sync 이므로 현재 컨텍스트인 스레드 2는 A가 리턴될 때 까지 블로킹 됨.
5. 3에서 글로벌 큐로 보내진 A는 스레드 2로 접근함.
6. 블로킹된 스레드 2는 멈춰진 상태 **[데드락]**

하지만 만약 테스크 A가 글로벌 큐에 sync로 보내지고 다시 스레드를 할당받을 때 원래 컨텍스트였던 스레드 2가 아닌 다른 스레드도 보내진다면 데드락이 발생하지 않는다.

이미 데드락이 걸릴 가능성을 내포하고 있기때문에 반드시 발생하지 않는다고 하더라도 그 가능성만으로도 절대로 해서는 안되는 구현방식이다.

더더군다나 동시큐가 아니라 직렬큐라면? 100% 확률로 데드락 걸린다. 

### weak, strong 캡쳐 주의

큐의 async든 sync든 클로저 형태로 사용되기 때문에 객체를 캡쳐하게 된다. 반드시 weak self를 사용하자.

### completion handler의 존재이유

비동기 작업이 명확하게 끝나는 시점을 활용할 필요가 있기 때문.

모든 비동기 함수와 관련된 작업들이 모두 컴플리션 핸들러를 가지고 있는 이유가 이것임

### 동기적 함수를 비동기 함수처럼 만드는 방법

왜 이런짓을 하느냐? 재활용을 위해!

시간이 오래걸리는 동기함수가 있다고 해봅시다.

```swift
public func tiltShift(image: UIImage?) -> UIImage? {
	guard let image else { return }
  sleep(1)
	let mask = topAndBottomGradient(size: image.size)
	return image.applyBlur(radius: 6, maskImage: mask)
}
```

tiltShift는 오래 걸리는 동기함수. 

이걸 비동기 함수로 바꿔보면 아래와 같다.

```swift
func asyncTiltShift(
  _ image: UIImage?,
 runQueue: DispatchQueue, // 실행할 큐
 completionQueue: DispatchQueue, // 작업이 끝났을 때 컴플리션 핸들러를 수행할 큐
 completion: @escaping Result<UIImage?, Error> -> ()
) { 
  runQueue.async {
	  let tiltedImage = tiltShift(image: image)

    completionQueue.async { 
			completion(.success(tiltedImage))
		}
  }
}
```

이미지 처리를 백그라운드에서 수행하고, 처리가 끝난 후 UI를 업데이트해야 한다면:

```swift
asyncTiltShift(
  image,
  runQueue: DispatchQueue.global(qos: .userInitiated),
  completionQueue: DispatchQueue.main,
  completion: { result in
    // UI 업데이트
  }
)
```

이미지 처리와 결과 처리 모두 백그라운드에서 수행하고, UI 업데이트가 필요 없다면:

```swift
asyncTiltShift(
  image,
  runQueue: DispatchQueue.global(qos: .utility),
  completionQueue: DispatchQueue.global(qos: .default),
  completion: { result in
    // 백그라운드에서 결과 처리
  }
)
```

---

## weak self 관련

컴플리션 핸들러에 꼭 weak self를 써서 ARC 카운팅을 방지해야 하는가? 에 대한 정리

결론부터 말하자면 꼭 써야한다.

아래 예시를 보자.

먼저 ARC 카운팅 테스트를 위한 클래스를 만들었다. 이름은 `TestedVC`.

```swift
import UIKit

class TestedVC: UIViewController {
  
  var name: String = "VC"
  
  func doSomething() {
    DispatchQueue.global().async {
      sleep(3)
      print("글로벌 큐에서 출력하기: \(self.name)")
      DispatchQueue.main.async {
        print("메인 큐에서 출력하기: \(self.name)")
      }
    }
  }
  
  deinit {
    print("\(name) 메모리 해제")
  }
}
```

`TestedVC`를 호출할 `TestingVC` 코드:

```swift
import UIKit

class TestingVC: UIViewController {
  
  var testedVC: TestedVC? = TestedVC()
  
  override func viewDidLoad() {
    super.viewDidLoad()
    self.localScopeFunction()
  }
  
  func localScopeFunction() {
    self.testedVC?.doSomething()
    self.testedVC = nil // 현실에서 이런 코드는 없을것이고 대신 TestedVC를 dismiss 혹은 pop이 되는 상황이라고 가정함으로서 nil을 할당
  }
}
```

`TestingVC`를 실행할 때 Heap과 Stack영역에서 벌어지는 일들을 차례대로 짚어보자.

![5-1.png](https://github.com/naldal/ConcurrencyNote/blob/main/Assets/5-1.png?raw=true.png)

`TestingVC`가 메모리에 올라갈 때, Stack에는 함수(`viewDidLoad()`, `localScopeFunction()`)와 로컬변수, 인스턴스의 복사본(`let testedVC`) 이 올라간다.

Heap에는 객체 인스턴스(`TestedVC`) 가 올라간다 

`TestingVC` 는  `TestedVC` 인스턴스를 가지고 참조하고 있으므로 이때 ARC 카운트는 +1 이 된다.

![5-2.png](https://github.com/naldal/ConcurrencyNote/blob/main/Assets/5-2.png?raw=true.png)

이제 TestedVC에서 사용하고 있는 컴플리션 핸들러들이 어떻게 메모리를 점유하고 있는지 보자.

`doSomgthing()` 메서드는 `DispatchQueue.global.async` 와 `DispatchQueue.main.async`, 두 개의 컴플리션 핸들러를 가지고 있다.

클로저는 Heap 메모리에 저장된다. (컴파일러가 크기를 미리 알 수 없는 것들이 힙에 저장됨. 그래서 객체 인스턴스도 힙에 저장되는 것.)

![5-3.png](https://github.com/naldal/ConcurrencyNote/blob/main/Assets/5-3.png?raw=true.png)

이 핸들러 안에서 `TestedVC`의 name 멤버변수를 참조하고 있다.

따라서 `TestedVC`의 ARC 카운트는 이 시점에서 +2가 되어 최종적으로 3이다.

![5-4.png](https://github.com/naldal/ConcurrencyNote/blob/main/Assets/5-4.png?raw=true.png)

`TestedVC`가 메모리에서 정상적으로 잘 해제가 된 것이라면 `var testedVC`에 nil을 할당하는 그 시점에서 로그에 “뷰 컨트롤러 메모리 해제” 가 나오고, 그 다음에 GCD 컴플리션 핸들러 안에 print 되는 “글로벌 큐에서 출력하기: ~” “메인 큐에서 출력하기: ~” 로그가 출력되어야 한다. 

이제 돌려보면 예상과 다르게 로그가 찍힌다.

![5-5.png](https://github.com/naldal/ConcurrencyNote/blob/main/Assets/5-5.png?raw=true.png)

GCD 컴플리션 핸들러가 모두 완료되고 난 후에야 `TestedVC`가 deinit 된 것이다.

왜 이런 결과가 발생했을까?

![5-6.png](https://github.com/naldal/ConcurrencyNote/blob/main/Assets/5-6.png?raw=true.png)

이렇게 ARC가 카운팅 된 상황에서

```swift
func localScopeFunction() {
  self.testedVC?.doSomething()
  self.testedVC = nil // 여기!
}
```

`testedVC`에 nil 이 할당됐을 때 ARC 카운트는 1이 줄어 2가 되었다.

![5-7.png](https://github.com/naldal/ConcurrencyNote/blob/main/Assets/5-7.png?raw=true.png)

그리고 나면 TestedVC는 앱이 종료될 때 까지 메모리에 저장되어 있을 것이다. 

이것을 메모리 릭(Memory leak) 이라고 한다.

그렇다면 이런 상황을 방지하려면? 

TestedVC 캡쳐를 약한 참조로 해야한다. 

weak self는 객체가 캡쳐될 때 ARC 카운팅이 올라가지 않게 하는 역할을 한다.

GCD 컴플리션 핸들러에서 weak self를 써주면 그 결과가 아래와 같이 나온다.

```swift
func doSomething() {
  DispatchQueue.global().async { [weak self] in
    sleep(3)
    print("글로벌 큐에서 출력하기: \(self?.name)")
    DispatchQueue.main.async {
      print("메인 큐에서 출력하기: \(self?.name)")
    }
  }
}
```

![5-8.png](https://github.com/naldal/ConcurrencyNote/blob/main/Assets/5-8.png?raw=true.png)

`TestedVC` 인스턴스를 약하게 참조함으로써 ARC카운트가 올라가지 않아, nil이 할당될 때 정상적으로 메모리에서 해제되고 이에 따라서 멤버변수 name 역시도 nil 로 처리된다는 사실.

이제 TestedVC는 메모리에서 정상적으로 해제됐다!

추가적으로 `guard let self = self else { return }` 을 통해 아예 컴플리션 핸들러 자체를 수행하지 않고 클로저를 early exit 시킬 수 있다.