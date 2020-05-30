# Protocol and Value Oriented Programming in UIKit Apps

Created: May 30, 2020 7:51 PM
Tags: 2016

# Model

```swift
class Dream {
    var description: String
    var creature: Creature
    var effects: Set<Effect>
    // ...
}
```

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__8.05.21.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__8.05.21.png)

공통된 값을 참조에 따른 버그

Complicated RelationShips

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__8.05.53.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__8.05.53.png)

복잡해지는 상태에서는 버그 추적이 더 어려워진다.

Value Semantics

```swift
struct Dream {
    var description: String
    var creature: Creature
    var effects: Set<Effect>
    // ...
}
```

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__8.11.52.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__8.11.52.png)

각각의 모델은 독립된 상태로 관리된다.

❌ Use values only for simple model types .❌

# View

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__8.15.37.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__8.15.37.png)

DreamDetailView , DreamCell 공통된 Layout이 있으나 이런 계층은 Layout을 tableview 외부 (DreamDetailView)에서 사용하기 어렵게 한다

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__8.27.12.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__8.27.12.png)

```swift
class DecoratingLayoutCell : UITableViewCell {
   var content: UIView
   var decoration: UIView
   // Perform layout...
}
```

```swift
struct DecoratingLayout {
   var content: UIView
   var decoration: UIView

   mutating func layout(in rect: CGRect) { 
   // Perform layout...
   }
}
```

상속계층에서 벗어나며, 레이아웃의 코드를 분리해내기위해 Layout 구조체를 정의하겠습니다.

```swift
class DreamDetailView : UIView { ...
   override func layoutSubviews() {
      var decoratingLayout = DecoratingLayout(content: content, decoration: decoration) decoratingLayout.layout(in: bounds)
   } 
}
```

```swift
class DreamCell : UITableViewCell { ...
   override func layoutSubviews() {
      var decoratingLayout = DecoratingLayout(content: content, decoration: decoration) decoratingLayout.layout(in: bounds)
    } 
}
```

와 같이 레이아웃 코드를 재사용 할 수 있습니다.

```swift
func testLayout() {
   let child1 = UIView()
   let child2 = UIView()
   var layout = DecoratingLayout(content: child1, decoration: child2)
   layout.layout(in: CGRect(x: 0, y: 0, width: 120, height: 40))
   XCTAssertEqual(child1.frame, CGRect(x: 0, y: 5, width: 35, height: 30)) XCTAssertEqual(child2.frame, CGRect(x: 35, y: 5, width: 70, height: 30))
}
```

와 같이 레이아웃에 대한 테스트도 쉬워집니다.

# Local Reasoning

```swift
struct DecoratingLayout {
   var content: UIView    // -> SKNode
   var decoration: UIView // -> SKNode

   mutating func layout(in rect: CGRect) { 
   // Perform layout...
   }
}
```

위와 같이 다른 계층의 타입에 레이아웃을 적용해야하는 필요성이 생긴다면? 
UIView & SKNode의 속성중 공통된 필요한 속성에 대한 추상화

```swift
protocol Layout {
   var frame: CGRect { get set }
}
```

```swift
struct DecoratingLayout {
   var content: Layout
   var decoration: Layout
   mutating func layout(in rect: CGRect) {
   content.frame = ...
   decoration.frame = ... 
   }
}

extension UIView: Layout
extension SKNode: Layout {}
```

와 객체의 합성으로 구현이 가능해집니다.

# Generic Types

- 더 많은 타입에 대한 제어
- 컴파일타임의 최적화

상속을 통한 코드의 공유

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__8.49.01.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__8.49.01.png)

### Techniques

- Local reasoning with value types
- Generic types for fast, safe polymorphism
- Composition of values

# Controller `Undo`

```swift
class DreamListViewController : UITableViewController {
    var dreams: [Dream] // Model
    var favoriteCreature: Creature // Model
    // ...
}
```

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__8.57.00.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__8.57.00.png)

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__8.56.49.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__8.56.49.png)

각각의 모델(`Dream`, `Creature` )의 변경을 View에 반영하는 건 복잡합니다.

이 과정에서 모델과 뷰의 불일치로 인해 버그는 생겨납니다.

Undo를 구현하는 데 있어서

Model이 있고, Model의 변경사항을 `UndoManager Stack` 에 담습니다.

변경사항(`dreams.removeLast`)

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__9.05.26.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__9.05.26.png)

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__9.08.06.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__9.08.06.png)

변경 사항에 따른 역변경을 구현해야합니다.

- 설정 → 취소(설정을 해야 취소 가능)

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__9.07.53.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__9.07.53.png)

현재 모델의 상태를 그대로 Stack에 저장하는 방법으로 쉽게 Undo를 버그없이 구현할 수 있습니다.

### Benefits

- Single code path
    - Better local reasoning - 변경에 따른 경로 추적이 지역에 따라 한정되게 됨
- Value compose well with other values
    - 다른 값들과 조합을 잘 해둘 수 있다.

# Controller - `UI State`

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__9.15.06.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__9.15.06.png)

`

```swift
class DreamListViewController : UITableViewController {
  var isInViewingMode: Bool   // UI State
  var sharingDreams: [Dream]? // UI State
  var selectedRows: IndexSet? // UI State
}
```

이 State를 관리하는 코드는 버그를 유발하기 아주 좋다.

이 상태들은 그리고 동시에 존재할 수 없는 State들이다.

```swift
enum State {
   case viewing
   case sharing(dreams: [Dream])
   case selecting(selectedRows: IndexSet)
} 
```

```swift
class DreamListViewController: UITableViewController {
    var state: State
}
```

버그의 추적이 쉬워지고 동시에 가질 수 없게 제한 시키는 방법으로 구현할 수 있다.

![Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit%20A%201864acadb98d412687ef356311449018/_2020-05-30__9.20.04.png](Protocol%20and%20Value%20Oriented%20Programming%20in%20UIKit/_2020-05-30__9.20.04.png)

# Techniques and Tools

- Customization through composition
- Protocols for generic, reusable code
- Taking advantage of value semantics
- Local reasoning
