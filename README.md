# 2026-05-06-task-4 小红书图文笔记发布

## 项目概述

本项目记录了使用 **playwright-cli** 自动化工具将图文笔记发布到小红书的完整流程，包含封面图生成、自动化发布脚本、以及全流程截图记录。

## 发布内容

- **主题**：🛵 2025电动车两轮（电动自行车/电摩）销量排行榜TOP10
- **平台**：小红书创作者中心
- **状态**：已发布 ✅ （审核中）

## 文件说明

| 文件 | 说明 |
|------|------|
| `ev_two_wheeler_ranking_cover.png` | 封面图（1242×1660 深蓝色主题 TOP10 排行卡片） |
| `xhs_publish_page.png` | 小红书发布页截图 |
| `xhs_image_tab.png` | 切换到"上传图文"标签页 |
| `xhs_image_uploaded.png` | 图片上传完成 |
| `xhs_filled.png` | 标题和正文填写完成 |
| `xhs_success.png` | 发布成功页面 |
| `xhs_note_manager.png` | 笔记管理页确认"审核中" |

## 技术栈

- **Python Pillow** — 封面图生成
- **playwright-cli** — 浏览器自动化（替代 agent-browser，因代理环境兼容性问题）
- **gh CLI v2.66.1** — GitHub 仓库管理

## 关键技术要点

### playwright-cli 使用
```bash
# 前置：必须清空 NODE_OPTIONS
NODE_OPTIONS="" playwright-cli <command>

# 核心流程
NODE_OPTIONS="" playwright-cli open "https://creator.xiaohongshu.com/publish/publish" --headed --persistent
# → 切换图文(JS) → 上传图片 → 填写内容 → 移除遮罩层 → 点击发布
```

### 坑点速查
1. `NODE_OPTIONS=""` 必须加在命令前（解决 `--use-system-ca` 冲突）
2. JS DOM 操作代替 ref click（解决元素视口外问题）
3. 发布前必须移除遮罩层（mask/overlay/drawer/modal/dialog/popup/tippy）
4. 精确匹配"发布"按钮文本，排除"加入合集"误触

## 相关技能

基于本次实战经验创建了通用 **[xhs-publisher](https://github.com/Mawebo-collab/.workbuddy/skills/xhs-publisher)** 小红书发布技能。

## License

MIT
