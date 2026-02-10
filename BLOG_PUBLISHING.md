# Hexo Blog Publishing Skill

**使用场景**：自动发布技术博客文章到 Hexo 博客系统。

## 博客架构

### 仓库信息
- **源码仓库**：https://github.com/chiuhoukazusa/chiuhou-blog-source
- **部署仓库**：https://github.com/chiuhoukazusa/chiuhou-tech-blog
- **博客地址**：https://chiuhoukazusa.github.io/chiuhou-tech-blog/

### 本地环境
- **工作目录**：`/root/.openclaw/workspace/chiuhou-blog-new/`
- **主题**：Butterfly 5.5.4
- **文章目录**：`source/_posts/`

## 发布文章流程

### 1. 创建新文章

```bash
cd /root/.openclaw/workspace/chiuhou-blog-new

# 方式 A：使用 hexo 命令创建
hexo new "文章标题"

# 方式 B：直接创建 Markdown 文件
cat > source/_posts/article-name.md << 'EOF'
---
title: 文章标题
date: YYYY-MM-DD HH:MM:SS
tags:
  - 标签1
  - 标签2
categories:
  - 分类名
cover: 封面图URL（可选）
---

文章内容...
EOF
```

### 2. Front-matter 配置说明

**必需字段**：
```yaml
title: 文章标题
date: 2026-02-10 10:00:00
```

**推荐字段**：
```yaml
tags:
  - C++
  - 图形学
categories:
  - 每日编程实践
cover: https://example.com/cover.png  # 文章封面图（首页卡片显示）
```

**Butterfly 主题特有**：
- `cover`: 文章封面图（显示在首页卡片上）
- `description`: 文章摘要（可选，默认自动提取）
- `top`: 是否置顶（true/false）

### 3. 生成和部署

```bash
cd /root/.openclaw/workspace/chiuhou-blog-new

# 一键发布（推荐）
hexo clean && hexo generate && hexo deploy

# 简写
hexo c && hexo g && hexo d
```

**自动完成的操作**：
1. 清理旧文件
2. 生成静态 HTML
3. 推送到源码仓库（手动 git push）
4. 推送到部署仓库（hexo deploy 自动）
5. GitHub Pages 自动更新

### 4. 推送源码（重要！）

```bash
cd /root/.openclaw/workspace/chiuhou-blog-new

git add .
git commit -m "新增博客文章: 文章标题"
git push origin main
```

**注意**：`hexo deploy` 只推送生成的 HTML，不会推送源码。源码需要手动 push！

## 图片处理

### 推荐方式：使用 GitHub 作为图床

```markdown
![图片说明](https://raw.githubusercontent.com/username/repo/branch/path/image.png)
```

### 本地图片

1. 放在 `source/images/` 目录
2. 引用：`![图片](/images/image.png)`

## 常用命令

```bash
# 创建新文章
hexo new "标题"

# 创建草稿
hexo new draft "标题"

# 发布草稿
hexo publish "标题"

# 清理
hexo clean

# 生成
hexo generate  # 或 hexo g

# 本地预览
hexo server    # 或 hexo s
# 访问 http://localhost:4000

# 部署
hexo deploy    # 或 hexo d

# 一键发布
hexo clean && hexo g && hexo d
```

## 自动化集成（用于 daily-coding-practice）

### 发布函数示例

```python
import os
import subprocess
from datetime import datetime

def publish_to_hexo_blog(title, content, tags, category, cover_image_url=None):
    """
    发布文章到 Hexo 博客
    
    Args:
        title: 文章标题
        content: Markdown 内容
        tags: 标签列表 ['tag1', 'tag2']
        category: 分类名称
        cover_image_url: 封面图 URL（可选）
    """
    blog_dir = "/root/.openclaw/workspace/chiuhou-blog-new"
    
    # 生成文件名（使用拼音或英文）
    filename = title.lower().replace(' ', '-')
    filename = re.sub(r'[^\w\-]', '', filename)
    
    # 创建 Front-matter
    front_matter = f"""---
title: {title}
date: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
tags:
{chr(10).join(f'  - {tag}' for tag in tags)}
categories:
  - {category}
"""
    
    if cover_image_url:
        front_matter += f"cover: {cover_image_url}\n"
    
    front_matter += "---\n\n"
    
    # 写入文件
    post_path = f"{blog_dir}/source/_posts/{filename}.md"
    with open(post_path, 'w', encoding='utf-8') as f:
        f.write(front_matter + content)
    
    # 生成并部署
    os.chdir(blog_dir)
    
    # 生成静态文件
    subprocess.run(["hexo", "clean"], check=True)
    subprocess.run(["hexo", "generate"], check=True)
    
    # 部署到 GitHub Pages
    subprocess.run(["hexo", "deploy"], check=True)
    
    # 推送源码
    subprocess.run(["git", "add", "."], check=True)
    subprocess.run(["git", "commit", "-m", f"新增博客文章: {title}"], check=True)
    subprocess.run(["git", "push", "origin", "main"], check=True)
    
    print(f"✅ 博客发布成功: {title}")
    print(f"🌐 访问地址: https://chiuhoukazusa.github.io/chiuhou-tech-blog/")
```

### Shell 脚本版本

```bash
#!/bin/bash
# publish-blog.sh

BLOG_DIR="/root/.openclaw/workspace/chiuhou-blog-new"
TITLE="$1"
FILENAME="$2"

cd "$BLOG_DIR"

# 创建文章
hexo new "$TITLE"

# 等待用户编辑...
echo "请编辑文章: source/_posts/$FILENAME.md"
read -p "编辑完成后按回车继续..."

# 生成并部署
hexo clean && hexo generate && hexo deploy

# 推送源码
git add .
git commit -m "新增博客文章: $TITLE"
git push origin main

echo "✅ 发布完成！"
```

## 故障排查

### 问题 1：hexo deploy 失败

**原因**：未安装部署插件

**解决**：
```bash
npm install hexo-deployer-git --save
```

### 问题 2：Git 推送需要密码

**原因**：未配置 Git credentials

**解决**：
```bash
# 方式 A：使用 Git credential store
git config --global credential.helper store

# 方式 B：配置 ~/.git-credentials
echo "https://username:token@github.com" > ~/.git-credentials
chmod 600 ~/.git-credentials
```

### 问题 3：主题显示异常

**原因**：主题未正确安装

**解决**：
```bash
npm install hexo-theme-butterfly --save
cp -r node_modules/hexo-theme-butterfly themes/butterfly
```

### 问题 4：GitHub 阻止推送（包含 token）

**原因**：_config.yml 中包含 GitHub token

**解决**：
```yaml
# 错误：
deploy:
  repo: https://user:TOKEN@github.com/...

# 正确：
deploy:
  repo: https://github.com/user/repo.git
```

使用 Git credentials 而不是在 URL 中包含 token。

## 配置文件参考

### _config.yml 关键配置

```yaml
# 网站信息
title: Chiuhou 技术博客
subtitle: '代码人生，持续学习'
description: '记录编程学习和项目实践'
author: Chiuhou
language: zh-CN
timezone: 'Asia/Shanghai'

# URL
url: https://chiuhoukazusa.github.io/chiuhou-tech-blog
root: /chiuhou-tech-blog/

# 主题
theme: butterfly

# 部署
deploy:
  type: git
  repo: https://github.com/chiuhoukazusa/chiuhou-tech-blog.git
  branch: main
```

### Butterfly 主题配置（可选）

主题配置文件：`themes/butterfly/_config.yml`

**常用配置**：
- 头像、背景图
- 社交链接
- 评论系统
- 搜索功能
- 代码高亮主题

参考官方文档：https://butterfly.js.org/

## 最佳实践

1. **文章命名**：使用英文或拼音，避免中文文件名
2. **图片管理**：使用外部图床（GitHub、CDN）
3. **定期备份**：源码推送到 GitHub
4. **提交信息**：清晰描述修改内容
5. **测试预览**：发布前运行 `hexo server` 本地预览

## 维护任务

### 定期更新依赖

```bash
cd /root/.openclaw/workspace/chiuhou-blog-new
npm update
git add package.json package-lock.json
git commit -m "更新依赖"
git push origin main
```

### 主题升级

```bash
npm update hexo-theme-butterfly
cp -r node_modules/hexo-theme-butterfly themes/butterfly
```

## 统计信息

- **创建日期**：2026-02-10
- **文章总数**：7 篇
- **主题**：Butterfly 5.5.4
- **部署方式**：hexo-deployer-git
- **托管平台**：GitHub Pages
