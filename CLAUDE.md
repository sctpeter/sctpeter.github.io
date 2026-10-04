# CLAUDE.md

本仓库是 sctpeter 的个人博客（Hugo + PaperMod 主题，通过 GitHub Actions 部署到 GitHub Pages）。

## 工作约定

- **排版、功能上的调整：直接提交并推送到 `main` 分支即可**，不需要新建分支或开 Pull Request。用户手工修改也一并提交`git add -A `
- 推送前先本地构建确认无报错：`hugo --gc --minify`。

## 排查问题与接入第三方服务

- **遇到问题先查源码或官方文档，再下结论。** 不要在没有依据时假设某个库、主题或第三方服务（Hugo、PaperMod、GoatCounter、utterances、follow.it 等）的行为。开源的就去读源码（如 `git clone` 到临时目录后 grep），否则查官方文档。
- **区分「已确认」和「推测」。** 没有查证的判断要明说是推测（「我怀疑……」），不能说成「找到原因了」；也不要在未验证的情况下推送「修复」并声称问题已解决。
- **接入第三方服务前先读它的文档**，把限制（缓存、延迟、计数规则、配额等）提前告诉用户，不要等用户以为功能坏了才发现。
- 能自己查到的信息先自己查，不要让用户一轮轮帮忙收集诊断信息。

## 项目结构

- `hugo.yaml`：站点配置。公共参数（评论等）在顶层 `params`；按语言区分的内容（标题、profile 模式、菜单、社交链接、follow.it 表单）在 `languages.zh` / `languages.en` 下。
- `content/`：文章与页面（`posts/`、`about.md`、`archives.md`、分类/标签索引）。英文译文用同名 `.en.md` 文件（如 `posts/foo.en.md`），生成在 `/en/` 下。
- `i18n/zh.yaml`、`i18n/en.yaml`：自定义模板文案（订阅框、评论标题等），与主题自带的 i18n 合并。
- `layouts/`：对主题模板的覆盖。不要直接修改 `themes/PaperMod`（git submodule），需要改模板时复制到 `layouts/` 下同路径再改。
  - `layouts/_partials/comments.html`：GitHub 登录评论（utterances，评论存为本仓库 Issues），配置在 `hugo.yaml` 的 `params.utterances`。
  - `layouts/_partials/extend_post_content.html`：文章末尾的订阅框（follow.it 邮件订阅 + RSS，表单地址在 `hugo.yaml` 的 `languages.<lang>.params.followit.formAction`，中英文各一个，留空则只显示 RSS）。
  - `layouts/_partials/header.html`：覆盖主题的导航栏，语言切换按钮跳到当前页面的译文（没有译文时回到另一语言首页）。
  - `layouts/_partials/extend_head.html`：按浏览器语言自动跳转到译文；用户手动切换后记在 localStorage，不再自动跳转。另外加载 GoatCounter 访问统计（站点代码在 `hugo.yaml` 的 `params.goatcounter`，只在生产构建中加载），并把阅读次数填进文章元信息。
  - `layouts/_partials/post_meta.html`：覆盖主题的文章元信息，末尾加一个默认隐藏的「N 次阅读」占位，取到 GoatCounter 计数后才显示。
  - `layouts/rss.xml`：覆盖主题的订阅源模板，栏目订阅源改用 `RegularPagesRecursive`，包含子栏目（如 `posts/hpc/...`）里的文章；主题原版只收录直接放在该目录下的文章，会导致 `/posts/index.xml` 为空。
  - `layouts/_partials/translation_list.html`：文章标题下的译文链接。链接要带 `data-lang`，点击后才会记住语言选择，否则会被自动跳转弹回。
- `.github/workflows/hugo.yaml`：部署流程（Hugo extended，版本见文件内 `HUGO_VERSION`）。

## 注意

- 单个页面可在 front matter 中用 `comments: false` 关闭评论（如 `about.md`）。
- 站点是中英双语：中文为默认语言（根路径），英文在 `/en/`。新增模板文案时用 `i18n` 函数，并在 `i18n/zh.yaml`、`i18n/en.yaml` 中都补上。
- Claude 翻译的英文页面，在正文末尾标注 `*This post was translated from the Chinese original by Claude.*`。如果是Codex翻译的，则标注Codex。
- 评论按语言分开（utterances 的 `pathname` 映射，中英文页面路径不同）。
- GoatCounter（已查源码/文档确认）：公开计数接口 `/counter/<path>.json` 在服务端缓存约 4 小时（包括 404），文章里的阅读次数会滞后，无法关闭；后台数据约 10 秒写入一次；同一访客 8 小时内重复访问同一页面只算 1 次；中英文页面路径不同，分开计数。

## 工作流

- 不论什么流程，先git pull拉取最新内容
- 新增文件
  - 先与用户沟通清楚新增的文件放置路径，是否需要放到对应的类别中
  - 然后从temp里面把文件移动到对应路径
  - 把文件翻译成英文并且放置到英文版本对应路径
  - 修改相关配置
  - 本地构建与检查
    - 是否成功构建
    - 路由url与页面显示的标题是否一致 如果不一致，则参考同级内容中原有的标题与url形式也可以
  - 推送

