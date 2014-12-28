---
layout: post
category: "language"
title:  "Go语言的Gob编码机制应用"
tags: [Go]
---

* Go语言中的Gob编码是自描述（self-describing）的，即不需要预先定义类似ProtocolBuffer之类的协议文件
* Gob包的关键方法有：
	* func NewEncoder(w io.Writer) *Encoder	//基于io.Writer生成Encoder，io.Writer可以是bytes.Buffer等
	* func NewDecoder(r io.Reader) *Decoder	//同上
	* func Register(value interface{})			//如果要编解码的类型中存在interface{}，则需要预先注册实际传输的类型
	* func (enc *Encoder) Encode(e interface{}) error	//将对象进行编码，并放入io.Writer中
	* func (dec *Decoder) Decode(e interface{}) error	//同上
* Gob的类图层次：![Gob Encoding](https://raw.githubusercontent.com/joezxy/joezxy.github.io/master/_img/20141227_gob_encoding.png)
* 如果想要自定义对象的序列化方式，可以实现接口BinaryMarshaler的MarshalBinary()和BinaryUnMarshaler的UnMarshalBinary()

示例代码：

```Go
package main

import (
	"bytes"
	"encoding/gob"
	"fmt"
	"log"
)

type P struct {
	X, Y  int
	Fname string
	Sname interface{}
}

type Q struct {
	X     int
	Fname string
	Sname interface{}
}

type C struct {
	I int
}

func main() {
	var buf bytes.Buffer
	
	gob.Register(C{}) //Register for interface{} type
	
	enc := gob.NewEncoder(&buf)
	dec := gob.NewDecoder(&buf)
	
	err := enc.Encode(P{1, 2, "p1", C{4}})
	if err != nil {
		log.Fatal("encoding error: ", err)
	}
	err = enc.Encode(P{10001, 10002, "p2", C{101}})
	if err != nil {
		log.Fatal("encoding error: ", err)
	}

	var q Q
	err = dec.Decode(&q)
	fmt.Println(q.X, q.Fname, q.Sname.(C).I)
	err = dec.Decode(&q)
	fmt.Println(q.X, q.Fname, q.Sname)
}
```

运行结果为：
> 1 p1 4
> 10001 p2 {101}

