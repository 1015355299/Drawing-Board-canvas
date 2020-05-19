# Drawing-Board-canvas-
canvas原生js实现简单画板

<!DOCTYPE html>
<html lang="zh-CN">

<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
  <style>
    * {
      border: 0;
      margin: 0;
      user-select: none;
    }

    html:root {
      font-size: 16px;
    }

    #draw {
      position: relative;
      font-size: .625rem;
      width: 50rem;
      height: 25rem;
      background-color: rgb(82, 231, 12);
    }

    #tools {
      text-align: center;
      position: absolute;
      width: 6.25rem;
      height: 100%;
    }

    #header {
      background-color: rgb(55, 205, 224);
      height: 52%;
    }

    #footer {
      position: absolute;
      display: flex;
      align-items: center;
      justify-content: center;
      flex-direction: column;
      flex-wrap: wrap;
      top: 15.625rem;
      width: 6.25rem;
      height: 35%;
      background-color: rgb(184, 72, 20);
    }

    .btn {
      margin: .3rem;
      box-shadow: 0 0 3px 1px inset;
    }

    #canvas {
      position: relative;
      margin-left: 6.25rem;
      box-shadow: 0 0 5px 5px inset;

    }

    input {
      margin: .3125rem;
    }

    .colors {
      width: 40%;
      height: 1.25rem;
    }

    .ranges {
      width: 90%;
    }

    .selected {
      background-color: rgb(166, 255, 0);
    }

    #clear:active,
    #back:active,
    #restore:active {
      background-color: rgb(166, 255, 0);
    }
  </style>
</head>

<body>
  <div id="draw">
    <div id="tools">
      <h2>工具栏</h2>
      <div id="header">
        <input id="color" type="color" class="colors" value="#000000">画笔颜色</input>
        <input id="bgc" type="color" class="colors" value="#ffffff">画板颜色</input>
        <input id="lineWidthSmall" class="ranges" type="range" value="0.01" min="0.002" max="0.02"
          step="0.001">画笔微调~<span id="text1"></span></input>
        <input id="lineWidthBig" class="ranges" type="range" value="2" min="0" max="10" step="0.01">画笔粗调~<span
          id="text2"></span></input>
        <input id="shadowBlur" class="ranges" type="range" value="1" min="0" max="10" step="0.01">画笔虚化~<span
          id="text3"></span></input>
      </div>
      <div id="footer">
        <button id="pen" class="btn selected">画笔</button>
        <button id="line" class="btn">直线</button>
        <button id="rect" class="btn">矩形</button>
        <button id="cir" class="btn">圆形</button>
        <button id="eraser" class="btn">橡皮</button>
        <button id="clear" class="btn">清空</button>
        <button id="back" class="btn">撤销</button>
        <button id="restore" class="btn">恢复</button>
      </div>
    </div>
    <canvas id="canvas" width="700" height="400"></canvas>
  </div>

  <script>
    // 获取画布
    const canvas = document.querySelector('#canvas')
    // 获取上下文
    const ctx = canvas.getContext('2d')

    /* ------------------------------定义全局变量---------------------------*/
    // 定义鼠标按下
    let isPressDown = false
    // 当前模式
    let curMode = 'penMode'
    // 记录按下坐标
    let pressPos = null
    // 记录当前坐标
    let curPos = null
    // 记录当前画板图像数据
    let startData = null

    /* ------------------------------定义函数---------------------------*/
    // 获取鼠标相对于画板的坐标
    const getMousePos = function (clientX, clientY) {
      // 获取元素相对于视口位置信息
      const rect = canvas.getBoundingClientRect()
      // 返回鼠标相对于画布的坐标
      return {
        x: clientX - rect.left,
        y: clientY - rect.top
      }
    }
    // 高亮选中
    const selected = function (ele) {
      // 获取全部模式按钮
      const buttons = document.querySelectorAll('.btn')
      // 排他
      for (let i = 0; i < buttons.length; i++) {
        buttons[i].className = 'btn'
      }
      // 给选中按钮高亮
      ele.className = 'btn selected'
    }
    // 前后退操作
    const backOrRestore = (function () {
      // 图像处理对象
      const imageOperate = {}
      // 记录每次操作后的画板图像数据集合
      imageOperate.imageDataArray = []
      // 记录当前渲染图像数据
      imageOperate.curImageIndex = -1
      // 图像保存
      imageOperate.save = function (data) {
        if (this.imageDataArray.length >= 20) this.imageDataArray.shift() // 删除最久一次记录
        this.imageDataArray.push(data) // 记录
        this.curImageIndex = this.imageDataArray.length - 1
      }
      // 图像撤销
      imageOperate.back = function () {
        this.curImageIndex--
        if (this.curImageIndex <= 0) {
          this.curImageIndex = 0
        }
        paint('drawMode', this.imageDataArray[this.curImageIndex])

      }
      // 图像恢复
      imageOperate.restore = function () {
        this.curImageIndex++
        if (this.curImageIndex >= this.imageDataArray.length) {
          this.curImageIndex = this.imageDataArray.length - 1
        }
        paint('drawMode', this.imageDataArray[this.curImageIndex])
      }
      // 清空记录
      imageOperate.clear = function () {
        this.imageDataArray.length = 0
        this.curImageIndex = -1
      }
      // 前后退执行函数
      return function (operate, imageData) {
        imageOperate[operate](imageData)
      }
    })()
    /* -----------------各类模式----------------*/
    // 绘制
    const paint = (function () {
      const Mode = {}
      // 画笔模式
      Mode.penMode = function () {
        // 画线
        // ctx.moveTo(pressPos.x, pressPos.y)
        ctx.lineTo(curPos.x, curPos.y)
        ctx.stroke()
      }
      // 直线模式
      Mode.lineMode = function () {
        // 画线
        ctx.moveTo(pressPos.x, pressPos.y)
        ctx.lineTo(curPos.x, curPos.y)
        ctx.stroke()
      }
      // 矩形模式
      Mode.rectMode = function () {
        // 画矩形
        ctx.rect(pressPos.x, pressPos.y, curPos.x - pressPos.x, curPos.y - pressPos.y)
        ctx.stroke()
      }
      // 圆形模式
      Mode.cirMode = function () {
        const cx = (curPos.x + pressPos.x) / 2 // 圆心x
        const cy = (curPos.y + pressPos.y) / 2 // 圆心y
        const dx = Math.abs(curPos.x - pressPos.x) // x偏移
        const dy = Math.abs(curPos.y - pressPos.y) // y偏移
        const r = Math.sqrt(Math.pow(dx, 2) + Math.pow(dy, 2)) / 2 // 半径
        // 画圆形
        ctx.arc(cx, cy, r, 0, 2 * Math.PI)
        ctx.stroke()
      }
      // 橡皮模式
      Mode.eraserMode = function () {
        // 保存当前环境
        ctx.save()
        // 重置当前路径
        ctx.beginPath()
        // 画圆形
        ctx.arc(curPos.x, curPos.y, lineWidthSmall.value * 1 + lineWidthBig.value * 1, 0, 2 * Math.PI)
        // 裁剪擦除区
        ctx.clip()
        // 清除擦除区
        ctx.clearRect(0, 0, canvas.width, canvas.height)
        // 恢复之前环境
        ctx.restore()
      }
      // 绘制给定图像数据模式
      Mode.drawMode = function (data = new ImageData(canvas.width, canvas.height)) {
        // 清除实时画板
        ctx.clearRect(0, 0, canvas.width, canvas.height)
        // 重绘按下时保存的画板图像
        ctx.putImageData(data, 0, 0)
        // 开始绘制路径
        ctx.beginPath()
      }
      // 返回绘制函数
      return function (mode = curMode, data = undefined) {
        Mode[mode](data)
      }
    })()
    // 鼠标按下时执行的绘制
    const mouseDown = function () {}
    // 鼠标移动时执行的绘制
    const mouseMove = function () {
      // 绘制前操作
      switch (curMode) {
        // 绘制实时辅助线
        case 'rectMode':
        case 'cirMode':
        case 'lineMode':
          paint('drawMode', startData) // 重绘之前保存图像
          break;
          // 无需绘制辅助线
        case 'penMode':
        default:
          break;
      }
      // 绘制
      paint()
    }
    // 鼠标抬起时执行的绘制
    const mouseUp = function () {
      switch (curMode) {
        // 画线、矩形、圆形需要鼠标抬起时绘制
        case 'lineMode':
        case 'rectMode':
        case 'cirMode':
          paint()
          break;
        case 'penMode':
        case 'eraserMode':
        default:
          break;
      }
    }
    // 应用绘制
    const draw = function (state) {
      switch (state) {
        case 'down':
          mouseDown()
          break;
        case 'move':
          mouseMove()
          break;
        case 'up':
          mouseUp()
          break;
        default:
          break;
      }
    }
    /* -----------------初始化函数----------------*/
    // 初始化画板
    const init = function () {
      // 初始化画笔颜色
      ctx.strokeStyle = btnColor.value
      // 初始化画板颜色
      canvas.style.backgroundColor = '#ffffff'
      // 初始化虚化颜色
      ctx.shadowColor = btnColor.value
      // 初始化画笔粗细
      ctx.lineWidth = lineWidthSmall.value * 1 + lineWidthBig.value * 1
      // 初始化虚化值
      ctx.shadowBlur = shadowBlur.value

      // 显示初始滑动值
      // 画笔微调值
      lineWidthSmallSpan.innerHTML = lineWidthSmall.value
      // 画笔粗调值
      lineWidthBigSpan.innerHTML = lineWidthBig.value
      // 画笔虚化值
      shadowBlurSpan.innerHTML = shadowBlur.value

      // 保存原始画板数据
      backOrRestore('save', ctx.getImageData(0, 0, canvas.width, canvas.height))
    }

    /* ------------------------------获取DOM---------------------------*/
    // 获取画笔颜色
    const btnColor = document.querySelector('#color')
    // 获取画板颜色
    const btnBgc = document.querySelector('#bgc')
    // 获取画笔粗细微调
    const lineWidthSmall = document.querySelector('#lineWidthSmall')
    const lineWidthSmallSpan = document.querySelector('#text1')
    // 获取画笔粗细粗调
    const lineWidthBig = document.querySelector('#lineWidthBig')
    const lineWidthBigSpan = document.querySelector('#text2')
    // 获取画笔虚化
    const shadowBlur = document.querySelector('#shadowBlur')
    const shadowBlurSpan = document.querySelector('#text3')
    // 获取画笔
    const btnPen = document.querySelector('#pen')
    // 获取直线
    const btnLine = document.querySelector('#line')
    // 获取矩形
    const btnRect = document.querySelector('#rect')
    // 获取圆形
    const btnCir = document.querySelector('#cir')
    // 获取橡皮擦
    const btnEraser = document.querySelector('#eraser')
    // 获取清空按钮
    const btnClear = document.querySelector('#clear')
    // 获取撤销按钮
    const btnBack = document.querySelector('#back')
    // 获取恢复按钮
    const btnRestore = document.querySelector('#restore')

    /* ------------------------------注册事件---------------------------*/
    /* --------画布设置------- */
    // 改变画笔颜色
    btnColor.onchange = function () {
      // 设置虚化颜色
      ctx.shadowColor = btnColor.value
      // 设置画笔颜色
      ctx.strokeStyle = btnColor.value
    }
    // 改变画板颜色
    btnBgc.onchange = function () {
      // 设置画板颜色
      canvas.style.backgroundColor = btnBgc.value
    }
    // 改变画笔粗细微调
    lineWidthSmall.onchange = function () {
      // 设置画笔粗细，微调+粗调
      ctx.lineWidth = lineWidthSmall.value * 1 + lineWidthBig.value * 1
      // 显示微调值
      lineWidthSmallSpan.innerHTML = lineWidthSmall.value
    }
    // 改变画笔粗细粗调
    lineWidthBig.onchange = function () {
      // 设置画笔粗细，微调+粗调
      ctx.lineWidth = lineWidthSmall.value * 1 + lineWidthBig.value * 1
      // 显示粗调值
      lineWidthBigSpan.innerHTML = lineWidthBig.value
    }
    // 改变画笔虚化
    shadowBlur.onchange = function () {
      // 设置画笔虚化值
      ctx.shadowBlur = shadowBlur.value * 1
      // 显示虚化值
      shadowBlurSpan.innerHTML = shadowBlur.value
    }
    /* --------切换模式------- */
    // 使用画笔
    btnPen.onclick = function (e) {
      // 按钮切换选中
      selected(btnPen)
      // 更新当前模式
      curMode = 'penMode'
    }
    // 使用直线
    btnLine.onclick = function (e) {
      selected(btnLine)
      curMode = 'lineMode'
    }
    // 使用矩形
    btnRect.onclick = function (e) {
      selected(btnRect)
      curMode = 'rectMode'
    }
    // 使用圆形
    btnCir.onclick = function (e) {
      selected(btnCir)
      curMode = 'cirMode'
    }
    // 使用橡皮
    btnEraser.onclick = function (e) {
      selected(btnEraser)
      curMode = 'eraserMode'
    }
    // 清空按钮
    btnClear.onclick = function (e) {
      // 清空画板
      ctx.clearRect(0, 0, canvas.width, canvas.height)
      // 清空记录
      backOrRestore('clear')
      // 保存原始画板数据
      backOrRestore('save', ctx.getImageData(0, 0, canvas.width, canvas.height))
    }
    // 撤销按钮
    btnBack.onclick = function (e) {
      // 撤销操作
      backOrRestore('back')
    }
    // 恢复按钮
    btnRestore.onclick = function (e) {
      // 恢复操作
      backOrRestore('restore')
    }
    /* --------------------------------------画板事件-------------------------------- */
    /* -------------------------画板--------------------- */
    // 画板按下
    canvas.onmousedown = function (e) {
      // 记录鼠标当前坐标，更新起始和当前鼠标坐标
      pressPos = curPos = getMousePos(e.clientX, e.clientY)
      // 保存当前画板图像
      startData = ctx.getImageData(0, 0, canvas.width, canvas.height)

      // 鼠标设置为按下
      isPressDown = true
      // 开始绘制路径
      ctx.beginPath()
      // 开始绘画
      draw('down')
    }
    // 画板移动
    canvas.onmousemove = function (e) {
      // 如果是按下状态
      if (isPressDown) {
        // 更新鼠标当前坐标
        curPos = getMousePos(e.clientX, e.clientY)
        // 开始绘画
        draw('move')
      }
    }

    // 画板抬起
    canvas.onmouseup = function (e) {
      // 更新鼠标当前坐标
      curPos = getMousePos(e.clientX, e.clientY)
      // 鼠标设置为抬起
      isPressDown = false
      // 开始绘画
      draw('up')
      // 保存环境
      ctx.save()
      // 获取当前画板图像
      const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height)
      // 记录当前图像
      backOrRestore('save', imageData)
    }

    // 初始化
    init()
  </script>
</body>

</html>