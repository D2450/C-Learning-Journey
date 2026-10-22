# 强行适配的浪漫：用 Agent 修复 Agent 的版本地狱

> **摘要**：本文记录了一次跨版本 DSH 插件的兼容性排障过程。为了给日常环境隔离风险，我搭建了独立沙箱，并采用“双 Agent 架构”（我作为决策者，工作用 DSH 作为代码执行者），最终花费约 1 小时(可能更多)、2 元 API 成本，成功将老版本插件适配至新内核。

## 一、 项目背景与初始困境

我是机器人工程专业的学生。为了研究 AI 角色扮演和记忆机制，我使用 DeepSeek Harness（DSH）作为底座，并尝试接入一个名为 Pygmalion 的开源聊天插件。

该插件依赖较老的 DSH 内核接口，而我的主力环境使用的是较新的版本。为了避免污染主力开发环境，我开辟了一个独立的沙箱：独立的 DSH 实例、独立的 home 目录与端口。

然而，环境搭好后，QQ 机器人虽然能正常接收消息并写入日志，但始终没有回复。(因为本人暂未对这些了如指掌，所以就请DeepSeek大人来了)查阅日志，发现核心报错指向了旧版 API：

```text
[qqbot-community] ERROR inbound handling failed (bot-role-xxxxx):
TypeError: this.deps.sessionPersistence.inspect is not a function
    at file:///C:/<Your-Sandbox-Path>/node_modules/pygmalion/src/inbound.js:620:62
```

在 **inbound.js** 第 619 行，插件调用了旧版的 **sessionPersistence.inspect** 方法：

```javascript
const persisted = this.deps.sessionPersistence !== undefined
  ? await this.deps.sessionPersistence.inspect(sessionId).then(() => true).catch(() => false)
  : false;
```

新内核中，该方法已被移除，替代的接口变成了 create 、pen 、 flush 、 stat 、 list。同时，旧代码存在一个隐患：当 inspect 不存在时，抛出的是同步的 TypeError，导致 .catch 无法拦截，直接导致消息处理流程死亡。（所以才说ai方便）

## 二、 排障策略：引入 AI 辅助

面对版本不兼容，通常有两种方案：
1. 降级主环境：风险极大，否决。
2. 修改插件源码，强行适配新内核：在沙箱中操作，试错成本低。

我选择了方案 2。但面对庞大的底层源码，纯手搓效率太低。因此，我采用了<strong>“人类决策 + AI 执行”的结对编程模式</strong>。我将平时工作用的 DSH 作为副驾驶，让它阅读旧版插件的源码、对比新内核的接口定义、生成补丁代码，而我负责执行命令、验证结果和提出新的问题。

**让 Agent 去修复 Agent 的兼容性，是这次排障中最核心的技术手段。**

## 三、 连环 Bug 的拆解与修复

在修改过程中，我遭遇了连环报错，以下是核心的修复记录（均由我定位方向，AI 生成具体修改代码）：

### 1. 会话状态查询接口变更
因为插件使用 pnpm 安装，源文件是硬链接，直接修改会影响全局缓存。就先断开了硬链接：
```powershell
Copy-Item inbound.js backup\inbound.js.orig
$tmp = "inbound.js.tmp"
Copy-Item inbound.js $tmp
Remove-Item inbound.js
Move-Item $tmp inbound.js
```
随后，将 inspect 替换为异步的 stat：
```javascript
// 旧写法
const persisted = await this.deps.sessionPersistence.inspect(sessionId).then(() => true).catch(() => false);

// 新写法（防御性处理）
const persisted = this.deps.sessionPersistence !== undefined
  ? await this.deps.sessionPersistence.stat(sessionId).then((s) => s !== undefined).catch(() => false)
  : false;
```

### 2. 预设机制重构
```text
RemoteError: Unknown agent preset: pygmalion-director
```
新内核不再扫描 .agent-presets 文件夹，而是采用声明式注册。我让 AI 辅助分析了新的 YAML 结构，将旧预设中的 11 个插件迁移至 profiles/web/cordis.patch.yml，并修复了桥接脚本的相对路径。

### 3. 配置字段更名
```text
persona: invalid config: - $.prefix missing required value (at prefix)
```
以前叫 text，现在改为 prefix。

### 4. 插件遗留 Bug
```text
ReferenceError: httpApiPath is not defined
    at InboundPipeline.setupAgent (setup-agent.js:77:17)
```
在 setup-agent.js 中，解构依赖时遗漏了 httpApiPath，导致后续调用报错。

### 5. 模型提供方 ID 变更
```json
{"message":"no adapter registered for provider \"deepseek\"","code":"NO_ADAPTER"}
```
新内核中，Provider ID 变更为 **deepseek-official** 和 **deepseek-account**。我让 AI 帮我全局搜索了角色配置，替换了旧的 **deepseek-chat** 等字样。

### 6. 扩展预处理的隐蔽报错
```json
{"message":"DeepSeek request extension preparation failed","code":"REQUEST_EXTENSION"}
```
通过阅读内核源码发现，插件在寻找相对入口 **./pygmalion-tools.js** 时，需要读取最近的 **package.json** 来校验版本，但目标目录下的文件缺失了 **version** 字段。

### 7. 工具注册状态丢失
```text
tool/result: Error: unknown tool "director_send"
```
这是一个进程级布尔变量引发的状态不同步。我让 AI 协助重构了 **setup-agent.js** 和 **tools.js** 的注册逻辑，将其改为幂等操作，并在会话恢复时强制补发注册。

## 四、 成本核算与数据总结

*   **耗时**：约 1 小时（主要集中在重启验证与定位报错）。
*   **API 费用**：约 2 元人民币（用于让工作 DSH 解析日志与生成 diff）。
*   **代码改动**：5 个源文件 + 2 个配置文件。
*   **备份**：保留 6 份原始文件备份于 **backups** 目录，支持随时回滚。

## 五、 反思与后记

这次排障让我真正理解了两代 API 在同一个进程里冲突的本质，也让我看到了 AI 辅助开发的真正价值：**它最大的作用不是替你写代码，而是极大地压低了“阅读和解析大型源码”的时间成本。** 几十万字节的日志、复杂的接口定义，它能迅速定位到冲突行。

当然，AI 只是工具，决策权和架构设计依然在人脑。为了保证后续学习的深度，**关于这里涉及的 DSH 底层机制、插件运行原理以及具体每一项修复的代码逻辑分析，我将另外整理一篇附属文档（原理解析篇）进行记录。**
~~不用在意这一行，ai帮助固然好，可是我也希望我自己就能够做到~~
