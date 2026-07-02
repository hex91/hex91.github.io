# Giscus 评论配置指南

## 什么是 Giscus？
Giscus 是基于 GitHub Discussions 的开源评论系统，读者用 GitHub 账号登录即可评论。

## 启用步骤（需先创建 GitHub 仓库）

### 1. 在 GitHub 仓库启用 Discussions
- 进入 https://github.com/atworkhi/atworkhi.github.io/settings
- 找到 "Features" 区域
- 勾选 "Discussions"

### 2. 获取 Giscus 配置参数
- 访问 https://giscus.app/zh-CN
- 填写仓库名称：`atworkhi/atworkhi.github.io`
- 选择页面 ↔️ Discussions 映射关系：`pathname`
- 分类：`Announcements`
- 复制生成的 `<script>` 标签中的参数

### 3. 更新 hugo.toml
将获取到的 `data-repo-id` 和 `data-category-id` 填入：
```toml
[params.giscus]
  repo = 'atworkhi/atworkhi.github.io'
  repoId = '<填这里>'
  categoryId = '<填这里>'
```

## 临时禁用评论（部署前）
如果暂时不配置，可将 `hugo.toml` 中的 `enable = true` 改为 `false`。
