# JishiNavigation - 济事楼四楼导览系统

JishiNavigation 是一个基于 Vue 3 和 Three.js 开发的现代化 3D 导航系统，专门为济事楼四楼打造。该项目利用最新的 Web 技术，为用户提供直观、交互性强的室内导航体验。

## 功能特性

### 核心功能
- 🎯 3D 场景渲染与交互
  - 济事楼四楼完整三维地图展示
  - 支持鼠标拖拽、缩放、旋转等交互操作
  - 真实还原建筑内部结构和空间布局


- 🔍 房间查询系统
  - 支持房间号快速搜索
  - 关键字智能匹配


### 交互功能
- 🎮 便捷操作
  - 房间信息快速预览
  - 房间位置快速定位


- 📱 响应式设计
  - 自适应不同屏幕尺寸
  - 支持移动端访问
  - 触屏友好的操作界面

### 信息展示
- 📊 房间信息展示
  - 显示房间用途
  - 显示房间设施信息

- 🎨 视觉体验
  - 流畅的动画效果
  - 清晰的视觉引导
  - 直观的图标系统
  - 舒适的配色方案

## 技术栈

- **前端框架**: Vue 3
- **构建工具**: Vite
- **3D 渲染**: Three.js
- **动画库**: GSAP
- **文本渲染**: Troika Three Text
- **开发工具**: VSCode + Volar

## 开发环境要求

- Node.js (推荐 v16.0.0 或更高版本)
- npm (推荐 v7.0.0 或更高版本)

## 项目设置

### 安装依赖

```sh
npm install
```

### 开发环境运行

```sh
npm run dev
```

### 生产环境构建

```sh
npm run build
```

### 预览生产构建

```sh
npm run preview
```

## 项目结构

```
JishiNavigation/
├── public/          # 静态资源目录
├── src/             # 源代码目录
│   ├── assets/      # 项目资源文件
│   ├── components/  # Vue 组件
│   ├── pages/       # 页面组件
│   ├── App.vue      # 根组件
│   └── main.js      # 入口文件
├── vite.config.js   # Vite 配置文件
└── package.json     # 项目配置文件
```

## 开发工具推荐

- [VSCode](https://code.visualstudio.com/) - 推荐的代码编辑器
- [Volar](https://marketplace.visualstudio.com/items?itemName=Vue.volar) - Vue 3 的官方 VSCode 扩展
  - 注意：使用 Volar 时需要禁用 Vetur

## 配置说明

查看 [Vite 配置参考](https://vite.dev/config/) 了解更多配置选项。

## 贡献指南

1. Fork 本仓库
2. 创建您的特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交您的更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 开启一个 Pull Request

## 许可证

本项目采用 MIT 许可证 - 查看 [LICENSE](LICENSE) 文件了解详情

## 联系方式

如有任何问题或建议，请通过以下方式联系我们：
- 提交 Issue
- 发送邮件至 qiankungangyi@163.com

## 致谢

感谢所有为本项目做出贡献的开发者！
