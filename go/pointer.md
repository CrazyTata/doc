在 Go 语言中，方法的调用规则对于值接收者和指针接收者有一些特别的行为，这使得代码变得更加简洁和易用。我们来详细解释一下“方法值调用规则”和“方法表达式调用规则”，以及它们如何影响代码的行为。

### 方法值调用（Method Value Call）

在 Go 中，方法调用的语法是 `receiver.method()`. 当一个方法具有接收者时，Go 允许我们在值接收者和指针接收者之间进行一定程度的自动转换。这些转换规则使得方法调用更加灵活。

#### 自动获取地址

当你调用一个接收者为指针的方法时，如果你持有的是一个值，Go 会自动获取这个值的地址并传递给方法。反过来，如果你调用一个接收者为值的方法时，持有的是指针，Go 会自动解引用指针并传递给方法。

### 举例解释

假设我们有以下结构体和方法：

```go
type Example struct {
    Value int
}

func (e Example) ValueMethod() {
    fmt.Println("ValueMethod called on", e.Value)
}

func (e *Example) PointerMethod() {
    fmt.Println("PointerMethod called on", e.Value)
}
```

#### 值接收者方法调用

```go
var ex Example
ex.ValueMethod()   // 直接调用，传递值
(&ex).ValueMethod() // 自动解引用 ex，传递值
```

#### 指针接收者方法调用

```go
var ex Example
ex.PointerMethod()  // 自动获取 ex 的地址，传递指针
(&ex).PointerMethod() // 直接传递指针
```

### 在代码中的应用


```go
type People struct{}

func (p *People) ShowA() {
    fmt.Println("showA")
    p.ShowB()
}

func (p *People) ShowB() {
    fmt.Println("showB")
}

type Teacher struct {
    People
}

func (t *Teacher) ShowB() {
    fmt.Println("teacher showB")
}

func main() {
    t := Teacher{}
    t.ShowB()
}
```

- `t := Teacher{}` 创建了一个 `Teacher` 类型的值。
- `t.ShowB()` 调用 `Teacher` 的 `ShowB()` 方法，尽管 `ShowB` 的接收者是 `*Teacher` 类型的指针，Go 自动获取 `t` 的地址并传递给方法，因此调用的是 `(&t).ShowB()`。

```go
type People struct{}

func (p People) ShowA() {
    fmt.Println("showA")
    p.ShowB()
}

func (p People) ShowB() {
    fmt.Println("showB")
}

type Teacher struct {
    People
}

func (t *Teacher) ShowB() {
    fmt.Println("teacher showB")
}

func main() {
    t := &Teacher{}
    t.ShowB()
}
```
- 这样也能正常输出

### 为什么可以这样？

这是因为 Go 编译器知道 `ShowB` 方法需要一个指针接收者，并且在 `t` 是一个值时，它会自动获取 `t` 的地址以符合方法的签名。这使得方法调用更加直观，无需显式地获取地址。

### 总结

- **值接收者方法**：可以使用值或者指针来调用。
- **指针接收者方法**：可以使用指针或者值来调用，Go 会自动获取地址。

这种自动转换规则使得方法调用变得更加简洁，不需要显式地在值和指针之间转换，从而减少了代码的复杂性和易错性。