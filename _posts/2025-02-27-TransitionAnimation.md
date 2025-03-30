---
title: Swift CABasicAnimation으로 움직이는 Masking + Gradient Border 애니메이션 구현하기
excerpt: Swift CALayer 애니메이션
categories:
  - iOS
permalink: /iOS/2025-02-27-TransitionAnimation/
toc: true
toc_sticky: true
date: 2025-02-27
last_modified_at: 2025-02-27
---
오늘은 진행했던 프로젝트의 애니메이션 요구사항을 어떻게 구현했는지 기록해보려고 한다.

요구사항을 간략히 설명하면 다음과 같다:
```
1. 서버에서 1~3개의 사각형 좌표(CGRect)가 전달된다 
2. 각각의 CGRect는 화면의 특정 부분을 나타낸다
3. 화면은 해당 CGRect 중 하나의 CGRect 영역만 밝게 나머지는 어둡게 표시되어야 한다
4. 밝은 영역의 테두리에는 그라데이션 컬러와 Radius 16이 적용되어야 한다
5. 사용자의 조작으로 다른 CGRect 값으로 변경될 경우, 마스킹 영역과 테두리가 애니메이션을 통해 부드럽게 이동되어야 한다
```

이미 프로젝트 내에 그라데이션 컬러 border 를 적용하는 함수와, 마스킹 처리를 하는 함수가 있었기 때문에 레이어의 좌표가 변경되는 애니메이션만 추가하면 간단하게 처리할 수 있을 것이라고 생각했지만 여러 시행착오를 겪었다.

구현하면서 까다로웠던 포인트는 다음과 같다:
1. 그라데이션 컬러 border와 마스킹 모두 CALayer로 처리되기 때문에, UIView 애니메이션 사용 불가
2. 정확하게 동일한 타이밍으로 border와 마스킹을 원하는 좌표로 이동 
3. 애니메이션 중 그라데이션 컬러 border 의 radius가 유지안되는 문제

첫번째 문제는 애니메이션을 CABasicAnimation 으로 처리 해주면 해결되는 문제였다.

> 정확하게 동일한 타이밍으로 border와 마스킹을 원하는 좌표로 이동

두번째 문제를 해결한 방법은 다음과 같다

```Swift
let commonBeginTime = animationLayer.convertTime(CACurrentMediaTime(), from: nil)

let animation1 = CABasicAnimation()
animation1.fromValue = fromValue
animation1.toValue = toValue
animation1.beginTime = commonBeginTime

let animation2 = CABasicAnimation()
animation2.fromValue = fromValue
animation2.toValue = toValue
animation2.beginTime = commonBeginTime

animationLayer.add(animation1, forKey: "animation1")
animationLayer.add(animation2, forKey: "animation2")
```
위처럼 동일한 beginTime을 설정함으로써 같은 타이밍에 애니메이션이 시작하는 것을 보장할 수 있었다

> 애니메이션 중 그라데이션 컬러 border 의 radius가 유지안되는 문제

세번째 문제는 검색해봐도 예제가 거의 보이지 않고 원인을 파악하기 어려워서 가장 해결하기 어려웠다

우선 border animation은 다음과 같이 구현했다:
```Swift
// CABasicAnimation 은 frame 의 변경을 지원하지 않기 때문에
// position, bounds 애니메이션을 각각 만들고 group 화
if let _ = self.layer.sublayers?.first(where: { $0.name == "GradientBorder" }) {
	let positionAnimation = CABasicAnimation(keyPath: "position")
	positionAnimation.fromValue = { 현재 포지션의 좌표 }
	positionAnimation.toValue = { 이동할 포지션의 좌표 }

	let boundsAnimation = CABasicAnimation(keyPath: "bounds")
	boundsAnimation.fromValue = { 현재 bounds }
	boundsAnimation.toValue = { 변경할 좌표 }

	let animationGroup = CAAnimationGroup()
	animationGroup.animations = [positionAnimation, boundsAnimation]
	animationGroup.duration = 0.15
	animationGroup.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)
	animationGroup.beginTime = commonBeginTime
	
	self.gradient.removeAllAnimations()
	self.gradient.add(animationGroup, forKey: "frameAndPositionAnimation")
	self.gradient.frame = rect
} else {
	... 초기세팅
}
```

이 방식으로 구현했을 때는 선택된 영역을 나타내는 Rectangle의 radius가 제대로 적용이 안되는 문제가 있었다
이 문제에 대해 검색 해보다가 [StackOverflow](https://stackoverflow.com/questions/36459532/uiview-layer-mask-animate)의 글에서 rounded rect 를 가진 사각형을 애니메이션할 때 발생하는 버그라는 것을 발견했다 

> "One thing to note with this is that there appears to be a bug in Core Animation, where it's unable to correctly animate from a path that includes rectangle, to a path that includes a rounded rect with a corner radius. I have filed a bug report with Apple for this." 

이걸 해결하기 위해서 위의 StackOverflow 글에서 제시한 방법과 다르게 아래처럼 해결한다

```Swift
private func addGradientBorderWithAnimation(
to rect: CGRect,
cornerRadius: CGFloat,
colors: [UIColor],
commonBeginTime: CFTimeInterval
) {
	if let _ = self.layer.sublayers?.first(where: { $0.name == "GradientBorder" }) {
		let pathAnimation = CABasicAnimation(keyPath: "path")
		pathAnimation.fromValue = self.createRoundedRectPath(rect: self.shape.path!.boundingBox, cornerRadius: cornerRadius)
		pathAnimation.toValue = self.createRoundedRectPath(rect: CGRect(x: 0, y: 0, width: rect.width, height: rect.height), cornerRadius: cornerRadius)
		pathAnimation.duration = 0.15
		pathAnimation.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)
		pathAnimation.beginTime = commonBeginTime
		pathAnimation.isRemovedOnCompletion = true
		self.shape.removeAllAnimations()
		self.shape.add(pathAnimation, forKey: "path")

		let positionAnimation = CABasicAnimation(keyPath: "position")
		positionAnimation.fromValue = self.gradient.presentation()?.position
		positionAnimation.toValue = CGPoint(x: rect.midX, y: rect.midY)

		let boundsAnimation = CABasicAnimation(keyPath: "bounds")
		boundsAnimation.fromValue = self.gradient.presentation()?.bounds
		boundsAnimation.toValue = CGRect(x: 0, y: 0, width: rect.width, height: rect.height)

		let animationGroup = CAAnimationGroup()
		animationGroup.animations = [positionAnimation, boundsAnimation]
		animationGroup.duration = 0.15
		animationGroup.timingFunction = CAMediaTimingFunction(name: .easeInEaseOut)
		animationGroup.beginTime = commonBeginTime
		self.gradient.removeAllAnimations()
		self.gradient.add(animationGroup, forKey: "frameAndPositionAnimation")

		self.gradient.mask = self.shape
		self.gradient.frame = rect
		self.shape.path = UIBezierPath(roundedRect: CGRect(x: 0, y: 0, width: rect.width, height: rect.height), cornerRadius: cornerRadius).cgPath

	} else {
		... 초기세팅
	}
}
```

코드가 정리가 안되어 좀 복잡해서 대략적으로 설명하자면
1. 영역을 표시할 사각형을 UIBezierPath를 이용해 그리는게 아니라 아래의 함수로 사각형을 직접 그려줌
```Swift
private func createRoundedRectPath(rect: CGRect, cornerRadius: CGFloat) -> CGPath {
	let path = UIBezierPath()
	let minRadius = min(rect.width, rect.height) / 2
	let radius = min(cornerRadius, minRadius)
	
	// Top-left corner
	path.move(to: CGPoint(x: rect.minX + radius, y: rect.minY))
	path.addLine(to: CGPoint(x: rect.maxX - radius, y: rect.minY))
	path.addQuadCurve(to: CGPoint(x: rect.maxX, y: rect.minY + radius),
					  controlPoint: CGPoint(x: rect.maxX, y: rect.minY))
	
	// Top-right corner
	path.addLine(to: CGPoint(x: rect.maxX, y: rect.maxY - radius))
	path.addQuadCurve(to: CGPoint(x: rect.maxX - radius, y: rect.maxY),
					  controlPoint: CGPoint(x: rect.maxX, y: rect.maxY))
	
	// Bottom-right corner
	path.addLine(to: CGPoint(x: rect.minX + radius, y: rect.maxY))
	path.addQuadCurve(to: CGPoint(x: rect.minX, y: rect.maxY - radius),
					  controlPoint: CGPoint(x: rect.minX, y: rect.maxY))
	
	// Bottom-left corner
	path.addLine(to: CGPoint(x: rect.minX, y: rect.minY + radius))
	path.addQuadCurve(to: CGPoint(x: rect.minX + radius, y: rect.minY),
					  controlPoint: CGPoint(x: rect.minX, y: rect.minY))
	
	path.close()
	return path.cgPath
}
```
2. .addQuadCurve 를 통해 설정한 radius는 UIBezierPath Corner radius와 차이가 있기 때문에 애니메이션이 끝나면 UIBezierPath로 다시 사각형을 그리고, 원하는 radius로 적용

코드가 조금 깔끔하지는 않지만 최종적으로 이렇게 하면 Gradient Border + Corner Radius + Masking을 애니메이션으로 구현할 수 있다

이 글이 CABasicAnimation으로 고통받고 있는 누군가에게 닿기를..

결과.gif :
![](https://i.imgur.com/tz9flQw.gif)
