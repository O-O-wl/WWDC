# LLDB: Beyond "po"

Created: May 23, 2020 1:49 PM

LLDB는 App에서 버그를 조사하는 동안 변수를 인쇄하는 기능을 제공한다.

LLDB는 이 작업을 3가지의 방법으로 제공한다.

첫번째로  `po` 이다.

객체 설명을 인쇄하는 명령이다. 

객체의 인쇄문은 인스턴스를 텍스트로 표현한 description이다.

`CustomDebugStringConvertible` 을 채택 후 `debugDescription`을 구현하면 해당 문자열이 디버깅 중 타입을 설명하는 인쇄문으로 출력된다.

```swift
struct Trip {
    let name: String
    let destinations: [String]
}
```

```swift
(lldb) po cruise
▿ Trip
  - name : "크루저 여행"
  ▿ destinations : 3 elements
    - 0 : "여수"
    - 1 : "서울"
    - 2 : "천안"
```

```swift
extension Trip: CustomDebugStringConvertible {
    var debugDescription: String {
        return "여행타입"
    }
}
```

```swift
(lldb) po cruise
▿ 여행타입
  - name : "크루저 여행"
  ▿ destinations : 3 elements
    - 0 : "여수"
    - 1 : "서울"
    - 2 : "천안"
```

하위 구조의 수정을 요한다면 `CustomerReflectable` 을 사용할 수 있다.

`po`는 단순히 변수를 출력하기 위한 용도만 사용하는 것이 아니다.

```swift
(lldb) po cruise.destinations.sorted()

▿ 3 elements
  - 0 : "서울"
  - 1 : "여수"
  - 2 : "천안"
```

사실 근데 `po` 는 `expression --object-description -- cruise` 의 alias 입니다.

```swift
(lldb) command alias my_po expression --object-description
(lldb) my_po cruise
▿ 여행타입
  - name : "크루저 여행"
  ▿ destinations : 3 elements
    - 0 : "여수"
    - 1 : "서울"
    - 2 : "천안"
```

`po` 가 일어나는 내부를 들어가보면

![LLDB%20Beyond%20po%20b59a68ceafdc4d9db34dcfcd839d6a0e/_2020-05-23__6.02.37.png](LLDB%20Beyond%20po%20b59a68ceafdc4d9db34dcfcd839d6a0e/_2020-05-23__6.02.37.png)

Create compliable code - 컴파일 가능한 소스를 생성

```swift
func __lldb_expr() {
		__lldb_res = view
}
```

Complie - 컴파일

Excute - 실행

Get result - 실행 후 나온 결과의 객체 접근

Create code to access description - 객체에 접근하기 위한 코드  생성

```swift
func __lldb_expr2 -> String() {
		return __lldb_res.description
}
```

Complie - 재 컴파일

Execute - 실행

Get string result - 객체의 descrition get

Display result string - 출력

두번쨰는 `p` 이다.

이 명령은 객체의 description없이 출력이 된다.

```swift
**(lldb) p cruise
(WWDC18.Trip) $R10 = {
  name = "크루저 여행"
  destinations = 3 values {
    [0] = "여수"
    [1] = "서울"
    [2] = "천안"
  }
}**
```

p는 설명을 가져오는 작업들이 필요없어서 Get result 까지만 하여 퍼포먼스상의 이점이 존재합니다.

![LLDB%20Beyond%20po%20b59a68ceafdc4d9db34dcfcd839d6a0e/_2020-05-23__7.30.00.png](LLDB%20Beyond%20po%20b59a68ceafdc4d9db34dcfcd839d6a0e/_2020-05-23__7.30.00.png)

하지만 정적인 타입을 참고하기때문에 런타임상에서 동적인 인스턴스이 프로퍼티에는 접근할 수 없습니다. 
동적인스턴스의 프로퍼티 출력을 위해서는 명시적인 캐스팅이 필요합니다.

그 이후 Formatter는 타입을 일반적으로 사용되는 유형의 설명으로  보여줍니다.

만약 Formatter를 거치지 않는다면 이런 형태가 됩니다.

```swift
(lldb) expression --raw -- (cruise as! Trip).name
(Swift.String) $R0 = {
  _guts = {
    _object = {
      _countAndFlagsBits = {
        _value = 1152921504606846992
      }
      _object = 0x800000010021cb40
    }
  }
}
```

마지막으로 `v` 입니다.

v명령은 이전과 다른 행보를 가집니다. 코드를 컴파일하지 않고 실행해서 매우 빠릅니다.

```swift
v (cruise as! Trip).name
```

이런 건 통하지 않아요 컴파일을 안하거든요!

![LLDB%20Beyond%20po%20b59a68ceafdc4d9db34dcfcd839d6a0e/_2020-05-23__7.34.38.png](LLDB%20Beyond%20po%20b59a68ceafdc4d9db34dcfcd839d6a0e/_2020-05-23__7.34.38.png)

v 는 변수를 찾고, 변수의 값을 읽어옵니다. 동적인 인스턴스의 타입을 확인합니다. 만약 체이닝을 해서 더 안쪽의 프로퍼티를 참조하게된다면, 이건 중첩되게 동작합니다. 그 후 포맷팅된 값이 반환되죠.

정리합니다

![LLDB%20Beyond%20po%20b59a68ceafdc4d9db34dcfcd839d6a0e/_2020-05-23__7.36.20.png](LLDB%20Beyond%20po%20b59a68ceafdc4d9db34dcfcd839d6a0e/_2020-05-23__7.36.20.png)

Data Formatter를 사용자 정의할 수 도 있습니다.

Formatter를 사용하는 `p`, `v` 에서만 유효합니다.

Filter 부터 알아봅시다

```swift
**(lldb) type filter add WWDC18.Trip --child name
(lldb) v cruise
(WWDC18.Trip) cruise = (name = "크루저 여행")**
```

필터를 더함으로써 원하는 프로퍼티만 출력하는 것도 가능합니다.

String Summary는 

```swift
(lldb) type summary add WWDC18.Trip --summary-string"${var.name} from ${var.destinations[0]} to ${var.destinations[2]}"
```

은 우리가 커스텀한 포맷팅을 인쇄할 수 있습니다.

```swift
(lldb) 크루저 여행 from "여수"  to "천안"
```

Python을 이용하는 방법도 있습니다.

Python으로 작성된 .py를 import 할 수 있습니다.

```swift
(lldb) command script import Trip.py
```

```swift
(lldb) type summary add WWDC18.Trip --python-function Trip.SummaryProvider
```

이런식으로 파이썬 코드를 반입하고 난 후 그 코드를 활용도 가능합니다.