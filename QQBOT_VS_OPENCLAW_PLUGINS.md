# qqbot 与 wecom / ddingtalk 及 OpenClaw 官方插件对比

本文对比 **qqbot** 与 **wecom**、**ddingtalk**，并参考 PartMe 官方 **openclaw_*** 插件（openclaw_wecom_kf、openclaw_auth_oauth2、openclaw_cluster、openclaw_ics、openclaw_management、openclaw_mqtt、openclaw_prometheus、openclaw_stomp、openclaw_tracing、openclaw_web_mqtt、openclaw_web_stomp），说明 qqbot 的差异、优势及可借鉴之处。

**约定**：**qqbot 是参考目标，不得修改 qqbot**。其它组件（wecom、ddingtalk、openclaw_*）应**参考 qqbot**，借鉴其 capabilities、skills、**多平台兼容**（openclaw / clawdbot / moltbot）等设计，让其它插件也能兼容多个平台；不要反过来改动 qqbot。

---

## 一、官方 OpenClaw 插件的统一形态

以下 11 个插件在结构和工具链上高度一致，可作为「标准形态」：

| 项目 | 构建 | 产物 | 脚本 | files | openclaw |
|------|------|------|------|-------|----------|
| openclaw_wecom_kf | tsup | dist 单入口 | build/dev/clean/typecheck/test* | dist + plugin + README(+CN) + skills | extensions only |
| openclaw_auth_oauth2 | tsup | dist | 同上 | dist + plugin + README(+CN) | extensions only |
| openclaw_cluster | tsup | dist | 同上 | 同上 | extensions only |
| openclaw_ics | tsup | dist | 同上 | 同上 | extensions only |
| openclaw_management | tsup | dist | 同上 | 同上 | extensions only |
| openclaw_mqtt | tsup | dist | 同上 | 同上 | extensions only |
| openclaw_prometheus | tsup | dist | 同上 | 同上 | extensions only |
| openclaw_stomp | tsup | dist | 同上 | 同上 | extensions only |
| openclaw_tracing | tsup | dist | 同上 | 同上 | extensions only |
| openclaw_web_mqtt | tsup | dist | 同上 | 同上 | extensions only |
| openclaw_web_stomp | tsup | dist | 同上 | 同上 | extensions only |

**共同点**：

- **包名**：`@partme.ai/openclaw_*`
- **构建**：`tsup`，单入口（多为 `src/index.ts`），产出 `dist/index.js` + `dist/index.d.ts`
- **脚本**：`build`(tsup)、`dev`(tsup --watch)、`clean`、`typecheck`、`test`/`test:watch`/`test:coverage`(vitest)
- **files**：`["dist","openclaw.plugin.json","README.md","README_CN.md"]`，部分含 `"skills"`
- **openclaw**：仅 `"openclaw":{"extensions":["./dist/index.js"]}`，无 channel/install 等（这些在 openclaw.plugin.json 或由宿主从 manifest 推断）
- **openclaw.plugin.json**：id、name、version、description、channels、commands（可选）、skills（可选）、configSchema、uiHints（可选）
- **devDependencies**：@types/node ^22、tsup ^8、typescript ^5.7、vitest ^4
- **publishConfig**：`{"access":"public"}`
- **pnpm**：`"onlyBuiltDependencies":["esbuild"]`

---

## 二、qqbot / wecom / ddingtalk 与标准形态对比

| 维度 | 官方 openclaw_*（标准） | ddingtalk | wecom | qqbot |
|------|-------------------------|-----------|-------|--------|
| **包名** | @partme.ai/openclaw_* | @partme.ai/dingtalk | @partme.ai/wecom | @sliverp/qqbot |
| **构建工具** | tsup | tsup | tsc | tsc |
| **产物** | dist 单文件 | dist 单文件 | dist 多文件 | dist 多文件 |
| **入口** | src/index.ts 或单文件 | index.ts | index.ts + src/** | index.ts + src/** |
| **scripts** | build/dev/clean/typecheck/test* | 有 dev/clean/typecheck，无 test | 仅 build/clean/typecheck | build/dev/prepack，无 clean/typecheck/test |
| **files** | dist + plugin + README | dist + plugin + README | dist + plugin + README | dist + **bin** + **src** + **skills** + index.ts + tsconfig + 3×plugin.json |
| **openclaw 字段** | 仅 extensions | extensions + **channel** + **install** | extensions + channel + install（openclaw/clawdbot 双份） | **仅 extensions**，且当前指向 **./index.ts**（应为 ./dist/index.js） |
| **插件描述** | openclaw.plugin.json 较完整 | 极简 plugin.json，channel 在 package | 极简 plugin.json，channel 在 package | **3 个** plugin.json（openclaw/clawdbot/moltbot），channel 在 plugin 内 |
| **宿主** | 仅 OpenClaw | 仅 OpenClaw | OpenClaw + ClawdBot | **OpenClaw + ClawdBot + MoltBot** |
| **bin** | 无 | 无 | 无 | **有**（qqbot-cli.js） |
| **skills** | 部分有（如 wecom_kf） | 无 | 无 | **有**（qqbot-cron、qqbot-media） |
| **capabilities** | 无或 plugin 内 | 无 | 无 | **有**（proactiveMessaging、cronJobs） |
| **prepack** | 无 | 无 | 无 | **npm install --omit=dev** |
| **publishConfig** | 有 | 有 | 有 | **无**（需补） |
| **vitest** | 有 | 无 | 无 | 无 |

---

## 三、qqbot 为何不一样

### 1. 来源与目标不同

- **qqbot**：社区项目（@sliverp/qqbot），设计目标为**多宿主、可独立使用**：
  - 同时支持 **OpenClaw**、**ClawdBot**、**MoltBot**，因此需要三份 plugin 描述（openclaw.plugin.json、clawdbot.plugin.json、moltbot.plugin.json）。
  - 提供 **CLI**（`bin/qqbot-cli.js`），可脱离 OpenClaw 做调试或脚本化使用。
  - 自带 **skills**（qqbot-cron、qqbot-media），在 plugin 的 `skills` 里声明，与 openclaw_wecom_kf 的 `"skills":["skills"]` 类似但更细分。
- **wecom / ddingtalk**：按「单一 OpenClaw 渠道插件」设计（wecom 顺带兼容 clawdbot），无 CLI、无独立 skills 目录，与官方插件的「纯插件」形态更接近。

### 2. 构建与产物

- **官方 + ddingtalk**：**tsup** 单入口打包，只发 **dist**，不发源码。
- **qqbot / wecom**：**tsc** 多文件编译，保留 `dist/index.js` + `dist/src/**` 结构；qqbot 还把 **src**、**index.ts**、**tsconfig.json** 放进 `files`，发布包内含源码，便于多宿主或本地调试。

### 3. 元数据位置

- **官方**：插件能力集中在 **openclaw.plugin.json**（id、channels、commands、skills、configSchema、uiHints），package.json 里 openclaw 只写 `extensions`。
- **wecom / ddingtalk**：openclaw.plugin.json 极简，**channel / install** 等写在 **package.json** 的 `openclaw`（及 wecom 的 `clawdbot`），便于 npm 安装与 OpenClaw 发现。
- **qqbot**：每个宿主一个 **\*.plugin.json**，channel 名、skills、capabilities 都在 plugin 里；package.json 只负责 extensions 和 bin，**且当前 extensions 仍指向 ./index.ts，与发布产物不一致**（应为 ./dist/index.js）。

### 4. 发布与 prepack

- **qqbot** 使用 **prepack: npm install --omit=dev**，在打包前装生产依赖，便于 bin 或运行时依赖；官方插件无 prepack，依赖在安装时由用户/宿主解决。
- **qqbot** 的 **files** 包含 src、skills、多个 plugin.json，面向「多宿主 + 可读源码」的消费方式；官方与 wecom/ddingtalk 只发 dist + 必要配置与文档。

---

## 四、qqbot 的优势

| 优势 | 说明 |
|------|------|
| **多宿主** | 一套实现同时接入 OpenClaw、ClawdBot、MoltBot，降低维护成本，扩大使用场景。 |
| **CLI** | `qqbot` 命令便于本地调试、脚本化、CI 或非 OpenClaw 环境使用。 |
| **内建 skills** | 自带 qqbot-cron、qqbot-media，并在 plugin 中声明，与渠道能力绑定清晰，和 openclaw_wecom_kf 的 skills 思路一致。 |
| **capabilities 声明** | openclaw.plugin.json 中 `proactiveMessaging`、`cronJobs` 等能力显式声明，便于 UI/文档/宿主做能力发现与展示。 |
| **源码随包发布** | 发布包含 src，便于二次开发、排查问题或在不改 node_modules 的前提下做小改动。 |
| **bundledDependencies** | silk-wasm、ws 等关键依赖打包进发布包，减少安装与运行环境差异问题。 |

---

## 五、可借鉴之处（如何借鉴）

### 1. 官方 / wecom / ddingtalk 可向 qqbot 借鉴

- **skills 目录 + plugin 声明**  
  若某渠道插件需要「与渠道强绑定的技能」（如定时任务、媒体处理），可像 qqbot 与 openclaw_wecom_kf 一样增加 `skills/` 目录，并在 openclaw.plugin.json 中写 `"skills":["skills"]` 或具体子路径。
- **capabilities 显式声明**  
  在 openclaw.plugin.json 中增加 `capabilities`（如 `proactiveMessaging`、`cronJobs`、`humanTransfer`），便于控制台或文档自动展示「该渠道支持哪些能力」。
- **可选 CLI**  
  若插件需要独立于 OpenClaw 的调试或脚本能力，可增加 `bin` 入口（参考 qqbot-cli），并在 package.json 中声明。
- **多平台兼容（多宿主）**  
  目标：**让其它插件也能像 qqbot 一样兼容多个平台**。qqbot 同时支持 OpenClaw、ClawdBot、MoltBot（三份 plugin.json：openclaw.plugin.json、clawdbot.plugin.json、moltbot.plugin.json；package.json 中为各宿主声明 extensions）。其它插件若需多平台兼容，可参考 qqbot 维护多份 \*.plugin.json 或在 package.json 中为各宿主写 extensions/channel/install（如 wecom 已双写 openclaw/clawdbot）。

### 2. 关于 qqbot 本身（不修改）

qqbot 保持现有形态，**不做“对齐”式修改**。其 extensions 指向 `./index.ts`、发布含 src、无 publishConfig 等，均为其多宿主/CLI/可调试设计的一部分。其它插件若需“只发 dist、统一 registry”，在各自 package 中配置即可，不反向改动 qqbot。

---

## 六、小结

- **为何 qqbot 不一样**：社区多宿主设计、带 CLI 与 skills、tsc 多文件构建、发布含源码、三份 plugin 与 capabilities 声明，和「单一 OpenClaw 渠道 + 单文件 dist」的官方形态不同。
- **qqbot 的优势**：多宿主、CLI、内建 skills、capabilities 声明、可选源码发布、bundledDependencies，适合需要扩展性和可运维性的场景。
- **借鉴方向**：**其它组件参考 qqbot**，借鉴 skills、capabilities、**多平台兼容**（多宿主）等，使其它插件也能兼容多个平台；**qqbot 是参考目标，不修改 qqbot**，不对其做 extensions/publishConfig/files 等任何“对齐”式修改。
