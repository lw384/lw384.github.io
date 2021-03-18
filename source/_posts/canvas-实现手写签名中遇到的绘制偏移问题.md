---
title: canvas 实现手写签名中遇到的绘制偏移问题
date: 2021-03-15 14:23:22
tags: tips
---

#  canvas 实现手写签名中遇到的绘制偏移问题

H5实现手写签名已有非常多的网上代码不多追述。

```javascript
// HTML  
// vue
<template>
<!--  手写签名-->
  <div>
    <canvas id="canvas" width="320" height="150" ref="canvas"></canvas>
    <div class="signature-btn">
      <button class="clear-btn" @click="toClear()">清除</button>
      <button  class="save-btn" @click="toSave()">保存</button>
    </div>
  </div>
</template>
```

```javascript
// script

 mounted () {
    this.initPage()
  },
  methods: {
    initPage () {
      this.canvas = this.$refs.canvas
      if (this.canvas.getContext) {
        this.ctx = this.canvas.getContext('2d')
        let background = '#ffffff'
        this.ctx.lineCap = 'round'
        this.ctx.fillStyle = background
        this.ctx.lineWidth = 2
        this.ctx.fillRect(0, 0, 350, 150)
        this.canvas.addEventListener('touchstart', (e) => {
          this.ctx.beginPath()
          this.ctx.moveTo(e.changedTouches[0].pageX, e.changedTouches[0].pageY)
        })

        this.canvas.addEventListener('touchmove', (e) => {
          e.preventDefault()
          this.ctx.lineTo(e.changedTouches[0].pageX, e.changedTouches[0].pageY)
          this.ctx.stroke()
        }, { passive: false })
      }
    },
    toClear () {
      this.ctx.clearRect(0, 0, 320, 150)
    },
    toSave () {
      let base64Img = this.canvas.toDataURL()
      this.$emit('toSaveSignature', base64Img)
    }
  }
```

## 问题：canves无位移则正常绘制，放置于页面中部则绘制偏差极大，画布无结果

初始以为是鼠标点击坐标 - 画布距离页面左上角的偏移 = 实际的绘制点

但一旦滚动页面，偏移还是会出现。

## 知识点

![引用示意图](http://unclealan.cn/usr/uploads/2017/11/228643359.png)

坐标系左上角为原点，X轴向右，Y轴向下

### screen

- 参照点：电脑屏幕左上角
- screenX：鼠标点击位置相对于电脑屏幕左上角的水平偏移量
- screenY：鼠标点击位置相对于电脑屏幕左上角的垂直偏移量

### client

- 参照点：浏览器内容区域左上角
- clientX：鼠标点击位置相对于浏览器可视区域的水平偏移量（不会计算水平滚动的距离）
- clientY：鼠标点击位置相对于浏览器可视区域的垂直偏移量（不会计算垂直滚动条的距离）

### page

- 参照点：网页的左上角
- pageX：鼠标点击位置相对于网页左上角的水平偏移量，也就是clientX加上水平滚动条的距离
- pageY：鼠标点击位置相对于网页左上角的垂直平偏移量，也就是clientY加上垂直滚动条的距离

### offset

（触发事件的元素,ie,chrome支持此属性，ff不支持）

* offsetX：设置或获取鼠标指针位置相对于触发事件的对象的 x 坐标。 
* offsetY ：设置或获取鼠标指针位置相对于触发事件的对象的 y 坐标。



## 解决

鼠标事件在canvas画布上画画，使用（e.offsetX，e.offsetY）非常容易就能获取到画布上的坐标。

但是，移动端的触摸事件就不一样了，它并没有提供这两个便捷的参数，必须去计算画布上的坐标。
原理：【页面上的坐标】-【画布左上角的坐标】=【画布上的坐标】

获取画布坐标方法
`let vertex = document.getElementById('canvas').getBoundingClientRect();`

因为移动端页面没有水平的偏移pageX = clientX，但是有竖直的滚动条，修改为，因为vertx的值是在绘画过程中实时触发的，所以在触发事件中读取一次

```javascript
this.canvas.addEventListener('touchstart', (e) => {
          let vertex = this.canvas.getBoundingClientRect()
          this.ctx.beginPath()
          this.ctx.moveTo(e.changedTouches[0].pageX - vertex.left, e.changedTouches[0].clientY - vertex.top)
        })

        this.canvas.addEventListener('touchmove', (e) => {
          let vertex = this.canvas.getBoundingClientRect()
          e.preventDefault()
          this.ctx.lineTo(e.changedTouches[0].pageX - vertex.left, e.changedTouches[0].clientY - vertex.top)
          this.ctx.stroke()
        }, { passive: false })
```

为了不使绘画过程造成页面滚动影响绘制，添加组织默认动作` e.preventDefault()`

