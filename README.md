# Webfem 拼豆图纸生成器

一个只使用单页 `index.html + Vue` 的拼豆图纸生成器，使用 Vue 3 + Vite。

## 功能

- 选择拼豆盘尺寸：29×29、32×32、52×36、64×64、80×80，或手动输入 8–100 格。
- 导入 JPG / PNG / WEBP 图片。
- 自动裁切图片并转换到当前网格尺寸。
- 自动从 MARD 221 色中寻找最接近的拼豆颜色。
- 「预览图片」：显示网格、坐标和 MARD 色号。
- 「熨烫效果」：隐藏格线和色号，只保留颜色区块。
- 「修改图片」：右侧显示 221 色调色盘，点击颜色后再点击格子即可重新上色。
- 拖动右下角 ↘ 手柄可以直接调整拼豆盘宽高。
- 支持导出当前视图为 PNG。

## 启动

```bash
npm install
npm run dev
```

打开 Vite 显示的本地地址即可。

## 结构

```text
index.html
src/
  App.vue       # 整个页面与功能
  main.js       # Vue 入口
  palette.js    # MARD 221 色盘
  style.css     # 页面样式
```

没有使用 Vue Router，因此整个项目只有一个 index page。
