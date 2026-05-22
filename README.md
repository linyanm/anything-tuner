# Anything Tuner

Vue 3 响应式吉他调音器，使用浏览器 Web Audio API 读取麦克风并估算音高。项目基于 Vite，可以部署到 GitHub Pages。

## 功能

- 支持电脑和手机布局
- 支持标准调弦、降 D、全降半音、全降一音、开放 G、DADGAD
- 显示目标音、检测音、当前频率、目标频率和 cents 偏差
- Vue 3 + Vite 实现

## 本地预览

由于麦克风权限通常要求安全上下文，建议使用本地 HTTP 服务或 GitHub Pages 预览：

```sh
nvm use
npm install
npm run dev
```

部署到 GitHub Pages 后会自动使用 HTTPS，麦克风权限可正常请求。

## GitHub Pages

```sh
npm run build
```

将 `dist` 目录作为 GitHub Pages 发布产物。仓库名不是根域名时，可在 `vite.config.js` 中按实际仓库名设置 `base`。
