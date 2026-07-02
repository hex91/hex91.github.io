# GoatCounter 访问统计配置指南

## 什么是 GoatCounter？
GoatCounter 是开源、无 Cookie、隐私友好的访问统计工具，适合技术博客。

## 注册步骤

### 1. 注册账号
- 访问 https://www.goatcounter.com/signup
- 填写站点名称（如：`atworkhi`），得到：`atworkhi.goatcounter.com`
- 验证邮箱

### 2. 获取 Site Code
- 登录后进入 Settings → Sites
- 复制你的站点代码（即注册时的名称）

### 3. 更新 hugo.toml
将 `hugo.toml` 中的 `goatcounter` 值改为你的站点代码：
```toml
[params.analytics]
  goatcounter = 'atworkhi'  # 改为你的 GoatCounter 站点名
```

## 备注
- GoatCounter 免费版支持每月 100 万 PV
- 数据完全隐私友好，符合 GDPR
- PaperMod 主题原生支持 GoatCounter
