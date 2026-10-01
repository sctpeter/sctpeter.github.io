# CLAUDE.md

本仓库是 sctpeter 的个人博客（Hugo + PaperMod 主题，通过 GitHub Actions 部署到 GitHub Pages）。

## 工作约定

- **排版、功能上的调整：直接提交并推送到 `main` 分支即可**，不需要新建分支或开 Pull Request。
- 推送前先本地构建确认无报错：`hugo --gc --minify`。

## 项目结构

- `hugo.yaml`：站点配置（语言 zh、profile 模式、菜单、社交链接、评论等）。
- `content/`：文章与页面（`posts/`、`about.md`、`archives.md`、分类/标签索引）。
- `layouts/`：对主题模板的覆盖。不要直接修改 `themes/PaperMod`（git submodule），需要改模板时复制到 `layouts/` 下同路径再改。
  - `layouts/_partials/comments.html`：GitHub 登录评论（utterances，评论存为本仓库 Issues），配置在 `hugo.yaml` 的 `params.utterances`。
  - `layouts/_partials/extend_post_content.html`：文章末尾的订阅框（follow.it 邮件订阅 + RSS，表单地址在 `hugo.yaml` 的 `params.followit.formAction`，留空则只显示 RSS）。
- `.github/workflows/hugo.yaml`：部署流程（Hugo extended，版本见文件内 `HUGO_VERSION`）。

## 注意

- 单个页面可在 front matter 中用 `comments: false` 关闭评论（如 `about.md`）。
- 站点文案使用中文。
