---
title: canvas 实现手写签名中遇到的绘制偏移问题
date: 2021-03-15 14:23:22
tags:tips
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

