<div align="center">

# B站竖刷 BiliShorts

**像刷抖音一样刷B站网页版 —— 让B站网页获得客户端级竖屏体验的浏览器扩展**

[![Version](https://img.shields.io/badge/version-2.4.0-fb7299)](#-功能一览)
[![Manifest](https://img.shields.io/badge/Manifest%20V3-00a1e0)](#-安装)
[![Edge](https://img.shields.io/badge/Edge-支持-0078d7)](#-安装)
[![Chrome](https://img.shields.io/badge/Chrome-支持-4285f4)](#-安装)

**滚轮 / 方向键无限切换 · DASH 1080P+ 硬解 · 实时弹幕 · 个性化推荐流 · 点赞 / 投币 / 收藏 · 评论侧边栏**

</div>

---

## ✨ 这是什么

打开任意一个B站视频页，按 `Alt+V`，整个页面变成全屏沉浸的竖屏信息流 —— 鼠标滚轮或方向键上下切换视频，像刷B站客户端 / 抖音网页版一样。它不是把视频嵌进自家播放器的"套壳"，而是**完整保留了B站网页端的能力**：画质跟随你的账号（1080P/1080P60/4K）、官方弹幕池实时渲染、点赞/投币/收藏真实写入账号、评论区可展开阅读，推荐内容与你的B站首页一致。

## 🎬 功能一览

| 模块 | 说明 |
| --- | --- |
| 🖱️ 竖屏信息流 | 滚轮 / ↑↓ / 触屏滑动切换，平滑转场动画，无限队列 |
| 📺 高清播放 | DASH + MSE 流式播放，画质跟随账号（1080P+/4K），右上角随时切换；播放异常自动回退 720P 直连 |
| 💬 实时弹幕 | 官方分段弹幕接口（protobuf），canvas 渲染，滚动/底部/顶部、多车道防碰撞、随进度自动对齐，`B` 键开关 |
| 🔔 评论侧边栏 | 官方评论接口，热评置顶、上滑翻页、跟随视频自动刷新，`X` 键开关 |
| ❤️ 互动 | 点赞 / 投币（1币）/ 收藏到默认收藏夹，均写入真实账号，自动识别已赞/已投/已藏状态 |
| 🔔 个性化推荐 | 队列来自B站首页同款推荐接口，相关视频与热门榜兜底 |
| ⚙️ 设置 | 默认画质、画面填充（完整/铺满）、弹幕开关、播完自动连播（sync 存储） |

## 🚀 安装

1. `git clone` 或下载本仓库到**长期保留**的目录
2. 打开 `edge://extensions/`（Chrome 为 `chrome://extensions/`）
3. 开启**开发人员模式**
4. 「加载解压缩的扩展」→ 选择本仓库根目录（`manifest.json` 所在层）
5. 打开B站视频页，`Alt+V` 开始

> 登录 bilibili.com 后自动获得高画质与互动能力；未登录也可用（360P、无弹幕）。

## ⌨️ 快捷键

| 按键 | 功能 | 按键 | 功能 |
| --- | --- | --- | --- |
| `Alt+V` | 开/关竖屏模式 | `M` | 静音 |
| `↓` `↑` / 滚轮 | 切换视频 | `B` | 弹幕开关 |
| `空格` / `K` | 播放/暂停 | `X` | 评论侧边栏 |
| `←` `→` | ±5 秒 | `F` | 全屏 |
| `J` `L` | ±10 秒 | `C` | 画面填充切换 |
| `Esc` | 退出全屏/竖屏 | | |

## 🧩 技术实现

想了解或参与开发，这里是项目的核心设计：

- **自研 MSE 播放器（无第三方依赖）**
  - `playurl(fnval=16)` 获取 DASH 流，逐段拉取 fMP4 喂给 `SourceBuffer`
  - 首帧优化：init 段渐进获取 → 首段 1MB 快速起播 → 8MB 后切 4MB 大分片
  - **Seek 实现**：无 sidx 索引下，按 `size/duration` 估算字节位 → 256KB 窗口扫描 `moof` → 解析 `tfdt` 时间戳定位分片；前进 seek 直接续流、后退 seek 清缓冲重对齐 `timestampOffset`
  - 双轨（音/视频）独立喂流与超时容错，失败自动回退 `platform=html5` 直连 MP4
- **WBI 签名（页面侧实现）**
  - 内置 MD5（RFC 1321，含 UTF-8），`nav` 拿 `img_key+sub_key` 经 64 位混排表取 32 位 mixinKey
  - 参数排序 + `wts` + `w_rid`，用于 playurl / 评论 / 弹幕等接口
- **弹幕引擎**
  - 官方分段接口 `dm/wbi/web/seg.so`（protobuf）+ 轻量手写 **protobuf wire-format 解析器**（全 wire type 防御式）
  - 携带 `dm_img_list` 等设备风控参数（2023 后缺失即 412）
  - canvas 渲染：车道分配、碰撞检测、固定/滚动模式、时间轴对齐、DPR 适配
- **推荐流**：`index/top/feed/rcmd`（个性化）→ 相关视频 → 热门榜，三级兜底
- **风控规避**：写操作（点赞/投币/收藏）与部分 GET 从**页面上下文直连**发起（Referer/Origin 与网页端一致，避免后台请求被 412），WBI 签名在页面侧完成；后台 Service Worker 仅承担无签名 GET 代理与快捷键
- **UI**：Shadow DOM 完全隔离样式；评论面板/清晰度菜单/Toast 均在 shadow 内实现

## 📁 目录结构

```
bili-shorts/
├── manifest.json      # MV3 清单
├── background.js      # Service Worker：接口代理、WBI 签名、弹幕解压、快捷键
├── content.js         # 核心：竖屏 UI、MSE 播放器、弹幕引擎、推荐队列、互动、评论
├── popup.html/js      # 弹窗：开关与设置（画质/弹幕/填充/连播）
├── gen_icons.ps1      # 图标生成脚本（开发用）
├── icons/             # 扩展图标
└── README.md
```

## 🧗 踩坑记录（欢迎参考/补充）

开发过程中确认过的几个"隐性规则"，对同类项目应该有帮助：

<details>
<summary><b>扩展目录里不能有 <code>_</code> 开头的文件</b></summary>

Chromium 保留 `_<name>` 命名，目录内任何 `_` 开头的文件都会让整个扩展拒绝加载，报 `Cannot load extension with file or directory name _xxx`。调试产物请移出扩展目录。
</details>

<details>
<summary><b>MSE 黑屏：init 段必须先喂</b></summary>

DASH 的 fMP4 若直接从 moof 碎片开始 append 而不先喂 init（ftyp+moov），SourceBuffer 会静默失败，表现为永久黑屏。且 init 段在 seek 后清缓冲重播时需要**重新 append** 并重设 `timestampOffset`。
</details>

<details>
<summary><b>写操作 412：必须在页面上下文发起</b></summary>

B站对点赞/投币/收藏等写接口校验 Referer/Origin，扩展 Service Worker 发起的请求即使带 Cookie 也会被 412。解法是"后台只签名，页面直连发请求"。
</details>

<details>
<summary><b>新弹幕接口的两道门槛</b></summary>

老 XML 接口 `dm/list.so` 已被风控（412）；新接口 `dm/wbi/web/seg.so` 除 WBI 签名外还要求 `dm_img_list` / `dm_img_str` / `dm_cover_img_str` / `dm_img_inter` 设备参数（可用固定伪造值），且返回 protobuf（字段 1=弹幕数组；7=文本；2=时间ms；3=模式；5=颜色）。
</details>

<details>
<summary><b>无 WBI 签名的 playurl 上限 720P</b></summary>

老 `x/player/playurl` 对无签名请求只发 ≤720P；带 WBI 签名的 `x/player/wbi/playurl` 才会按账号权益下发 1080P+/4K 的 DASH 流。
</details>

<details>
<summary><b>推荐流接口的字段名差异</b></summary>

`index/top/feed/rcmd` 的视频 id 字段是 <code>id</code>（非 <code>aid</code>），收藏数字段是 <code>fav</code>（非 <code>favorite</code>）——直接按 view 接口的字段名取值会拿到 undefined。
</details>

## ❓ 常见问题

- **按 Alt+V 没反应？** 到 `edge://extensions/shortcuts` 检查快捷键是否被占用；或点工具栏「扩展」拼图菜单里的 B站竖刷
- **没有声音？** 浏览器自动播放策略限制，点右上角喇叭或按 `M`
- **个别视频加载失败？** 充电专属/地域限制视频不支持网页直连，扩展会自动回退或提示「打开原视频」
- **每次启动浏览器弹"是否关闭开发人员模式扩展"？** 选「保留」即可（开发者模式的例行提醒）

## 📜 免责声明

本项目仅供学习交流使用，仅调用B站公开 Web 接口，所有行为与你正常刷B站一致；不破解、不下载、不绕过任何付费或权限限制。B站接口随时可能调整导致功能失效。请勿用于商业用途，因使用本项目产生的任何问题由使用者自行承担。

## 🤝 贡献

欢迎 Issue / PR：接口失效修复、新功能（弹幕发送、楼中楼、多收藏夹选择、倍速…）、性能优化都欢迎。
