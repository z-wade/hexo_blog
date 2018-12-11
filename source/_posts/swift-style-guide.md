---
title: 编码规范
date: 2018-12-11 16:39:54
tags: Swift
categories: 其他

---

# Swift 编码规范

## 正确性

尽量使代码在没有警告下编译

### 命名
描述性和一致性的命名会使代码更新易读易懂，使用`Swift`官方命名约定中描述的 [API Design Guidelines](https://swift.org/documentation/api-design-guidelines/)。现总结常用几点：

* 避免不常用的缩写
* 命名清晰比简洁重要
* 使用驼峰命名方法
* 类、协议、枚举不用像`Objectice-c`那样添加前缀

### 类前缀

Swift 自动以模块名称作为命名空间，不需要给类添加前缀，例如：RW。如果相同的两个类在不同的命名空间下，可以增加模块名字作为前缀。

```
import SomeModule
let myClass = MyModule.UsefulClass()
```

### Delegate
在创建自定义的`delegate`方法时，第一个参数不带名字的参数应该是`delegate`原来的对象

**Preferred**

```
func namePickerView(_ namePickerView: NamePickerView, didSelectName name: String)
func namePickerViewShouldReload(_ namePickerView: NamePickerView) -> Bool
```

**Not Preferred**:

```
func didSelectName(namePicker: NamePickerViewController, name: String)
func namePickerShouldReload() -> Bool
```

### 使用类型推断上下文

使用类型推断可以使代码整洁 (可参考[Type Inference](#type-inference).)

**Preferred**:

```swift
let selector = #selector(viewDidLoad)
view.backgroundColor = .red
let toView = context.view(forKey: .to)
let view = UIView(frame: .zero)
```

**Not Preferred**:

```swift
let selector = #selector(ViewController.viewDidLoad)
view.backgroundColor = UIColor.red
let toView = context.view(forKey: UITransitionContextViewKey.to)
let view = UIView(frame: CGRect.zero)
```

### 泛型 

泛型命名应该是可描述性的，使用驼峰命名。当类型名字没有意义时，使用传统的大写字母：T，U 或者 V

**Preferred**:

```swift
struct Stack<Element> { ... }
func write<Target: OutputStream>(to target: inout Target)
func swap<T>(_ a: inout T, _ b: inout T)
```

**Not Preferred**:

```swift
struct Stack<T> { ... }
func write<target: OutputStream>(to target: inout target)
func swap<Thing>(_ a: inout Thing, _ b: inout Thing)
```

## 类和结构体

### 应该用哪个

结构体是[值类型](https://developer.apple.com/library/mac/documentation/Swift/Conceptual/Swift_Programming_Language/ClassesAndStructures.html#//apple_ref/doc/uid/TP40014097-CH13-XID_144)。使用结构体的对象是不具有唯一性，例如：数组[a,b,c]和另外的一个定义在其他的数组[a,b,c]是可以互换的。不管是第一个数组还是第二个数组，它们所代表的含义是一样的。这就是为什么数组是结构体的原因。

类是[引用类型](https://developer.apple.com/library/mac/documentation/Swift/Conceptual/Swift_Programming_Language/ClassesAndStructures.html#//apple_ref/doc/uid/TP40014097-CH13-XID_145)。对拥有自己的id或者有生命周期的事物使用类。常常把人建模为一个类，两个人的对象是不同的东西，就算他们有相同的名字和生日，也不意味着他们是同一个人。但是生日应该为结构体，因为1950年3月的日期跟其他相同日期表示的意思是一样的。

### 例子
这是一个比较不错的定义Class的例子

```swift
class Circle: Shape {
  var x: Int
  var y: Int
  var radius: Double
  var diameter: Double {
    get {
      return radius * 2
    }
    set {
      radius = newValue / 2
    }
  }

  init(x: Int, y: Int, radius: Double) {
    self.x = x
    self.y = y
    self.radius = radius
  }

  convenience init(x: Int, y: Int, diameter: Double) {
    self.init(x: x, y: y, radius: diameter / 2)
  }

  override func area() -> Double {
    return Double.pi * radius * radius
  }
  
  //MARK: - CustomStringConvertible
  
  var description: String {
    return "center = \(centerString) area = \(area())"
  }
  private var centerString: String {
    return "(\(x),\(y))"
  }
}
```

### 使用Self

为了简洁起见，避免使用`self`，因为`Swift`不强求使用`self`来访问对象的属性或者调用方法。

只有在编译器要求使用`self`的时候，（在`@escaping`闭包里面或者在对象初始化的方法里面为了消除歧义），否则在没有编译器提醒时都应该省略它

### 计算属性

**Preferred**:

```
var diameter: Double {
  return radius * 2
}
```

**Not Preferred**:

```
var diameter: Double {
  get {
    return radius * 2
  }
}
```

### Final关键字

如果你的类不需要派生子类，那就给类定义为`final`吧

```
// Turn any generic type into a reference type using this Box class.
final class Box<T> {
  let value: T
  init(_ value: T) {
    self.value = value
  }
}
```

## 函数定义

函数定义在一行可以定义完包括大括号

```
func reticulateSplines(spline: [Double]) -> Bool {
  // reticulate code goes here
}
```

多个参数，让每个参数应该在新的一行

```
func reticulateSplines(
  spline: [Double], 
  adjustmentFactor: Double,
  translateConstant: Int, comment: String
) -> Bool {
  // reticulate code goes here
}
```

使用Void表示缺省

**Preferred**:

```
func updateConstraints() -> Void {
  // magic happens here
}

typealias CompletionHandler = (result) -> Void
```

**Not Preferred**:

```
func updateConstraints() -> () {
  // magic happens here
}

typealias CompletionHandler = (result) -> ()
```

## 函数调用

Mirror the style of function declarations at call sites. Calls that fit on a single line should be written as such:

```swift
let success = reticulateSplines(splines)
```

If the call site must be wrapped, put each parameter on a new line, indented one additional level:

```swift
let success = reticulateSplines(
  spline: splines,
  adjustmentFactor: 1.3,
  translateConstant: 2,
  comment: "normalize the display")
```

## 闭包表达式

只有参数列表末尾有一个闭包表达式参数时，才使用尾随闭包。否则还是应该加上参数名字

**Preferred**:

```
UIView.animate(withDuration: 1.0) {
  self.myView.alpha = 0
}

UIView.animate(withDuration: 1.0, animations: {
  self.myView.alpha = 0
}, completion: { finished in
  self.myView.removeFromSuperview()
})
```

**Not Preferred**:

```
UIView.animate(withDuration: 1.0, animations: {
  self.myView.alpha = 0
})

UIView.animate(withDuration: 1.0, animations: {
  self.myView.alpha = 0
}) { f in
  self.myView.removeFromSuperview()
}
```

对于上下文清晰的单个参数表达式时，使用隐式返回:

```swift
attendeeList.sort { a, b in
  a > b
}
```

使用尾随闭包的链式方法上下文应该是易读的。而间隔换行、参数等由作者自觉定义：

```
let value = numbers.map { $0 * 2 }.filter { $0 % 3 == 0 }.index(of: 90)

let value = numbers
  .map {$0 * 2}
  .filter {$0 > 50}
  .map {$0 + 10}
```

## 类型

尽量使用`Swfit`的原生类型表达式

**Preferred**:

```
let width = 120.0                                    // Double
let widthString = "\(width)"                         // String
```

**Less Preferred**:

```
let width = 120.0                                    // Double
let widthString = (width as NSNumber).stringValue    // String
```

**Not Preferred**:

```
let width: NSNumber = 120.0                          // NSNumber
let widthString: NSString = width.stringValue        // NSString
```

在使用`CG`开头的相关类型，使用`CGFloat`会使代码更加清晰易读

### 常量 

常量可以应该用`let`关键字定义

**Tip:** 最好是全部都使用`let`，只有要编译有问题时才使用`var`

定义类常量比实例常量会更好。定义类常量使用`static let`关键字。

**Preferred**:

```
enum Math {
  static let e = 2.718281828459045235360287
  static let root2 = 1.41421356237309504880168872
}

let hypotenuse = side * Math.root2

```

**Not Preferred**:

```
let e = 2.718281828459045235360287  // pollutes global namespace
let root2 = 1.41421356237309504880168872

let hypotenuse = side * root2 // what is root2?
```

### Optionals 可选类型

如果定义变量和函数返回值有可能为`nil`，应该定义为可选值`?`

使用`!`定义强制解包类型，只有在你接下来会明确该变量被会初始化。例如：将会在`viewDidLoad()`方法实例的子view。

访问可选类型时，如果有多个类型或者只访问一次，可以使用语法链

```
textContainer?.textLabel?.setNeedsDisplay()
```
多处使用应该使用一次性绑定

```
if let textContainer = textContainer {
  // do many things with textContainer
}
```

是否可以类型不应该出现在命名中，例如：`optionalString`、`maybeView`、`unwrappedView`,因为这些信息已经包含在类型声明中

**Preferred**:

```
var subview: UIView?
var volume: Double?

// later on...
if let subview = subview, let volume = volume {
  // do something with unwrapped subview and volume
}

// another example
UIView.animate(withDuration: 2.0) { [weak self] in
  guard let self = self else { return }
  self.alpha = 1.0
}
```

**Not Preferred**:

```
var optionalSubview: UIView?
var volume: Double?

if let unwrappedSubview = optionalSubview {
  if let realVolume = volume {
    // do something with unwrappedSubview and realVolume
  }
}

// another example
UIView.animate(withDuration: 2.0) { [weak self] in
  guard let strongSelf = self else { return }
  strongSelf.alpha = 1.0
}
```

### 类型推断

让编译器推断变量或者常量的类型，当真需要时，才会指定特定的类型，例如： `CGFloat`和 `Int16`

**Preferred**:

```
let message = "Click the button"
let currentBounds = computeViewBounds()
var names = ["Mic", "Sam", "Christine"]
let maximumWidth: CGFloat = 106.5
```

**Not Preferred**:

```
let message: String = "Click the button"
let currentBounds: CGRect = computeViewBounds()
var names = [String]()
```

### 语法糖

尽量使用较短快捷的定义版本

**Preferred**:

```
var deviceModels: [String]
var employees: [Int: String]
var faxNumber: Int?
```

**Not Preferred**:

```
var deviceModels: Array<String>
var employees: Dictionary<Int, String>
var faxNumber: Optional<Int>
```

## 内存管理

代码无论在什么时候都不应该产生循环引用，使用`weak`和`unowned`引用防止产生循环引用，或者使用值类型。

### 延长对象寿命
延长对象寿命使用`[weak self]` 和 `guard let = self else { return }` 语法。

**Preferred**

```swift
resource.request().onComplete { [weak self] response in
  guard let self = self else {
    return
  }
  let model = self.updateModel(response)
  self.updateUI(model)
}
```

**Not Preferred**

```swift
// might crash if self is released before response returns
resource.request().onComplete { [unowned self] response in
  let model = self.updateModel(response)
  self.updateUI(model)
}
```

**Not Preferred**

```swift
// deallocate could happen between updating the model and updating UI
resource.request().onComplete { [weak self] response in
  let model = self?.updateModel(response)
  self?.updateUI(model)
}
```

## 控制流

Prefer the `for-in` style of `for` loop over the `while-condition-increment` style.

**Preferred**:

```swift
for _ in 0..<3 {
  print("Hello three times")
}

for (index, person) in attendeeList.enumerated() {
  print("\(person) is at position #\(index)")
}

for index in stride(from: 0, to: items.count, by: 2) {
  print(index)
}

for index in (0...3).reversed() {
  print(index)
}
```

**Not Preferred**:

```swift
var i = 0
while i < 3 {
  print("Hello three times")
  i += 1
}


var i = 0
while i < attendeeList.count {
  let person = attendeeList[i]
  print("\(person) is at position #\(i)")
  i += 1
}
```

### 三元表达式

**Preferred**:

```swift
let value = 5
result = value != 0 ? x : y

let isHorizontal = true
result = isHorizontal ? x : y
```

**Not Preferred**:

```swift
result = a > b ? x = c > d ? c : d : y
```

## 黄金路径

当使用条件编写代码时，应该及时`return`。也就是说，不要嵌套`if`语句，关键字`guard`你值得了解

**Preferred**:

```swift
func computeFFT(context: Context?, inputData: InputData?) throws -> Frequencies {

  guard let context = context else {
    throw FFTError.noContext
  }
  guard let inputData = inputData else {
    throw FFTError.noInputData
  }

  // use context and input to compute the frequencies
  return frequencies
}
```

**Not Preferred**:

```swift
func computeFFT(context: Context?, inputData: InputData?) throws -> Frequencies {

  if let context = context {
    if let inputData = inputData {
      // use context and input to compute the frequencies

      return frequencies
    } else {
      throw FFTError.noInputData
    }
  } else {
    throw FFTError.noContext
  }
}
```

如果是多个条件的情况下，使用`guard`更加清晰明了。例子：

**Preferred**:

```swift
guard 
  let number1 = number1,
  let number2 = number2,
  let number3 = number3 
  else {
    fatalError("impossible")
}
// do something with numbers
```

**Not Preferred**:

```swift
if let number1 = number1 {
  if let number2 = number2 {
    if let number3 = number3 {
      // do something with numbers
    } else {
      fatalError("impossible")
    }
  } else {
    fatalError("impossible")
  }
} else {
  fatalError("impossible")
}
```

## 分号

Swift 不要求在每句语句后面加分号，只要在多句语句在同一行的时才需要分号

不要把多行语句在同一行用分号隔开来

**Preferred**:

```swift
let swift = "not a scripting language"
```

**Not Preferred**:
```swift
let swift = "not a scripting language";
```

## 括号

在条件语句中括号不是一定要的，应该省略掉。

**Preferred**:

```swift
if name == "Hello" {
  print("World")
}
```

**Not Preferred**:

```swift
if (name == "Hello") {
  print("World")
}
```

很长的表达式中，使用括号可以使代码更加清晰

**Preferred**:

```swift
let playerMark = (player == current ? "X" : "O")
```

## 多行字符串

当创建很长字符串时，应该使用多行字符串语法

**Preferred**:

```swift
let message = """
  You cannot charge the flux \
  capacitor with a 9V battery.
  You must use a super-charger \
  which costs 10 credits. You currently \
  have \(credits) credits available.
  """
```

**Not Preferred**:

```swift
let message = """You cannot charge the flux \
  capacitor with a 9V battery.
  You must use a super-charger \
  which costs 10 credits. You currently \
  have \(credits) credits available.
  """
```

**Not Preferred**:

```swift
let message = "You cannot charge the flux " +
  "capacitor with a 9V battery.\n" +
  "You must use a super-charger " +
  "which costs 10 credits. You currently " +
  "have \(credits) credits available."
```


## References

* [The Official raywenderlich.com Swift Style Guide.
](https://github.com/raywenderlich/swift-style-guide/blob/master/README.markdown)
* [The Swift API Design Guidelines](https://swift.org/documentation/api-design-guidelines/)

