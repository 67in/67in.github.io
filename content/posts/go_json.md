---
title: "Go语言json技巧"
date: 2022-10-31T18:09:06+08:00
draft: false
---

## 基本的序列化

首先我们来看一下Go语言中`json.Marshal`（序列化）与`json.Unmarshal`（反序列化）的基本用法。

- 忽略没有值的字段 omitempty

- 指定json序列化/反序列化时忽略此字段

```go
type Person struct {
	Name   string  `json:"name"`          // 指定json序列化/反序列化时使用小写name
	Age    int64   `json:"age,omitempty"` //如果想要在序列序列化时忽略这些没有值的字段时，可以在对应字段添加omitempty tag。
	Weight float64 `json:"-"`             // 指定json序列化/反序列化时忽略此字段
}

func main1() {
	p1 := Person{
		Name:   "67in",
		Age:    18,
		Weight: 50,
	}
	//struct -> json string
	b, err := json.Marshal(p1)
	if err != nil {
		fmt.Println("json.Marshal failed, err:%v\n", err)
		return
	}
	fmt.Println(b)
	fmt.Printf("str:%s\n", b)

	//json string -> struct
	var p2 Person
	err = json.Unmarshal(b, &p2)
	if err != nil {
		fmt.Printf("json.UnMarshal failed, err:%v\n", err)
		return
	}
	fmt.Println(p2)
	fmt.Printf("p2:%v\n", p2)
	fmt.Printf("p2:%#v\n", p2)
}
```

- **忽略嵌套结构体空值字段**

```go
type User struct {
	Name  string   `json:"name"`
	Email string   `json:",omitempty"`
	Hobby []string `json:"hobby,omitempty"`
	Profile
}

type Profile struct {
	Website string `json:"site"`
	Slogan  string `json:"slogan"`
}

func main2() {
	u1 := User1{
		Name:  "67in",
		Hobby: []string{"足球", "双色球"},
	}
	b, err := json.Marshal(u1)
	if err != nil {
		fmt.Printf("json.Marshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
}
//匿名嵌套Profile时序列化后的json串为单层的
//输出   str:{"name":"七米","hobby":["足球","双色球"],"site":"","slogan":""}

//想要变成嵌套的json串，需要改为具名嵌套或定义字段tag：
type User1 struct {
	Name    string   `json:"name"`
	Email   string   `json:"email,omitempty"`
	Hobby   []string `json:"hobby,omitempty"`
	Profile `json:"profile"`
}

//输出   str:{"name":"七米","hobby":["足球","双色球"],"profile":{"site":"","slogan":""}}

//想要在嵌套的结构体为空值时，忽略该字段，仅在profile字段添加omitempty是不够的
type User2 struct {
	Name    string   `json:"name"`
	Email   string   `json:"email,omitempty"`
	Hobby   []string `json:"hobby,omitempty"`
	Profile `json:"profile,omitempty"`
}

//输出  str:{"name":"七米","hobby":["足球","双色球"],"profile":{"site":"","slogan":""}}
//此时并没有忽略，需要使用嵌套的结构体指针
type User3 struct {
	Name     string   `json:"name"`
	Email    string   `json:"email,omitempty"`
	Hobby    []string `json:"hobby,omitempty"`
	*Profile `json:"profile,omitempty"`
}
```

- 不修改原结构体忽略某字段

> 需要json序列化`User`，但是不想把密码也序列化，又不想修改`User`结构体，这个时候我们就可以使用创建另外一个结构体`PublicUser`匿名嵌套原`User`，同时指定`Password`字段为匿名结构体指针类型，并添加`omitempty`tag，示例代码如下：

```go
type User4 struct {
	Name     string `json:"name"`
	Password string `json:"password"`
}
type PublicUser struct {
	*User4             //匿名嵌套，指针，可忽略空的Password字段
	Password *struct{} `json:"password,omitempty"` //指针，可忽略空的Password字段
}

//omitPasswordDemo
func main3() {
	u1 := User4{
		Name:     "67in",
		Password: "123345",
	}
	publicUser := PublicUser{
		User4: &u1,
	}
	fmt.Printf("publicUser: %v\n", publicUser.User4)
	fmt.Printf("publicUser: %v\n", publicUser.User4.Password)
	fmt.Printf("publicUser: %v\n", publicUser.Password)
	b, err := json.Marshal(publicUser)
	if err != nil {
		fmt.Printf("json.Marshal u1 failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b) //输出str:{"name":"67in"}
}
```

## 优雅处理字符串格式的数字

- json数据中string类型数字反序列化为数字类型

```go
type Card struct {
	ID    int64   `json:"id,string"`
	Score float64 `json:"score,string"`
}

//initAndStringDemo
func main4() {
	jsonStr1 := `{"id":"1234567","score":"88.50"}`
	var c1 Card
	if err := json.Unmarshal([]byte(jsonStr1), &c1); err != nil {
		fmt.Printf("json.Unmarshal jsonStr1 failed, err:%v\n", err)
		return
	}
	fmt.Printf("c1:%#v\n", c1)
}
```

- 整数变浮点数

> ```
> 在JSON协议中是没有整型和浮点型之分的，统称为number。
> Json字符串中的数字经过Go语言中的json包反序列化之后都会成为float64类型
> 通常这并不会有什么问题，但是在某些特殊场景下就会产生意想不到的结果。比如，将Json格式的数据
> 反序列化为map[string]interface{}时，数字都变成科学计数法表示的浮点数
> ```

```go
func main5() {
	type student struct {
		ID   int64  `json:"id"`
		Name string `json:"name"`
	}
	s := student{
		ID:   123456789,
		Name: "q1mi",
	}
	b, _ := json.Marshal(s)
	var m map[string]interface{}
	//decode
	json.Unmarshal(b, &m)
	fmt.Printf("id:%#v\n", m["id"])
	fmt.Printf("id type: %T\n", m["id"])

	//use Number decode
	decoder := json.NewDecoder(bytes.NewReader(b))
	decoder.UseNumber()
	decoder.Decode(&m)
	fmt.Printf("id:%#v\n", m["id"])
	fmt.Printf("id type:%T\n", m["id"])
}
//id:1.23456789e+08
//id type: float64
//id:"123456789"
//id type:json.Number

func main6() {
	//map[string]interface{} -> json string
	var m = make(map[string]interface{}, 1)
	m["count"] = 1 //int
	b, err := json.Marshal(m)
	if err != nil {
		fmt.Printf("marshal failed, err:%v\n", err)
	}
	fmt.Printf("str:%#v\n", string(b))
	//json string -> map[string]interface{}
	var m2 map[string]interface{}
	err = json.Unmarshal(b, &m2)
	if err != nil {
		fmt.Printf("unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("value:%v\n", m2["count"])
	fmt.Printf("value type:%T\n", m2["count"])
}
//str:"{\"count\":1}"
//value:1
//value type:float64
```

- 如果想更合理的处理数字就需要使用decoder去反序列化，示例代码如下

```go
func main7() {
	//map[string]interface{} -> json string
	var m = make(map[string]interface{}, 1)
	m["count"] = 1 //int
	b, err := json.Marshal(m)
	if err != nil {
		fmt.Printf("marshal failed, err:%v\n", err)
	}
	fmt.Printf("str:%#v\n", string(b))
	//json string -> map[string]interface{}
	var m2 map[string]interface{}
	//使用decoder方式反序列化，指定使用number类型
	decoder := json.NewDecoder(bytes.NewReader(b))
	decoder.UseNumber()
	err = decoder.Decode(&m2)
	if err != nil {
		fmt.Printf("unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("value:%v\n", m2["count"]) // 1
	fmt.Printf("type:%T\n", m2["count"])  // json.Number

	count, err := m2["count"].(json.Number).Int64()
	//将m2["count"]转为json.Number后调用Int64方法获得int64类型的值
	if err != nil {
		fmt.Printf("parse to int64 failed, err: %v\n", err)
		return
	}
	fmt.Printf("type:%T\n", int(count))
}
//str:"{\"count\":1}"
//value:1
//type:json.Number
//type:int
```

## 自定义解析时间字段

> 内置的json包不识别我们常用的字符串时间格式，如2022-11-03 10:00:00。不过我们通过实现 json.Marshaler/json.Unmarshaler 接口实现自定义的事件格式解析。

```go
type CustomTime time.Time

//首字母要大写
const CtLayout = "2006-01-02 15:04:05"

func (t *CustomTime) UnmarshalJSON(data []byte) (err error) {
	if string(data) == `""` {
		return
	}

	now, err := time.ParseInLocation(`"`+CtLayout+`"`, string(data), time.Local)
	*t = CustomTime(now)
	return
}

func (ct CustomTime) MarshalJSON() ([]byte, error) {
	b := make([]byte, 0, len(CtLayout)+2)
	b = append(b, '"')
	if !time.Time(ct).IsZero() || !time.Time(ct).Truncate(8*time.Hour).IsZero() {
		b = time.Time(ct).AppendFormat(b, CtLayout)
	}
	b = append(b, '"')
	return b, nil
}

type Post1 struct {
	CreateTime CustomTime `json:"create_time"`
}

func main9() {
	p1 := Post1{
		CreateTime: CustomTime(time.Now()),
	}
	b, err := json.Marshal(p1)
	if err != nil {
		fmt.Printf("json.Marshal p1 failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
	jsonStr := `{"create_time":"2022-11-03 11:14:06"}`
	var p3 Post1
	if err := json.Unmarshal([]byte(jsonStr), &p3); err != nil {
		fmt.Printf("json.Unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Println(p3.CreateTime)
	fmt.Printf("p3:%v\n", p3.CreateTime)
	fmt.Printf("p3:%#v\n", p3.CreateTime)
}
```

## 使用匿名结构体添加字段

> 使用内嵌结构体能够扩展结构体的字段，但有时候我们没有必要单独定义新的结构体，可以使用匿名结构体简化操作

```go
type UserInfo struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
}

func main11() {
	u1 := UserInfo{
		ID:   123456,
		Name: "七米",
	}
	// 使用匿名结构体内嵌User并添加额外字段Token
	b, err := json.Marshal(struct {
		*UserInfo
		Token string `json:"token"`
	}{
		&u1,
		"91je3a4s72d1da96h",
	})
	if err != nil {
		fmt.Printf("json.Marsha failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
	// str:{"id":123456,"name":"七米","token":"91je3a4s72d1da96h"}
}
```

## 使用匿名结构体组合多个结构体

```go
type Comment struct {
	Content string
}

type Image struct {
	Title string `json:"title"`
	URL   string `json:"url"`
}

func main12() {
	c1 := Comment{
		Content: "永远不要高估自己",
	}
	i1 := Image{
		Title: "赞赏码",
		URL:   "https://www.liwenzhou.com/images/zanshang_qr.jpg",
	}
	// struct -> json string
	b, err := json.Marshal(struct {
		*Comment
		*Image
	}{&c1, &i1})
	if err != nil {
		fmt.Printf("json.Marshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("str:%s\n", b)
	// json string -> struct
	jsonStr := `{"Content":"永远不要高估自己","title":"赞赏码","url":"https://www.liwenzhou.com/images/zanshang_qr.jpg"}`
	var (
		c2 Comment
		i2 Image
	)
	if err := json.Unmarshal([]byte(jsonStr), &struct {
		*Comment
		*Image
	}{&c2, &i2}); err != nil {
		fmt.Printf("json.Unmarshal failed, err:%v\n", err)
		return
	}
	fmt.Printf("c2:%#v i2:%#v\n", c2, i2)
}
//str:{"Content":"永远不要高估自己","title":"赞赏码","url":"https://www.liwenzhou.com/images/zanshang_qr.jpg"}
//c2:main.Comment{Content:"永远不要高估自己"} i2:main.Image{Title:"赞赏码", URL:"https://www.liwenzhou.com/images/zanshang_qr.jpg"}
```

## json.RawMessage 处理不确定层级的json

> 如果json串没有固定的格式导致不好定义与其相对应的结构体时，我们可以使用json.RawMessage原始字节数据保存下来

```go
type sendMsg struct {
	User string `json:"user"`
	Msg  string `json:"msg"`
}

func main13() {
	jsonStr := `{"sendMsg":{"user":"q1mi","msg":"永远不要高估自己"},"say":"Hello"}`
	//定义一个map，value类型为json.RawMessage
	var data map[string]json.RawMessage
	if err := json.Unmarshal([]byte(jsonStr), &data); err != nil {
		fmt.Println(err)
		return
	}
	var msg sendMsg
	if err := json.Unmarshal(data["sendMsg"], &msg); err != nil {
		fmt.Println(err)
		return
	}
	fmt.Println(msg)
}
```

