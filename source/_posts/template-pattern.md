---
title: 模版模式
date: 2019-11-04 19:00:33
tags: [go, 设计模式]
categories: 设计模式
---

[go] Template Pattern

<!-- more --> 

# 简介
Template Pattern（模板模式）

> 白话文

定一个“抽象类”，定义一个方法A，定义需要子类实现的方法，所有子类对象在执行A的时候，会调用各自实现的方法。

在golang中，由于不存在抽象类和真正的继承，所以只能通过一个基础类来充当抽象类，子类通过组合基础类来实现通用方法的继承

> 故事

阳光明媚的一天，我家来了位香港的朋友（毕竟那边太乱），我们决定一起做一桌菜，于是他做香港菜，我做杭州菜，比拼就这么开始了

## 实际代码例子

### 普通例子

> 代码

```go
package main

import (
	"fmt"
	"testing"
)

type Cooking interface {
	DoOperate()
} 

type AbstractCooking struct {
	Cooking
	Prepare    func()
	GetContent func() string
}

func (d AbstractCooking) DoOperate() {
	d.Prepare()
	fmt.Println("烹饪内容:", d.GetContent())
	fmt.Println("烹饪完成")
}

type HZCooking struct {
	AbstractCooking
}

func NewHZCooking() *HZCooking {
	c := new(HZCooking)
	c.AbstractCooking.GetContent = c.GetContent
	c.AbstractCooking.Prepare = c.Prepare
	return c
}

func (c *HZCooking) GetContent() string {
	return "杭州菜."
}

func (c *HZCooking) Prepare() {
	fmt.Println(" -- 准备杭州菜 -- ")
}

type HkCooking struct {
	HZCooking
}

func NewHKCooking() *HkCooking {
	c := new(HkCooking)
	c.AbstractCooking.GetContent = c.GetContent
	c.AbstractCooking.Prepare = c.Prepare
	return c
}

func (c *HkCooking) GetContent() string {
	return "香港菜."
}

func (c *HkCooking) Prepare() {
	fmt.Println(" -- 准备香港菜 -- ")
}

func TestCooking(t *testing.T) {
	chinaCooking := NewHZCooking()

	chinaCooking.DoOperate()

	hkCooking := NewHKCooking()

	hkCooking.DoOperate()
}

```

> 输出结果

```text
 -- 准备杭州菜 -- 
烹饪内容: 杭州菜.
烹饪完成
 -- 准备香港菜 -- 
烹饪内容: 香港菜.
烹饪完成
```

# 参考
https://www.tutorialspoint.com/design_pattern/template_pattern.htm