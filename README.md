# CurveDrawing

This template should help get you started developing with Vue 3 in Vite.

## Recommended IDE Setup

[VSCode](https://code.visualstudio.com/) + [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) (and disable Vetur).

## Customize configuration

See [Vite Configuration Reference](https://vite.dev/config/).

## Project Setup

```sh
npm install
```

### Compile and Run

```sh
npm run dev
```

### 操作指南
使用了 Catmull-Rom 样条曲线 来绘制曲线
分为 预览模式 和 固定模式 
预览模式：
  随着鼠标移动，曲线将鼠标所在点作为最后一个点，实时显示曲线。
  按下鼠标左键后，固定下一个点
固定模式：
  利用已确认的点绘制曲线
  可以调整已确认的点的位置，重新调整曲线形状
按下 ESC 可以在两个模式之间切换
