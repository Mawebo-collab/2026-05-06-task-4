# xhs-publisher - 小红书图文笔记发布自动化

## 项目概述

本项目记录了使用 **playwright-cli** 自动化工具将图文笔记发布到小红书的完整流程，包含封面图生成、自动化发布脚本、以及全流程截图记录。

**实战验证**：2026-05-06 发布电动车两轮排行榜，2026-05-07 发布忧郁诗词笔记（含仅自己可见设置），全流程通过验证。

## 发布内容示例

### 示例 1：电动车两轮销量排行榜

- **主题**：🛵 2025电动车两轮（电动自行车/电摩）销量排行榜TOP10
- **平台**：小红书创作者中心
- **状态**：已发布 ✅ （审核中）

### 示例 2：忧郁诗词笔记

- **主题**：🌙 今天，辛苦了（忧郁诗词风格）
- **可见性**：仅自己可见 ✅
- **状态**：已发布 ✅

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
| `SKILL.md` | xhs-publisher 技能文档（8步完整流程） |

## 技术栈

- **Python Pillow** — 封面图生成
- **playwright-cli** — 浏览器自动化（替代 agent-browser，因代理环境兼容性问题）
- **gh CLI v2.66.1** — GitHub 仓库管理

## 关键技术要点

### 发布流程（8步）

```bash
# 前置：必须清空 NODE_OPTIONS
NODE_OPTIONS="" playwright-cli <command>

# 核心流程（8步）
1. 打开发布页
2. 切换到"上传图文"标签
3. 上传封面图
4. 获取快照，定位输入框
5. 填写标题和正文
6. 设置可见性（可选：仅自己可见/仅互关好友可见）
7. 移除遮罩层 + 点击发布
8. 确认发布成功
```

### 坑点速查

| # | 问题 | 原因 | 解决 |
|---|---|---|---|
| 1 | `NODE_OPTIONS=""` 必须加在命令前 | `--use-system-ca` 冲突 | 所有命令前加 `NODE_OPTIONS=""` |
| 2 | 元素在视口外（outside viewport） | 上传图文标签点不上 | 用 JS `querySelector` + `.click()` 代替 ref click |
| 3 | 发布按钮点了没反应 | 悬浮弹窗/遮罩层遮挡 | 先移除所有 overlay/mask/drawer 等 |
| 4 | 误触"加入合集" | 两按钮位置相邻 | JS 精确匹配文本 `t.trim() === '发布'` 且 `!t.includes('合集')` |
| 5 | 找不到输入框 | 上传图片后编辑器还没加载完 | 上传后 sleep 5~8 再 snapshot + fill |
| 6 | ref 失效 | 页面刷新/DOM 重构 | 每次 DOM 变化后重新 snapshot 获取新 ref |
| 7 | "公开可见"点击无反应 | ref 是 playwright-cli 内部标识 | 使用 `playwright-cli click <ref>` 命令 |
| 8 | agent-browser 页面空白 | ERR_NO_SUPPORTED_PROXIES | 改用 playwright-cli |

## 相关技能

基于本次实战经验创建了通用 **[xhs-publisher](https://github.com/Mawebo-collab/xiaohongshufabu/blob/main/SKILL.md)** 小红书发布技能。

技能已安装到本地：`~/.workbuddy/skills/xhs-publisher/`

## License

MIT
