---
title: 装饰模式
date: 2019-11-04 17:00:24
tags: [go, 设计模式]
categories: 设计模式
---

[go] Decorator Pattern

<!-- more --> 

# 简介
Decorator Pattern（装饰模式）

> This pattern creates a decorator class which wraps the original class and provides additional functionality keeping class methods signature intact.

> 白话文

这个模式我们需要创建一个新的装饰类，然后装饰类会拥有一个属性是被装饰类，并且会有“一个”方法和需要使用的被装饰类的方法签名完全一致，调用装饰类的方法会执行装饰类内容并调用被装饰类的被装饰方法。

> 故事

我买了一本书《go编程思想》，小明也买了一本书《go编程思想》并给它带上了一个黄金封面，两本书内容一摸一样，只是小明的变成金灿灿的土豪版。

## 实际代码例子

### 普通例子

对于某个接口的方法装饰

> 代码

```go
package decorator

import (
	"fmt"
	"testing"
)

type Shape interface {
	draw()
}

type Circle struct {
}

func (shape Circle) draw() {
	fmt.Println("Shape: Circle")
}

type Rectangle struct {
}

func (shape Rectangle) draw() {
	fmt.Println("Shape: Rectangle")
}

type ShapeDecorator struct {
	decoratorShape Shape
}

func (shape ShapeDecorator) draw() {
	shape.decoratorShape.draw()
}

type RedShapeDecorator struct {
	ShapeDecorator
}

func NewRedShapeDecorator(s Shape) *RedShapeDecorator {
	d := new(RedShapeDecorator)
	d.decoratorShape = s
	return d
}

func (shape RedShapeDecorator) draw() {
	shape.ShapeDecorator.draw()
	fmt.Println("red")
}

type BlueShapeDecorator struct {
	ShapeDecorator
}

func NewBlueShapeDecorator(s Shape) *BlueShapeDecorator {
	d := new(BlueShapeDecorator)
	d.decoratorShape = s
	return d
}

func (shape BlueShapeDecorator) draw() {
	shape.ShapeDecorator.draw()
	fmt.Println("blue")
}

func TestName(t *testing.T) {
	redShapedDecorator := NewRedShapeDecorator(Circle{})
	redShapedDecorator.draw()

	redShapedDecorator = NewRedShapeDecorator(Rectangle{})
	redShapedDecorator.draw()

	blueShapedDecorator := NewBlueShapeDecorator(Circle{})
	blueShapedDecorator.draw()

	blueShapedDecorator = NewBlueShapeDecorator(Rectangle{})
	blueShapedDecorator.draw()
}
```

> 输出结果

```text
Shape: Circle
red
Shape: Rectangle
red
Shape: Circle
blue
Shape: Rectangle
blue
```

### 直接装饰方法

> 代码如下

```go
type drawFunc func()

func CircleDraw() {
	fmt.Println("Shape: Circle")
}

func ReadCircleDraw(d drawFunc) drawFunc {
	return func() {
		d()
		fmt.Println("red")
	}
}

func TestFunc(t *testing.T) {
	var drawFunc drawFunc
	drawFunc = CircleDraw
	drawFunc()

	drawFunc = ReadCircleDraw(drawFunc)
	drawFunc()
}
```

> 输出结果

```text
Shape: Circle
Shape: Circle
red
```

# 参考文章
https://www.tutorialspoint.com/design_pattern/decorator_pattern.htm