# boke

个人博客：Hugo 静态站点，GitHub Pages 自动部署。

- 线上地址：https://clike-peng.github.io/boke/
- 文章目录：`content/posts/`
- frontmatter：`title` / `date` / `tags` / `author`

## 写一篇新文章

```bash
git switch -c post/文章名
# 在 content/posts/ 下新建 xxx.md，写好 frontmatter 与正文
git add content/posts/xxx.md
git commit -m "post: 文章标题"
git push -u origin post/文章名
```

然后在 GitHub 上开 PR，合并到 `main` 后 `.github/workflows/pages.yml`
会自动构建并发布到 Pages。

## 本地预览（可选）

需要先安装 Hugo（本机当前未安装）：

```bash
hugo server -D
```
