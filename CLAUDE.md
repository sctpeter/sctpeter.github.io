# CLAUDE.md

本仓库是 sctpeter 的个人博客（Hugo + PaperMod 主题，通过 GitHub Actions 部署到 GitHub Pages）。

## 工作约定

- **排版、功能上的调整：直接提交并推送到 `main` 分支即可**，不需要新建分支或开 Pull Request。
- 推送前先本地构建确认无报错：`hugo --gc --minify`。

## 项目结构

- `hugo.yaml`：站点配置。公共参数（评论等）在顶层 `params`；按语言区分的内容（标题、profile 模式、菜单、社交链接、follow.it 表单）在 `languages.zh` / `languages.en` 下。
- `content/`：文章与页面（`posts/`、`about.md`、`archives.md`、分类/标签索引）。英文译文用同名 `.en.md` 文件（如 `posts/foo.en.md`），生成在 `/en/` 下。
- `i18n/zh.yaml`、`i18n/en.yaml`：自定义模板文案（订阅框、评论标题等），与主题自带的 i18n 合并。
- `layouts/`：对主题模板的覆盖。不要直接修改 `themes/PaperMod`（git submodule），需要改模板时复制到 `layouts/` 下同路径再改。
  - `layouts/_partials/comments.html`：GitHub 登录评论（utterances，评论存为本仓库 Issues），配置在 `hugo.yaml` 的 `params.utterances`。
  - `layouts/_partials/extend_post_content.html`：文章末尾的订阅框（follow.it 邮件订阅 + RSS，表单地址在 `hugo.yaml` 的 `languages.<lang>.params.followit.formAction`，中英文各一个，留空则只显示 RSS）。
  - `layouts/_partials/header.html`：覆盖主题的导航栏，语言切换按钮跳到当前页面的译文（没有译文时回到另一语言首页）。
  - `layouts/_partials/extend_head.html`：按浏览器语言自动跳转到译文；用户手动切换后记在 localStorage，不再自动跳转。
- `.github/workflows/hugo.yaml`：部署流程（Hugo extended，版本见文件内 `HUGO_VERSION`）。

## 注意

- 单个页面可在 front matter 中用 `comments: false` 关闭评论（如 `about.md`）。
- 站点是中英双语：中文为默认语言（根路径），英文在 `/en/`。新增模板文案时用 `i18n` 函数，并在 `i18n/zh.yaml`、`i18n/en.yaml` 中都补上。
- Claude 翻译的英文页面，在正文末尾标注 `*This post was translated from the Chinese original by Claude.*`。
- 评论按语言分开（utterances 的 `pathname` 映射，中英文页面路径不同）。

