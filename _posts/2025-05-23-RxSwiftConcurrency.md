---
title: RxSwift에서 Swift 6 Concurrency 대응하기
excerpt: RxSwift에서 Swift 6 Concurrency 대응하기
categories:
  - iOS
permalink: /iOS/2025-05-23-RxSwiftConcurrency/
toc: true
toc_sticky: true
date: 2025-05-23
last_modified_at: 2025-05-23
---
우리 프로젝트에서는 이벤트 흐름을 RxSwift로 관리하고 있고,

버튼 클릭 -> 새로운 ViewController Push 의 흐름을 아래처럼 관리하고 있다
1. 버튼 클릭 이벤트 발생
2. ViewModel의 input으로 전달
3. input이 들어오면 ViewController를 만들어서 output에 전달
4. ViewController에서 output을 구독하고 있다가 전달된 ViewController push

ViewController는 어떤 ViewController를 만들어서 push 할지 알 필요 없이 전달되는 ViewController를 열기만 하면 돼서 코드가 간결해지는 장점이 있다

```swift
@MainActor
private func makeViewController() -> UIViewController {
	return UIViewController()
}

// MARK: - ViewModel
input.openButtonDidTap
	.map { [weak self] in
		return self?.makeViewController()
	}
	.bind(to: self.output.pushViewController)
	.disposed(by: self.disposedBag)

// MARK: - ViewController
output.pushViewController
	.observe(on: MainScheduler.asyncInstance)
	.bind { [weak self] in
		guard let self else { return }
		self.navigationController?.pushViewController($0, animated: true)
	}
	.disposed(by: self.disposedBag)
```

이 흐름을 코드로 나타내면 위와 같은데, 기존에는 `input.openButtonDidTap.map { }` 안에서 MainActor 함수를 호출해도 문제 없었으나 프로젝트의 Swift 버전을 6.0으로 올리면서 이런 에러가 발생한다
![](https://i.imgur.com/fDpIxh7.png)
대략 MainActor 함수인 `makeViewController()`가 메인 엑터 컨텍스트가 아닌 곳에서 호출되는게 문제라는 얘기다

이 에러를 해결 하기 위해서 ViewModel의 코드를 아래처럼 수정해보려고 했다
```swift
input.openButtonDidTap
	.bind { [weak self] in
		Task { @MainActor in
			self?.output.pushViewController.accept(
				self?.makeViewController()
			)
		}
	}
	.disposed(by: self.disposedBag)
```

이렇게 수정하면 `makeViewController()`가 메인 액터 컨텍스트에서 호출되어서 에러가 해결되지만 bind 안에서 새로운 Task 가 생성되는 것도 그렇고, 그냥 보기에도 코드가 깔끔해 보이지 않는다...

이런 문제들을 대응하기 위해서 RxSwift 6.5.0 에서 [Swift Concurrency 대응](https://github.com/ReactiveX/RxSwift/blob/main/Documentation/SwiftConcurrency.md)이 추가되었다

```swift
let someEvents = PublishRelay<Void>()

// 1. 기존
someEvents
	.bind { [weak self] in
		self?.doSomething()
	}
	.disposed(by: self.disposeBag)

// 2. 신규
Task { [weak self] in
	for try await event in someEvents.values {
		self?.doSomething()
	}
}
```
이런 식으로 RxSwift 스럽게 스트림을 구독하는 코드를 `AsyncStream`인 .values에 접근하여 await을 사용하여 처리할 수 있도록 되었다

```swift
Task { [weak self] in
	for try await _ in input.imageSearchButtonDidTap.values {
		self?.output.pushViewController.accept(self?.makeViewController())
	}
}
```
이걸 이용해서 위의 에러를 수정하면 위처럼 Swift Concurrency 스럽게 처리할 수 있게 되었다

.subscribe 가 아니라 Task { } 을 사용하면서 귀찮은 [weak self]를 안 붙여도 될까 기대했는데 실제로 AsyncStream을 구독하는 경우에는 [weak self] 를 명시하지 않으면 메모리가 해제되지 않는 걸 알 수 있다
이에 대한 자세한 내용은 [# Swift — Task에서 weak self는 언제 쓸까요?](https://medium.com/@sunghyun_kim/swift-task%EC%97%90%EC%84%9C-weak-self%EB%8A%94-%EC%96%B8%EC%A0%9C-%EC%93%B8%EA%B9%8C%EC%9A%94-f7ccf78e22bb) 참조

이 방식 덕분에 Swift Concurrency 환경에서도 기존 RxSwift 흐름을 깔끔하게 유지할 수 있었다
앞으로도 Concurrency와 RxSwift의 공존 방식에 대해 좀 더 고민해보면서, 점점 이식해 나갈 예정이다