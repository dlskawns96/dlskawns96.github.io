---
title: SwiftUI 데이터 바인딩 - State
excerpt: SwiftUI - State
categories:
  - iOS
permalink: /iOS/2025-03-30-SwiftUIState/
toc: true
toc_sticky: true
date: 2025-03-30
last_modified_at: 2025-03-30
---
최근에 기존 프로젝트에 컴포넌트들을 SwiftUI로 만들어서 부분적으로 도입하려는 중이다

기존 프로젝트는 RxSwift로 데이터 바인딩 처리를 했는데, SwiftUI로 만들어진 View 들은 프로퍼티 래퍼(@State, @StateObject, @ObservedObject)을 통해서 데이터 바인딩을 해보려한다

# State

<img width="668" alt="Image" src="https://github.com/user-attachments/assets/6fc0e3d5-154e-444d-8808-1b38f3419c28"/>

> SwiftUI가 관리하는 Property Wrapper

```swift
struct PlayButton: View {
  @State private var isPlaying: Bool = false // Create the state.

  var body: some View {
      Button(isPlaying ? "Pause" : "Play") { // Read the state.
          isPlaying.toggle() // Write the state.
      }
  }
}
```

isPlaying의 값이 변경될 때마다 isPlaying 의 값과 연관있는 뷰 계층을 업데이트

state는 뷰 계층의 최상위 뷰에서 private으로 선언하고, 하위 뷰에 공유

## @Binding

> 하위 뷰에 State 공유하기

```swift
struct PlayView: View {
	@State private var isPlaying = false
	
	var body: some View {
	  PlayButton(isPlaying: isPlaying)
	    .onAppear {
	      DispatchQueue.main.asyncAfter(deadline: .now() + 3) {
	        print("isPaying = \\(isPlaying)")
	      }
	    }
	}
}

struct PlayButton: View {
  @State var isPlaying: Bool
  
  var body: some View {
    Button(isPlaying ? "Pause" : "Play") {
      isPlaying.toggle()
    }
  }
}
```

위 코드처럼 하위 뷰 PlayButton에 State를 넘겨주면 하위 뷰 역시 isPlaying의 값이 바뀔 때마다 UI 상태가 업데이트 된다

하지만 하위 뷰 PlayButton에서 isPlaying의 값을 변경한다고 해서 부모뷰의 isPlaying 값이 변경되지는 않는다

```swift
struct PlayerView: View {
  @State private var isPlaying: Bool = false

  var body: some View {
      VStack {
	        // binding을 전달
          PlayButton(isPlaying: $isPlaying)
      }
  }
}

struct PlayButton: View {
  @Binding var isPlaying: Bool

  var body: some View {
      Button(isPlaying ? "Pause" : "Play") {
          isPlaying.toggle()
      }
  }
}
```

하위 뷰에서도 State의 값을 변경하기 위에서는 위처럼 State가 아니라 Binding을 전달하면 된다

## Object로 State 관리하기

```swift
@Observable
class Library {
  var name = "My library of books"
	var location = "Near"
}
```

위처럼 @Observable을 사용하면 여러개의 property를 묶어서 하나의 오브젝트로 관리할 수 있다

```swift
struct ContentView: View {
  @State private var library = Library()

  var body: some View {
		// 1.
    LibraryView(library: library)
    // 2.
    LibraryView(library: library)
	    .task {
		    library = Library()
	    }
  }
}
```

@Observable로 정의된 오브젝트를 뷰에서 사용할 때 1번 방법처럼 사용하면 ContentView가 업데이트 될 때마다 library 오브젝트가 새로 생성되기 때문에 2번 방법처럼 `task` 를 사용하여, 뷰가 처음 그려질 때만 오브젝트가 생성되도로 하는 것이 효율적이다