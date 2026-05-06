---
name: xhs-publisher
description: 小红书图文笔记发布自动化。基于 2026-05-06 实战验证成功的流程，使用 playwright-cli 发布图文笔记到小红书。触发词："发小红书"、"发布笔记"、"小红书发布"。
agent_created: true
---

# 小红书发布（xhs-publisher）

> **实战验证**: 2026-05-06 发布电动车两轮排行榜，2026-05-07 发布忧郁诗词笔记（含仅自己可见设置），全流程通过验证。

## 前置条件

| 项目 | 要求 |
|---|---|
| 封面图 | PNG/JPG，推荐 1242×1660（3:4），最大 32MB |
| 标题 | <100 字 |
| 正文 | <1000 字（小红书硬限制） |
| 环境 | playwright-cli 已安装，`NODE_OPTIONS=""` |

## 核心命令模板

**所有 playwright-cli 命令必须加 `NODE_OPTIONS=""` 前缀！**

```bash
NODE_OPTIONS="" playwright-cli <command> [args]
```

> ⚠️ 不加会报错 `node: --use-system-ca is not allowed in NODE_OPTIONS`

---

## 发布流程（8步，已验证）

### Step 1：打开发布页

```bash
NODE_OPTIONS="" playwright-cli open "https://creator.xiaohongshu.com/publish/publish" --headed --persistent
```

等待 8 秒让页面加载完成。

**验证方式**：检查 URL 是否为 `https://creator.xiaohongshu.com/publish/publish`

```bash
NODE_OPTIONS="" playwright-cli eval "() => window.location.href"
# 预期输出: https://creator.xiaohongshu.com/publish/publish
```

如果 URL 包含 `login?source` → 未登录，需要手动扫码登录。

### Step 2：切换到"上传图文"标签

默认进入的是「上传视频」标签，**必须切换**。

```bash
NODE_OPTIONS="" playwright-cli eval "() => {
  const all = Array.from(document.querySelectorAll('span'));
  const target = all.find(s => s.textContent.includes('\u56fe\u6587')); // '图文'
  if (target) { target.click(); return 'clicked'; }
  return 'not found';
}"
```

**验证**：URL 应变为 `.../publish/publish?from=tab_switch`

### Step 3：上传封面图

先触发文件选择器，再上传文件：

```bash
# 触发 file chooser
NODE_OPTIONS="" playwright-cli eval "() => {
  document.querySelector('input[type=file]').click();
  return 'triggered';
}"

# 上传图片（必须在下一步立即执行，否则 chooser 会消失）
NODE_OPTIONS="" playwright-cli upload "/path/to/cover.png"
```

**验证**：等 5~8 秒后截图，应显示「图片编辑 1/18」和编辑区域。

### Step 4：获取快照，定位输入框

上传成功后页面结构变化，必须重新获取快照：

```bash
NODE_OPTIONS="" playwright-cli snapshot --filename snap.yaml
```

在 snap.yaml 中找：
- **标题框**: `textbox "填写标题会有更多赞哦"` （如 `[ref=e220]`）
- **正文框**: `textbox` 含 placeholder `输入正文描述` （如 `[ref=e229]`）
- **发布按钮**: `button "发布"` （如 `[ref=e468]`）

### Step 5：填写标题和正文

```bash
# 填写标题（替换 e220 为实际 ref）
NODE_OPTIONS="" playwright-cli fill e220 "你的标题"

# 填写正文（替换 e229 为实际 ref）
NODE_OPTIONS="" playwright-cli fill e229 "你的正文内容..."
```

**正文格式要求**：
- 支持 emoji ✅ 换行 ✅ 列表符号 ✅
- 末尾用 `#话题标签` 格式添加话题
- 总字数 <1000

### Step 6：设置可见性（可选）

如果需要设置为"仅自己可见"或"仅互关好友可见"，执行此步骤。默认是"公开可见"。

#### 6a. 点击"公开可见"展开下拉菜单

先获取快照找到"公开可见"的 ref：

```bash
NODE_OPTIONS="" playwright-cli snapshot --filename snap2.yaml
```

在 `snap2.yaml` 中找：`公开可见`（如 `[ref=e435]`），然后点击：

```bash
NODE_OPTIONS="" playwright-cli click e435  # 替换为实际 ref
sleep 2
```

#### 6b. 选择可见性选项

再次获取快照，找到目标选项：

```bash
NODE_OPTIONS="" playwright-cli snapshot --filename snap3.yaml
```

在 `snap3.yaml` 中找：
- **仅自己可见**（如 `[ref=e529]`）
- **仅互关好友可见**（如 `[ref=e534]`）

点击对应选项：

```bash
NODE_OPTIONS="" playwright-cli click e529  # 仅自己可见
# 或
NODE_OPTIONS="" playwright-cli click e534  # 仅互关好友可见
```

**验证**：快照中应显示"仅自己可见"或"仅互关好友可见"（而不是"公开可见"）。

### Step 7：移除遮罩层 + 点击发布

⚠️ 这是最关键的一步！

#### 7a. 移除遮罩（悬浮页/弹窗遮挡会导致点击失败）

```bash
NODE_OPTIONS="" playwright-cli eval "() => {
  const sels = ['[class*=mask]','[class*=overlay]','[class*=drawer]',
    '[class*=modal]','[class*=dialog]','[class*=popup]',
    '[class*=tippy]','[class*=popover]'];
  let n=0;
  sels.forEach(s=>document.querySelectorAll(s).forEach(e=>{e.remove();n++}));
  return 'removed:'+n;
}"
```

#### 7b. 精确点击"发布"（排除"加入合集"）

```bash
NODE_OPTIONS="" playwright-cli eval "() => {
  const btns = Array.from(document.querySelectorAll('button'));
  for (const b of btns) {
    const t = b.textContent || '';
    // 精确匹配'发布'，排除'加入合集'、'定时发布'
    if (t.trim() === '\u53d1\u5e03' && !t.includes('\u5408\u96c6') && !t.includes('\u5b9a\u65f6')) {
      b.scrollIntoView({ block:'center' });
      b.click();
      return 'clicked publish';
    }
  }
  return 'not found';
}"
```

### Step 8：确认发布成功

```bash
sleep 3
NODE_OPTIONS="" playwright-cli eval "() => window.location.href"
# 预期: 包含 /publish/success 或 published=true
# 页面会重置回空白的发布界面 = 发布成功标志
```

去笔记管理页最终确认：

```bash
NODE_OPTIONS="" playwright-cli eval "() => { window.location.href = 'https://creator.xiaohongshu.com/new/note-manager'; }"
sleep 5
NODE_OPTIONS="" playwright-cli screenshot --filename note_manager.png
```

---

## 一键发布脚本模板

将以下脚本保存为 `publish_xhs.sh`，修改顶部变量即可使用：

```bash
#!/bin/bash
set -e

# ========== 配置区 ==========
TITLE="🛵 你的笔记标题"
BODY="你的正文内容（<1000字）"
COVER="/path/to/cover.png"
WORK_DIR="/tmp/xhs-publish"
# =============================

export NODE_OPTIONS=""
mkdir -p "$WORK_DIR"

echo "[1/7] 打开发布页..."
playwright-cli open "https://creator.xiaohongshu.com/publish/publish" --headed --persistent > /dev/null 2>&1
sleep 8

echo "[2/7] 切换到图文模式..."
playwright-cli eval "() => {
  const s = Array.from(document.querySelectorAll('span')).find(s=>s.textContent.includes(String.fromCharCode(22270,25991)));
  if(s){s.click();return'ok'}return'fail';
}" > /dev/null 2>&1
sleep 3

echo "[3/7] 上传封面图..."
playwright-cli eval "() => document.querySelector('input[type=file]').click()" > /dev/null 2>&1
playwright-cli upload "$COVER" > /dev/null 2>&1
sleep 8

echo "[4/7] 获取输入框 ref..."
playwright-cli snapshot --filename "$WORK_DIR/snap.yaml" > /dev/null 2>&1
TITLE_REF=$(grep -oP 'textbox \[ref=\K(e\d+)' "$WORK_DIR/snap.yaml" | head -1)
BODY_REF=$(grep -oP '^ *- textbox \[ref=\K(e\d+)' "$WORK_DIR/snap.yaml" | tail -1)
if [ -z "$TITLE_REF" ] || [ -z "$BODY_REF" ]; then
  echo "❌ 无法找到输入框"; exit 1
fi

echo "[5/8] 填写内容..."
playwright-cli fill "$TITLE_REF" "$TITLE" > /dev/null 2>&1
playwright-cli fill "$BODY_REF" "$BODY" > /dev/null 2>&1

echo "[6/8] 设置可见性（仅自己可见）..."
playwright-cli snapshot --filename "$WORK_DIR/snap2.yaml" > /dev/null 2>&1
VISIBILITY_REF=$(grep -oP 'generic \[\K(e\d+)(?=.*公开可见)' "$WORK_DIR/snap2.yaml" | head -1)
if [ -n "$VISIBILITY_REF" ]; then
  playwright-cli click "$VISIBILITY_REF" > /dev/null 2>&1
  sleep 2
  playwright-cli snapshot --filename "$WORK_DIR/snap3.yaml" > /dev/null 2>&1
  PRIVATE_REF=$(grep -oP 'generic \[\K(e\d+)(?=.*仅自己可见)' "$WORK_DIR/snap3.yaml" | head -1)
  if [ -n "$PRIVATE_REF" ]; then
    playwright-cli click "$PRIVATE_REF" > /dev/null 2>&1
  fi
fi

echo "[7/8] 移除遮罩 + 点击发布..."
playwright-cli eval "() => {
  ['[class*=mask]','[class*=overlay]','[class*=drawer]',
   '[class*=modal]','[class*=dialog]','[class*=popup]',
   '[class*=tippy]'].forEach(s=>document.querySelectorAll(s).forEach(e=>e.remove()));
  for(const b of Array.from(document.querySelectorAll('button'))){
    const t=b.textContent||'';
    if(t.trim()==='\u53d1\u5e03'&&!t.includes('\u5408\u96c6')){b.scrollIntoView({block:'center'});b.click();return'ok'}
  }
  return 'fail';
}" > /dev/null 2>&1

echo "[8/8] 确认结果..."
sleep 5
playwright-cli screenshot --filename "$WORK_DIR/result.png" > /dev/null 2>&1
echo "✅ 完成！截图: $WORK_DIR/result.png"
```

---

## 坑点速查表（实战踩过）

| # | 问题 | 原因 | 解决 |
|---|---|---|---|
| 1 | `--use-system-ca is not allowed` | NODE_OPTIONS 环境变量冲突 | 所有命令前加 `NODE_OPTIONS=""` |
| 2 | agent-browser 页面空白 | ERR_NO_SUPPORTED_PROXIES | 当前环境代理问题，改用 playwright-cli |
| 3 | 上传图文标签点不上 | 元素在视口外（outside viewport） | 用 JS `querySelector` + `.click()` 代替 ref click |
| 4 | 发布按钮点了没反应 | 悬浮弹窗/遮罩层遮挡 | 先执行 Step 7a 移除所有 overlay/mask/drawer 等 |
| 5 | 误触"加入合集" | 两按钮位置相邻 | JS 精确匹配文本 `t.trim() === '发布'` 且 `!t.includes('合集')` |
| 6 | 找不到输入框 | 上传图片后编辑器还没加载完 | 上传后 sleep 5~8 再 snapshot + fill |
| 7 | ref 失效 | 页面刷新/DOM 重构 | 每次 DOM 变化后重新 snapshot 获取新 ref |
| 8 | "公开可见"点击无反应 | ref 是 playwright-cli 内部标识，不能用 `document.querySelector('[ref=...]')` | 使用 `playwright-cli click <ref>` 命令 |

## 内容格式参考

### 标题示例
```
🛵 2025电动车两轮销量排行榜TOP10
🎬 2026年5月电影票房排行榜
🤖 AI Agent 排行榜 · 2026年6月刊
```
- 带 emoji 吸睛
- < 100 字
- 点明主题 + 时间范围

### 正文示例
```
🛵 2025年两轮电动车销量排行榜出炉！

📊 TOP10品牌排行：
🥇 雅迪 Yadea - 年销超900万辆
🥈 爱玛 Aima - 年销超800万辆
🥉 台铃 Tails - 年销超600万辆
...

✨ 关键洞察：
• 洞察一
• 洞察二
• 洞察三

完整排行数据见配图👆

#电动车 #排行榜 #雅迪 #爱玛
```

- 开头一句话概括
- 用 emoji 编号（🥇🥈🥉 / 1️⃣2️⃣3️⃣）
- 关键洞察 3~5 条
- 结尾引导看配图
- 标签 5~10 个，与内容相关

## 文件路径参考

| 类型 | 路径 |
|---|---|
| 浏览器 Profile（agent-browser） | `/Users/mawenbo/.workbuddy/browser-profiles/xiaohongshu` |
| playwright-cli 登录态 | 使用 `--persistent` 自动保存 |

## 更新记录

- **2026-05-07 v1.1**：新增 Step 6"设置可见性（可选）"，支持设置为"仅自己可见"或"仅互关好友可见"。更新坑点速查表，添加 ref 使用注意事项。更新一键发布脚本模板，添加可见性设置步骤。
- **2026-05-06 v1.0**：基于实战验证创建，完整 7 步流程全部跑通
