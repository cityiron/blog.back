---
title: 图片添加水印
date: 2019-10-16 16:07:54
tags: [go, 水印, image]
categories: go
copyright: true
---

[go] WaterMark, base on freetype, image.draw 

<!-- more --> 

既然开始写博客，就需要给图片加水印，之前用Java写过一个已经找不到影踪，只记得Java写的还比较麻烦。最近又在重温go的内容，就顺便用go写了下图片加水印的代码

# 输出文字图片

## 效果图：

![效果图：](https://res.cloudinary.com/dogbaobao/image/upload/v1571535791/blog/watermask/wattermark-1-wm-1571388899_hojx8n.png)

## 代码

依赖 ` github.com/golang/freetype ` 仓库

```go
type WaterMark struct {
    // freetype的参数
	width    int     // 图片的大小 宽度
	height   int     // 图片的大小 高度
	fontSize float64 // 字体尺寸
	fontDPI  float64 //   屏幕每英寸的分辨率
	fontFile string  // 需要使用的字体文件

	srcPath string // 原文件
}
```

```go
func (wm *WaterMark) CreateStringPic(srcPath, content string) {
	// 新建一个 指定大小的RGBA位图
	img := image.NewNRGBA(image.Rect(0, 0, wm.width, wm.height))

	// 画背景
	//for y := 0; y < wm.height; y++ {
	//	for x := 0; x < wm.width; x++ {
	//		// 设置某个点的颜色，依次是 RGBA，变化的
	//		img.Set(x, y, color.RGBA{255, 255, 255, 255})
	//	}
	//}

	// 读字体数据
	fontBytes, err := ioutil.ReadFile(wm.fontFile)

	if err != nil {
		log.Println(err)
		return
	}

	font, err := freetype.ParseFont(fontBytes)

	if err != nil {
		log.Println(err)
		return
	}

	c := freetype.NewContext()
	c.SetDPI(wm.fontDPI)
	c.SetFont(font)
	c.SetFontSize(wm.fontSize)
	c.SetClip(img.Bounds())
	c.SetDst(img)
	c.SetSrc(image.Black)

	pt := freetype.Pt(15, 30+int(c.PointToFixed(wm.fontSize)>>8)) // 字出现的位置

	_, err = c.DrawString(content, pt)
	if err != nil {
		log.Println(err)
		return
	}

	create(makePath(srcPath), img)
}
```

上面例子用到了 freetype 包，输出图片的文字内容是我的博客地址，你可以替换成自己想要的内容。

官方字体格式的说明如下：

> Drawing Font Glyphs
To draw a font glyph in blue starting from a point p, draw with an image.ColorImage source and an image.Alpha mask. For simplicity, we aren't performing any sub-pixel positioning or rendering, or correcting for a font's height above a baseline.

结合下面的实现代码：

```go
func (c *Context) DrawString(s string, p fixed.Point26_6) (fixed.Point26_6, error) {
	// ......
	for _, rune := range s {
		// ......
		advanceWidth, mask, offset, err := c.glyph(index, p) // 最后会进入下个方法
		if err != nil {
			return fixed.Point26_6{}, err
		}
		p.X += advanceWidth
		glyphRect := mask.Bounds().Add(offset)
		dr := c.clip.Intersect(glyphRect)
		if !dr.Empty() {
			mp := image.Point{0, dr.Min.Y - glyphRect.Min.Y}
			draw.DrawMask(c.dst, dr, c.src, image.ZP, mask, mp, draw.Over)
		}
		prev, hasPrev = index, true
	}
	return p, nil
}

func (c *Context) rasterize(glyph truetype.Index, fx, fy fixed.Int26_6) (
	fixed.Int26_6, *image.Alpha, image.Point, error) {

	// ......
	a := image.NewAlpha(image.Rect(0, 0, xmax-xmin, ymax-ymin))
	c.r.Rasterize(raster.NewAlphaSrcPainter(a))
	return c.glyphBuf.AdvanceWidth, a, image.Point{xmin, ymin}, nil
}
```

可以看到用到的是 ` image.NewAlpha ` 和 ` draw.DrawMask() `，和官方的说明一致，具体详细内容在结尾有网址查看



# 输出图片水印

## 原图：

![原图](https://res.cloudinary.com/dogbaobao/image/upload/v1571535804/blog/watermask/watermark-2_yyj8ak.jpg)

## 效果图：

![效果图：](https://res.cloudinary.com/dogbaobao/image/upload/v1571535802/blog/watermask/watermark-2-wm-1571391482_kudhos.jpg)

## 代码

```go
func (wm *WaterMark) AddImage(wmPath string) {

	srcImg := getImage(wm.srcPath)

	b := srcImg.Bounds()

	dst := image.NewNRGBA(b)

	// 绘入原始图片
	draw.Draw(dst, b, srcImg, image.ZP, draw.Src)

	waterMark := getImage(wmPath)

	maxW := b.Max.X / 5
	maxH := b.Max.Y / 4

    // 会出现多个水印，offset位置信息
	for offsetWidth := 0; offsetWidth < b.Max.X; offsetWidth += maxW {
		for offsetHeight := 0; offsetHeight < b.Max.X; offsetHeight += maxH {
			offset := image.Pt(offsetWidth, offsetHeight)
			draw.Draw(dst, waterMark.Bounds().Add(offset), waterMark, image.ZP, draw.Over)
		}
	}

    // 固定位置
	// offset := image.Pt((b.Bounds().Max.X-waterMark.Bounds().Max.X)/2, (b.Bounds().Max.Y-waterMark.Bounds().Max.Y)/2)
	// draw.Draw(dst, waterMark.Bounds().Add(offset), waterMark, image.ZP, draw.Over)

	create(makePath(wm.srcPath), dst)
}
```

图片加水印用到的是 draw 包的内容，相对比较简单，只需要修改中间那部分 ` image.Pt() ` 和 ` draw.Draw() ` 就可以调整位置

# 缩放图片

缩放图片参考的是 ` github.com/nfnt/resize ` 包，比较简单，这里就不上效果了。

## 代码
```go
func (wm *WaterMark) Resize() {

	src := getImage(wm.srcPath)

	b := src.Bounds()
	width := b.Max.X
	height := b.Max.Y

	w, h := calculateRatioFit(width, height)
	// 调用resize库进行图片缩放
	dst := resize.Resize(uint(w), uint(h), src, resize.Lanczos3)

	create(makePath(wm.srcPath), dst)
}
```

# 其它
## 补充代码

```go
func getImage(s string) (image.Image) {

	// 水印图片
	f, err := os.Open(s)
	if err != nil {
		fmt.Println("打开水印图片异常")
		return nil
	}

	defer f.Close()

	suffix := path.Ext(s)

	var img image.Image

	switch suffix {
	case ".jpeg", ".jpg":
		fmt.Println("这是JPG文件")
		img, err = jpeg.Decode(f)
		if err != nil {
			fmt.Println("解码jpeg图片异常")
		}
		break
	case ".png":
		fmt.Println("这是PNG文件")
		img, err = png.Decode(f)
		if err != nil {
			fmt.Println("解码jpeg图片异常")
		}
		break
	}

	return img
}

func create(target string, m image.Image) {
	imgNew, err := os.Create(target)
	if err != nil {
		log.Println(err)
	}

	defer imgNew.Close()

	suffix := path.Ext(target)

	switch suffix {
	case ".jpeg", ".jpg":
		err = jpeg.Encode(imgNew, m, &jpeg.Options{100})
		break
	case ".png":
		err = png.Encode(imgNew, m)
		break
	}

	if err != nil {
		log.Println(err)
	}
}
```

## 扩展&问题

> 直接给图片添加文字的采坑过程

我在freetype绘入文字之前把背景图绘入，会导致生成的图片直接无法打开，导致没法一步到位，所以现在是通过给图片加图片，直接水印的方式做的。
这个问题在freetype例子里面找到了解决，它是用到了 ` golang.org/x/image/math/fixed `

```go
func (d *Drawer) DrawString(s string) {
	prevC := rune(-1)
	for _, c := range s {
		if prevC >= 0 {
			d.Dot.X += d.Face.Kern(prevC, c)
		}
		dr, mask, maskp, advance, ok := d.Face.Glyph(d.Dot, c)
		if !ok {
			// TODO: is falling back on the U+FFFD glyph the responsibility of
			// the Drawer or the Face?
			// TODO: set prevC = '\ufffd'?
			continue
		}
		draw.DrawMask(d.Dst, dr, d.Src, image.Point{}, mask, maskp, draw.Over)
		d.Dot.X += advance
		prevC = c
	}
}
```

![效果图：](https://res.cloudinary.com/dogbaobao/image/upload/v1571534711/blog/watermask/watermark-3-wm-1571395380_pmwbo2.jpg)

参考地址如下：
https://github.com/golang/freetype/tree/master/example/drawer

> 批量水印

参考单个水印，构建的 struct 对象新增字段 []string 来接受需要水印的文件地址数组，或者使用文件夹，遍历文件夹下的所有文件统一加水印

# 参考内容
> 水印
http://www.nljb.net/default/Golang-绘图技术-image-draw-包介绍/
https://blog.golang.org/go-imagedraw-package
https://golang.org/doc/progs/image_draw.go
https://studygolang.com/articles/12049
> 文字
https://github.com/golang/freetype