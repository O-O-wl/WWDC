# High Performance Auto Layout

Category: UIKit
Created: Aug 3, 2020 5:39 PM
Property: https://developer.apple.com/videos/play/wwdc2018/220
Tags: 2018

> ❌ 
Don’t do this! Removes and re-adds constraints potentially at 120 frames per second

```swift
override func updateConstraints() {
	NSLayoutConstraint.deactivate(myConstraints)
	myConstraints.removeAll()
	let views = ["text1":text1, "text2":text2]
	myConstraints += NSLayoutConstraint.constraints(
		withVisualFormat: "H:|-[text1]-[text2]",
		options: [.alignAllFirstBaseline],
		metrics: nil, views: views
	)
	myConstraints += NSLayoutConstraint.constraints(
		withVisualFormat: "V:|-[text1]-|",
		options: [],
		metrics: nil, 
		views: views
	)
	NSLayoutConstraint.activate(myConstraints)
	super.updateConstraints()
}
```

> ✅
 This is ok!  Doesn't do anything unless self.myConstraints has been nil'd out

```swift
override func updateConstraints() {
	if self.myConstraints == nil {
		var constraints = [NSLayoutConstraint]()
		let views = ["text1": text1, "text2": text2]
		constraints += NSLayoutConstraint.constraints(
			withVisualFormat: "H:|-[text1]-[text2]",
			option: [.alignAllFirstBaseline],
			metrics: nil,
			views: views
		)
		constraints += NSLayoutConstraint.constraints(
			withVisualFormat: "V:|-[text1]-|",
			option: [.alignAllFirstBaseline],
			metrics: nil,
			views: views
		)
		NSLayoutConstraints.activate(constraints)
		self.myConstraints = constraints
	}
	super.updateConstraints()
}
```

## Render Loop

120 times every second.

### 3 Phase

 Window

       
Leaf View

![High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__5.58.42.png](High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__5.58.42.png)

Update Constraints

- every views will receive `updateCostraints()` from leaf view up to window
- 제약 조건

Layout

- `layoutSubviews()` from window up to leaf view
- 제약조건을 이용해 위치와 크기

Display

- `draw()`from window up to leaf view
- 화면에 렌더링

[Render  Loop 3 Phase](https://www.notion.so/09277b7bdffe483da1d746f90cf97d8c)

asd

> ❌ 
Don’t do this! Removes and re-adds constraints potentially at 120 frames per second

```swift
override func layoutSubviews() {
	text1.removeFromSuperview()
	text1 = nil
  text1 = UILabel(frame: CGRect(x: 20, y: 20, width: 300, height: 30))
	self.addSubview(text1)

	text2.removeFromSuperview()
	text2 = nil
  text2 = UILabel(frame: CGRect(x: 340, y: 20, width: 300, height: 30))
	self.addSubview(text2)
	
	super.layoutSubview()
}
```

![High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__7.25.27.png](High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__7.25.27.png)

![High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__7.40.33.png](High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__7.40.33.png)

![High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__7.41.27.png](High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__7.41.27.png)

## Performance Scales Linearly When No Interaction

![High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__8.00.55.png](High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__8.00.55.png)

equal constraint apply to another view happen linear caculation

![High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__7.56.06.png](High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__7.56.06.png)

### linear performance

reseason: aren't denpendencies between these

> **The engine is a layout cache and dependency tracker**

## Building a More Performant Layoyt

![High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__8.33.52.png](High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__8.33.52.png)

![High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__8.33.45.png](High%20Performance%20Auto%20Layout%205ec9cec4cd8d4a6398657022a78babae/_2020-08-03__8.33.45.png)

toggle constaints (active, deactive)

add constaints by all condition, selective active

### Constaints Churn

- Avoid removing all constraints
- Add static constraints once
- Only change the constraints that need changing
- **Hide views instead of removing them**

## instrinsic Content Size

Not all views need it

Views with non view content:

- UIImaegView returns its image size
- UILabel return ist text size

UIView uses it to make constraints

### Overriding

Optionally override for text measurement

- Return size if known without text measurement
- Use `UIView.noIntrinsicMetric`(-1) and Constraints

# **System Layout Size Fitting Size**

`systemLayoutSizeFitting(_ targetSize:)`

Used when combining with frames