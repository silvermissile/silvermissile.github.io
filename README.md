# Be Water

基于 [Quartz 5](https://quartz.jzhao.xyz/) 构建的个人数字花园。

**在线访问**: https://silvermissile.github.io

## 本地开发

```bash
# 安装依赖
npm install

# 本地预览（热重载）
npx quartz build --serve

# 构建
npx quartz build
```

## 写作

- 在 `content/` 目录下用 Markdown 编写文章
- 支持 Obsidian WikiLink (`[[]]`) 和标准 Markdown 链接
- 文章需要 YAML frontmatter（title, date, tags, description）
- 推送到 `master` 分支后自动部署

## 目录结构

```
content/
├── index.md          # 首页
├── about.md          # 关于页面
├── posts/            # 博客文章
├── assets/           # 图片等静态资源
└── llms.txt          # AI 爬虫导航文件
```
