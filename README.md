# Anything Tuner

Vue 3 响应式吉他调音器，使用浏览器 Web Audio API 读取麦克风并估算音高。项目基于 Vite，可以部署到 GitHub Pages。

## 在线访问

GitHub Pages 地址：

https://linyanm.github.io/anything-tuner/

## 功能

- 支持电脑和手机布局
- 支持标准调弦、降 D、全降半音、全降一音、开放 G、DADGAD
- 显示目标音、检测音、当前频率、目标频率和 cents 偏差
- Vue 3 + Vite 实现

## 本地开发

由于麦克风权限通常要求安全上下文，建议使用本地 HTTP 服务或 GitHub Pages 预览：

```sh
nvm use
npm install
npm run dev
```

本地开发地址默认是：

```text
http://127.0.0.1:5173/
```

部署到 GitHub Pages 后会自动使用 HTTPS，麦克风权限可正常请求。

## 开发须知

- Node 版本以 `.nvmrc` 为准，当前使用 Node v24。
- 本地首次开发使用 `npm install`；CI 使用 `npm ci`，因此需要提交 `package-lock.json`。
- `dist/` 是构建产物，不提交到仓库；GitHub Actions 会自动构建并部署。
- 仓库项目页访问路径是 `/anything-tuner/`，所以 `vite.config.js` 中需要保留 `base: "/anything-tuner/"`。
- 调音器依赖麦克风权限；如果浏览器拒绝授权，需要检查站点权限或使用 HTTPS 环境访问。

## 构建

```sh
npm run build
```

构建产物会输出到 `dist/`。

## 部署

项目使用 GitHub Actions 自动部署到 GitHub Pages。推送到 `master` 分支后，会自动执行构建并发布 `dist/`。

GitHub 仓库需要在 `Settings` -> `Pages` -> `Build and deployment` 中将 `Source` 设置为 `GitHub Actions`。
