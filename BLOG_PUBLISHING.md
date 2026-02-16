# Hexo Blog Publishing Skill - 强制性流程文档

**⚠️ 严重警告**：README.md ≠ 博客发布！必须执行完整的7步流程！

## ❌ 常见错误（严禁）

### 错误1：混淆概念
```
✗ 错误：写README.md = 发布博客
✓ 正确：README.md是项目文档，博客需要单独发布到Hexo网站
```

### 错误2：跳过流程
```
✗ 错误：只创建文章文件就认为完成
✓ 正确：必须执行hexo deploy才能发布到网站
```

### 错误3：缺少验证
```
✗ 错误：没有访问博客网站确认文章存在
✓ 正确：打开博客URL验证文章已发布
```

## ✅ 强制性7步流程（缺一不可）

### 📋 发布前检查清单
在开始发布前，必须确认：
- [ ] 项目日期正确（匹配当天日期）
- [ ] 项目代码存在且可编译
- [ ] 项目有输出图片
- [ ] 图片文件完整无损

### 第1步：确认项目信息 ✓
```bash
# 找到今日项目目录
ls /root/.openclaw/workspace/daily-coding-practice/2026/02/

# 查看项目README了解内容
cat <项目目录>/README.md

# 确认输出图片存在
ls <项目目录>/*.png
```

**验证点**：项目日期、名称、代码文件、输出图片都存在

### 第2步：准备封面图 ✓
```bash
# 创建图床目录
mkdir -p /root/.openclaw/workspace/blog_img/<项目名>-<日期>/

# 复制封面图
cp <项目目录>/output.png /root/.openclaw/workspace/blog_img/<项目名>-<日期>/
```

**验证点**：图片文件成功复制到图床目录

### 第3步：创建博客文章 ✓
```bash
# 工作目录（注意：是chiuhou-blog-source不是chiuhou-blog-new）
cd /root/.openclaw/workspace/chiuhou-blog-source

# 创建文章文件
vi source/_posts/daily-coding-<项目名>-<日期>.md
```

**Front-matter模板**（强制格式）：
```yaml
---
title: <文章标题> - 图形学每日挑战
date: YYYY-MM-DD HH:MM:SS
categories: [每日编程实践]
tags: [图形学, 算法, <相关标签>]
cover: https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/<项目名>-<日期>/<图片名>.png
---
```

**验证点**：
- [ ] categories必须是`[每日编程实践]`（方括号格式）
- [ ] cover链接格式正确
- [ ] 文章内容基于实际项目README

### 第4步：上传图片到图床 ✓
```bash
cd /root/.openclaw/workspace/blog_img
git add <项目名>-<日期>/
git commit -m "添加<项目名>封面图"
git push origin main
```

**验证点**：GitHub上能看到新上传的图片

### 第5步：生成静态网站 ✓
```bash
cd /root/.openclaw/workspace/chiuhou-blog-source
hexo clean && hexo generate
```

**验证点**：输出显示"XX files generated"，无错误信息

### 第6步：部署到GitHub Pages ✓
```bash
cd /root/.openclaw/workspace/chiuhou-blog-source
hexo deploy
```

**验证点**：
- 输出显示"Deploy done: git"
- Git commit成功推送到chiuhou-tech-blog仓库

### 第7步：推送博客源码 ✓
```bash
cd /root/.openclaw/workspace/chiuhou-blog-source
git add source/_posts/*.md
git commit -m "新增博客文章：<文章标题>"
git push origin main
```

**验证点**：GitHub上chiuhou-blog-source仓库有新的commit

---

## 🔍 发布后验证（强制）

### 验证步骤
1. **等待30秒**（GitHub Pages缓存更新）
2. **访问博客首页**：https://chiuhoukazusa.github.io/chiuhou-tech-blog/
3. **确认文章出现在首页**
4. **点击文章标题**，确认内容完整
5. **检查封面图**是否正确显示

### 验证清单
- [ ] 首页能看到新文章卡片
- [ ] 封面图正确显示
- [ ] 文章标题正确
- [ ] 文章内容完整
- [ ] 分类显示为"每日编程实践"
- [ ] 标签正确

---

## 📚 关键概念区分

### README.md vs 博客文章

| 项目 | README.md | 博客文章 |
|------|-----------|---------|
| **位置** | GitHub项目目录 | Hexo博客网站 |
| **作用** | 项目文档 | 公开博客 |
| **读者** | GitHub访问者 | 博客访问者 |
| **格式** | Markdown | Markdown + Front-matter |
| **发布** | git push | hexo deploy |

**核心区别**：README是项目的说明文档，博客是面向公众的技术文章，两者完全独立！

---

## ⚠️ 紧急提醒（每次发布前必读）

### 失误1：混淆README和博客
**症状**：认为"README.md已发布"等于"博客已发布"
**后果**：博客网站上没有文章，用户看不到内容
**解决**：严格执行7步流程，最后验证博客网站

### 失误2：跳过hexo deploy
**症状**：只创建了文章文件，没有运行hexo deploy
**后果**：文章没有部署到GitHub Pages
**解决**：第6步是强制性的，不能跳过

### 失误3：分类格式错误
**症状**：使用旧的YAML格式或错误的分类名
**后果**：文章分类混乱，无法统一管理
**解决**：统一使用`categories: [每日编程实践]`格式

---

## 🎯 流程自检表（每次发布必填）

```
发布日期：_______
项目名称：_______

[ ] 第1步：确认项目信息
[ ] 第2步：准备封面图
[ ] 第3步：创建博客文章
[ ] 第4步：上传图片到图床
[ ] 第5步：生成静态网站
[ ] 第6步：部署到GitHub Pages
[ ] 第7步：推送博客源码
[ ] 发布后验证：访问博客确认

签名：_______ 完成时间：_______
```

---

## 📖 正确工作目录

**⚠️ 注意路径变更**

| 用途 | 正确路径 |
|------|---------|
| 博客源码 | `/root/.openclaw/workspace/chiuhou-blog-source` |
| 图床仓库 | `/root/.openclaw/workspace/blog_img` |
| 项目代码 | `/root/.openclaw/workspace/daily-coding-practice` 或 `/root/.openclaw/workspace/github-daily-coding-practice` |

**错误路径**（已废弃）：
- ~~`/root/.openclaw/workspace/chiuhou-blog-new`~~（不存在）

---

## 🔧 故障排查

### 问题1：hexo命令找不到
```bash
# 检查hexo是否安装
cd /root/.openclaw/workspace/chiuhou-blog-source
npm list hexo

# 重新安装（如需要）
npm install
```

### 问题2：部署失败
```bash
# 检查Git配置
git config --list | grep user

# 检查远程仓库
git remote -v
```

### 问题3：图片不显示
```bash
# 检查图片URL格式
# 正确：https://raw.githubusercontent.com/chiuhoukazusa/blog_img/main/<path>/<file>.png
# 错误：相对路径或错误域名

# 手动验证图片URL
curl -I <图片URL>
```

---

## 💡 最佳实践

1. **先项目后博客**：项目完成并验证后再写博客
2. **内容一致性**：博客内容必须100%基于实际项目
3. **命名规范**：使用统一的文件命名格式
4. **立即验证**：发布后立即访问博客确认
5. **定期备份**：定期推送所有仓库

---

## 🎓 记忆口诀

**"7步发布法，缺一不可"**

1. 确认项目（日期名称代码图）
2. 准备封面（复制到图床目录）
3. 创建文章（Front-matter+内容）
4. 上传图片（push到blog_img）
5. 生成网站（hexo clean generate）
6. 部署上线（hexo deploy重点）
7. 推送源码（git push源仓库）

**最后一步**：打开博客，眼见为实！

---

**文档版本**：v2.0  
**更新日期**：2026-02-16  
**更新原因**：修正README.md与博客发布的混淆问题  
**关键改进**：强制性流程+验证清单+概念区分