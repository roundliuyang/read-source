# Golang: 解析JSON数据

摘自：[Golang: 解析JSON数据之三](https://www.cnblogs.com/liuhe688/p/11105571.html)

前面我们介绍了 Marshal 和 Unmarshal 方法，今天再解一下另外两个 API：Encoder 和 Decoder。

## Encoder

Encoder 主要负责将结构对象编码成 JSON 数据，我们可以调用 json.NewEncoder(io.Writer) 方法获得一个 Encoder 实例：

```go
// NewEncoder returns a new encoder that writes to w.
func NewEncoder(w io.Writer) *Encoder {
    return &Encoder{w: w, escapeHTML: true}
}
```

io.Writer 是接口类型，包含一个 Write(p []byte) 方法，凡是实现了这个方法的类型实例，都可以作为参数传递进去。

接下来我们直接调用 Encoder 实例的 Encode(interface{}) 方法即可完成编码操作：

```go
func (enc *Encoder) Encode(v interface{}) error {
    // ...
}
```

下面，我们以一个实例来演示这个操作：

```go
package main

import (
    "encoding/json"
    "os"
)

type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
    // 如果Child字段为nil 编码JSON时可忽略
    Child *Person `json:"child,omitempty"`
}

func main() {
    person := Person{
        Name: "John",
        Age: 40,
        Child: &Person{
            Name: "Jack",
            Age: 20,
        },
    }

    // File类型实现了io.Writer接口
    file, _ := os.Create("person.json")

    // 根据io.Writer创建Encoder 然后调用Encode()方法将对象编码成JSON
    json.NewEncoder(file).Encode(&person)
}
```

上面程序会将结构体对象编码成 JSON 数据，存入 person.json 文件中，程序运行后，会生成下面文件内容：

```json
{"name":"John","age":40,"child":{"name":"Jack","age":20}}
```

## Decoder

Decoder 主要负责将 JSON 数据解析成结构对象，我们可以调用 json.NewDecoder(io.Reader) 方法获得一个 Decoder 实例：

```go
// NewDecoder returns a new decoder that reads from r.
func NewDecoder(r io.Reader) *Decoder {
    return &Decoder{r: r}
}
```

同样地，io.Reader 也是接口类型，包含一个 Read(p []byte) 方法，凡是实现了这个方法的类型实例，都可以作为参数传递进去。

获取到 Decoder 实例之后，我们直接调用它的 Decode(interface{}) 方法即可完成解析操作：

下面我们写一段程序，读取 person.json 文件，将文件中的 JOSN 内容解析为对象类型：

```go
package main

import (
    "encoding/json"
    "fmt"
    "os"
)

type Person struct {
    Name string `json:"name"`
    Age  int    `json:"age"`
    // 如果Child字段为nil 编码JSON时可忽略
    Child *Person `json:"child,omitempty"`
}

func main() {
    var person Person

    // File类型也实现了io.Reader接口
    file, _ := os.Open("person.json")

    // 根据io.Reader创建Decoder 然后调用Decode()方法将JSON解析成对象
    json.NewDecoder(file).Decode(&person)

    fmt.Println(person)
    fmt.Println(*person.Child)
}
```

运行上面程序，控制台打印如下：

```bash
{John 40 0xc42000a080}
{Jack 20 <nil>}
```