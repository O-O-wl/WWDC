# Swift Generics

Created: May 23, 2020 3:44 PM
Tags: 2018

# Why Generics?

```swift
struct Buffer {
    var count: Int
    
    subscript(at: Int) -> ??? { }
}
```

`Buffer` 의 `Index` 의  위치에 따라 `Element` 를 가져오는 기능을 구현

- 반환 타입을 어떻게 명시할 수 있을까

### Generic이 없다면?

우리에게는 `Any` 이 있다!

```swift
struct Buffer {
    var count: Int
    
    subscript(at: Int) -> Any {}
}
```

하지만 이건 좋은 방법이 아니다.

이런 방법에서 우리는 String만을 위한 Buffer에 Int를 넣을 수도 있다.  이건 우리가 원한 사용성이 아니다.

우리의 또 하나의 문제는 메모리안에 값을 표현하는 것과 관련된다.

String Buffer의 우리가 원하는 이상적인 표현은 연속적인 메모리 블록에 표현하는 것이다.

메모리를 연속적으로 할당하기 위해서는 Type의 크기를 알아야합니다. 또 Any는 tracking, boxing, unboxing 등의 오버헤드 비용이 발생합니다.

우리는 성능상의 문제, 명확한 사용성을 위해 이 문제를 해결하기를 원합니다.

Buffer는 이제 자신이 포함할 것들의 타입을 표현합니다. 

```swift
struct Buffer<Element> {
    var count: Int
    
    subscript(at: Int) -> Element {}
}
```

Element로 표현하고 이 타입은 Generic Parameter 라고 부립니다. 이건 Parametric Polymorphism을 강화합니다. 우리는 컴파일 타임에서 Buffer가 어떤 타입을 가질 지 알려주면 됩니다.

타입 추론도 잘해서

```swift
let buffer: Buffer = [1,2,3] 
```

으로 선언해도 잘 정의됩니다.

우리는 연속적인 메모리 블록에 요소들을 잡는 목표도 달성할 수 있습니다. 이 과정에서 오버헤드도 해결됩니다.

우리는 Buffer의 Int를 더하기위해서, sum 메소드를 구현합니다.

```swift
extension Buffer {
    func sum() -> Element {
        
        var total = 0
        for i in 0..<self.count {
            total += self[i]
        }
        return total
    }
}
```

여기서의 문제는 Element 과 Int는 같다는 것을 확정지을 수 없습니다.

```swift
extension Buffer where Element: Numeric {
    func sum() -> Element {
        
        var total = Element.zero
        for i in 0..<self.count {
            total += self[i]
        }
        return total
    }
}
```

이 방법은 Element는 Numeric 프로토콜 구현체만에서의 확장이 이루어지고 우리가 원하던 바를 달성할 수 있습니다.

# Designing a Protocol

```swift
protocol Collection {
    associatedtype Element
    
    var count: Int { get }
    subscript(at: Int) -> Element { get set }
}
```

Collection 이라는 프로토콜이 존재한다.

우리는 요소를 Index로 접근하려고 하고, count를 기본 구현하기를 원한다.

```swift
protocol Collection {
    associatedtype Element
    associatedtype Index
    
    subscript(at: Int) -> Element { get }
    func index(after: Index) -> Index
    var startIndex: Index { get }
    var endIndex: Index { get }
}
extension Collection {
    var count: Int {
        var i = 0
        var position = startIndex
        while position != endIndex {
            position = Index(after: position)
            i += 1
        }
        return i
    }
}
```

여기서 생기는 문제는 

position != endIndex Index타입은 비교연산이 가능한지를 확정할 수없다.

```swift
protocol Collection {
    associatedtype Element
    associatedtype Index: Equatable
    
    subscript(at: Int) -> Element { get }
    func index(after: Index) -> Index
    var startIndex: Index { get }
    var endIndex: Index { get }
}
```

Index 타입에 Equatable을 강제하는 방식으로 이를 해결할 수 있다.

# Customization Points

map 메소드를 참고해보자

```swift
extension Collection {
    func map<T>(_ transform: (Element) -> T) -> [T] {
        var result: [T] = []
        
        var position = startIndex
        while position != endIndex {
            result.append(transform(self[position]))
            position = index(after: position)
        }
        return result
    }
}
```

새로운 빈 array을 만들고, 각각의 Element를 transform 하여서 array에 append한다.

그 과정에서 자동으로 array는 자동적으로 커진다.

새로운 storage에 재할당하게 된다.

이런 메모리 할당들은 꽤 비싼 연산이다.

우리는 미리 이 공간을 예약해둠으로써 더 빠르게 할 수 있다.

```swift
extension Collection {
    func map<T>(_ transform: (Element) -> T) -> [T] {
        var result: [T] = []
        
        result.reserveCapacity(self.count)
        
        var position = startIndex
        while position != endIndex {
            result.append(transform(self[position]))
            position = index(after: position)
        }
        return result
    }
}
```

```swift
protocol Collection {
	 var count: Int { get }
}
```

그리고 구체타입에서는 count의 구현할 것이기에 문제없이 동작할 것이다.

# Protocol Inheritance

![Swift%20Generics%207a8f34b95b744f9ebc7ab14621db20a4/_2020-05-30__4.26.34.png](Swift%20Generics/_2020-05-30__4.26.34.png)

우리가 이전에 만든 Collection은 SinglelyLinkedCollection 이다.

양방향으로 가능한 Collection을 만들수 있다.

그러나 그 과정에서 Collection의 기본구현과 퍼블릭 인터페이스를 접근하기 위해서는 protocol inheritance를 이용할 수 있다.

```swift
extension BidirectionalCollection {
    func lastIndex(where predicate: (Element) -> Bool) -> Index? {
        var position = endIndex
        while position != startIndex {
            position = index(after: position)
            if predicate(self[position]) { return position }
        }
        return nil
    }
}
```

`index(after: position)` 이 함수의 구현을 보장 할 수있다.

Shuffle을 구현한 걸 프로토콜만들어보는 건 어떨까요?

```swift
protocol ShuffleCollection: Collection {
    mutating func shuffle()
}
extension ShuffleCollection {
    mutating func shuffle() {
        let n = count
        guard n > 1 else { return }
        for (i, position) in indices.dropLast().enumerated() {
            let otherPostion = index(startIndex, offsetBy: Int.random(in: i..<n))
        }
        swapAt(position, otherPostion)
    }
}
```

이렇게 하나의 알고리즘을 가지게 하기위해서 프로토콜을 정의하는 건 안티패턴입니다.

이런식으로 요구사항에 따라 프로토콜을 하나씩 구현하게 되면 의미없는 프로토콜들이 엄청 많이 생겨나게됩니다.

우리는 shuffle을 위해서는 random access 와 mutation이 필요하다는 것을 알 수 있습니다.

우리는 이것들을 분리하고 프로토콜을 분류할 수 있습니다.

```swift
protocol RandomAccessCollection: BidirectionalCollection {
    func index(_ position: Index, offsetBy n: Index) -> Index
    func distance(from start: Index, to end: Index) -> Int
}

protocol MutableCollection: Collection {
    subscript (index: Index) -> Element { get set }
    mutating func swapAt(_: Index, _: Index)
}
```

이렇게 더 능력에 따라 분리를 할 수 있습니다.

 

또 이 분리한 프로토콜 두개를 만족하는 타입에 한해서 우리는 shuffle을 가능하게 기본 구현 할 수 있습니다.

```swift
extension RandomAccessCollection where Self: MutableCollection {
    mutating func shuffle() {
        let n = count
        guard n > 1 else { return }
        for (i, position) in indices.dropLast().enumerated() {
            let otherPostion = index(startIndex, offsetBy: Int.random(in: i..<n))
        }
        swapAt(position, otherPostion)
    }
}
```

이렇게 말이죠 이렇게 우리는 여러 프로토콜을 조합할 수 있습니다.

# Recursive Constraints

프로토콜에 재귀적인 제약을 걸 수 있습니다.

```swift
protocol Collection {
    associatedtype SubSequence: Collection
}
```

Binary Search를 아시나요?

이 알고리즘을 구현하는 데에는 Collection의 SubSequence를 필요로 합니다.

계속적인 SubSequence에 대한 재귀적인 탐색을 아래와 같은 타입을 따라가게 됩니다.

```swift
protocol Collection {
    associatedtype Element
    associatedtype Index: Equatable
    associatedtype SubSequence: Collection

    subscript (range: Range<Index>) -> SubSequence { get }
}
```

우리는 Binary Search 과정에서 SubSequence에 재귀적으로 접근하게 됩니다.

```swift
self[bounds][bounds][bounds][bounds][bounds][bounds][bounds][bounds][bounds][bounds][bounds]Self.SubSequence.SubSequence.SubSequence.SubSequence.SubSequence.SubSequence.SubSequence.SubSequ

```

이의 타입은

```swift
Self.SubSequence.SubSequence.SubSequence.SubSequence.SubSequence.SubSequence
```

이 될겁니다.

이 재귀적인 타입 확정을 끊을 수 있습니다.

```swift
protocol Collection {
    associatedtype Element
    associatedtype Index: Equatable
    associatedtype SubSequence: Collection
        where
        SubSequence.Element == Element,
        SubSequence.Index == Index,
        SubSequence.SubSequence == SubSequence
}
```

# Generics and Classes

```swift
class Vehicle {
    //...
}

class Taxi: Vehicle {
    //...
}

class PoliceCar: Vehicle {
    //...
}

extension Vehicle {
    func drive() {}
}

let taxi = Taxi()
taxi.drive()
```

리스코프 치환 원칙에 따라 서브클래스는 슈퍼클래스로 완전히 치환가능해야합니다.

프로토콜에서도 가능합니다.

```swift
protocol Drivable {
    func drive()
}

extension Vehicle: Drivable {}
```

프로토콜의 채택은 서브 클래스에도 내려간다.

모든 서브 클래스는 이 동작을 하게됩니다.

한번 Decodable 프로토콜을 봅시다.

```swift
protocol Decodable {
    init(from decoder: Decoder) throws
}

extension Decodable {
    static func decode(from decoder: Decoder) throws -> Self {
        return try self.decode(from: decoder)
    }
}
```

```swift
class Vehicle: Decodable {
      // ..
}

Taxi.decode(from: decoder)
```

여기서  문제는 initializer는 직접 서브클래싱될 수 없습니다. 이건 강제화된 구현을 서브클래스로 미루어야합니다.

```swift
class Taxi: Vehicle {
    var hourlyRate: Double
}
```

hourlyRate와 같은 프로퍼티를  슈퍼클래스의 initializer는 초기화할 수없습니다.

```swift
class Vehicle: Decodable {
    required init(from decoder: Decoder) throws {
    }
    
    
    //...
```

를 강제하게 되고 이를 면제하는 방법은

final 키워드를 사용하는 방법이 있습니다.

# **Summary**

- 스위프트 제네릭은 정적 타입 코드중 재사용 가능한 코드를 재사용하게 해줍니다.
- 프로토콜 상속은 기능들을 특정 타입에 대해서 수용가능하게 할 수 있고, 이 조합들은 우리에게 조건적인 프로토콜 순응을 제공해줍니다.
- 클래스를 가지고 리스코프 치환 원칙을 적용도 할 수 있습니다.
