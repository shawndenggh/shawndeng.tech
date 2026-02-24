# ShawnDeng's Tech Blog

我的个人技术博客，使用 Hugo + PaperMod 主题构建。

## 📝 博客地址

https://shawndeng.tech

## 🛠️ 技术栈

- **静态网站生成器**: [Hugo](https://gohugo.io/)
- **主题**: [PaperMod](https://github.com/adityatelange/hugo-PaperMod)
- **评论系统**: [Giscus](https://giscus.app/) (GitHub Discussions)
- **部署**: Nginx + Let's Encrypt (HTTPS)
- **统计**: Google Analytics

## 🚀 本地开发

### 预览博客

```bash
hugo server -D
```

访问 http://localhost:1313

### 创建新文章

```bash
hugo new content/posts/文章标题.md
```

### 生成静态文件

```bash
hugo --minify
```

生成的文件在 `public/` 目录

## 📦 部署

部署到服务器：

```bash
# 在服务器上
cd /var/www/shawndeng.tech
git pull
hugo --minify
```

## 📄 License

文章内容采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议。

代码部分采用 MIT 许可协议。
