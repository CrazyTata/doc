## Go语言中JSON标签的用法与技巧

在Go语言中，JSON标签（JSON tags）是用来指定结构体字段在序列化为JSON时的名称和行为的。JSON标签通常写在结构体字段的后面，用反引号（`）括起来。以下是一些常用的JSON标签：

1. `json:"field_name"`：指定JSON对象中的字段名。例如：

```go
type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
}
```

2. `omitempty`：当字段值为空（如零值、空字符串、nil等）时，忽略该字段。例如：

```go
type Person struct {
    Name string `json:"name,omitempty"`
    Age  int    `json:"age,omitempty"`
}
```

3. `-`：忽略该字段，不将其序列化到JSON中。例如：

```go
type Person struct {
    Name string `json:"name"`
    Age  int    `json:"-"`
}
```

4. `json:",inline"`：将结构体字段内联到父结构体中，即不生成嵌套的JSON对象。例如：

```go
type Address struct {
    City string `json:"city"`
    State string `json:"state"`
}

type Person struct {
    Name    string  `json:"name"`
    Age     int     `json:"age"`
    Address `json:",inline"`
}
```

5. `json:",string"`：将字段值序列化为JSON字符串。例如：

```go
type Person struct {
    Name string `json:"name"`
    Age  int    `json:",string"`
}
```

6. `json:",number"`：将字段值序列化为JSON数字。这个标签通常用于`float64`和`int64`类型的字段。例如：

```go
type Product struct {
    Name  string  `json:"name"`
    Price float64 `json:",number"`
}
```

7. `json:",bool"`：将字段值序列化为JSON布尔值。这个标签通常用于`bool`类型的字段，但在Go中，默认情况下布尔值已经是布尔类型，所以这个标签很少使用。

注意：JSON标签是大小写不敏感的，但在Go中，结构体字段名是大小写敏感的。因此，在使用JSON标签时，请确保它们与结构体字段名匹配。

这些标签可以组合使用，例如：

```go
type Person struct {
    Name    string  `json:"name"`
    Age     int     `json:"age,omitempty,string"`
    Address *Address `json:"address,omitempty"`
}
```