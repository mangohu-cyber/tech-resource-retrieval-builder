# tech-resource-retrieval-builder

面向嵌入式、机器人、计算机视觉、AI 工程类项目的**技术资料检索策略生成器**。

给它一套系统架构，它还你一份可以直接复制去搜的检索方案：子模块怎么拆、每块用什么词、哪些方案要排除、按什么顺序学。需要的话，还可以联网把资料搜出来，逐条告诉你为什么选它、为什么淘汰它。

## 解决的问题

做嵌入式 / 机器人项目时，最耗时的往往不是写代码，而是**搜错方向**：

- 搜"ESP32 接摄像头"，出来全是 ESP32 本地跑 YOLO 的教程，而你的模型明明跑在 PC 上；
- 搜"串口通信"，一半结果是 MQTT 和 WiFi；
- 好不容易找到一篇，是 ESP-IDF 框架，你用的是 Arduino；
- 教程代码能跑，但库版本是三年前的。

这个 Skill 把"检索"当成一项工程任务来处理：**先定架构和接口，再拆模块，最后才生成搜索词**，并且把不想要的方案写进检索式里直接过滤掉。

## 核心设计

### 1. 策略与结果解耦

检索策略是静态推理产物，不依赖网络、永久可复用；链接和实时结果是附加产物，天然具备不确定性和失效风险。两者分开产出，网页挂了策略还在。

### 2. 两种运行模式

| 模式 | 行为 | 适用 |
| --- | --- | --- |
| **A 仅策略** | 只输出检索方案，不联网、不给链接 | 想自己搜、想沉淀方法论、无网络工具 |
| **B 策略 + 检索结果** | 策略 + 实际搜到的资料 + 逐条入选/排除理由 | 想直接拿到可用资料 |

模式 B 会先给出风险提示，并对每条结果标注：归属哪个子模块、解决什么问题、来源站点、更新时间、库版本。搜索工具不可用时会如实告知并降级为模式 A，**不会编造链接**。

### 3. 先定接口契约，再定检索词

大多数联调失败不是代码问题，而是协议没对齐：波特率、`\n` 还是 `\r\n`、字符编码、有没有 ACK。所以流水线在拆模块之前会先为每一对相邻模块写明接口契约，并直接把它转成搜索词。

### 4. 负向词的克制使用

排除项**逐条 query 局部使用**，不做全局拼串——串口查询不该背摄像头相关的排除词。同时优先用**正向锚定**（`arduino-esp32`、`Wire.h`、`ultralytics`）来锁定范围，减号只留给该查询内高频污染词。

## 执行流水线

```text
Step 1  架构复述 + 负向词提取（明确项 / 推断项分级）
   ↓
Step 2  接口契约（物理层 / 帧格式 / 行结束符 / 编码 / 方向 / 防抖）
   ↓
Step 3  子模块拆分 + 三类检索词（精确 / 迁移 / 排坑），中英双语
   ↓
Step 4  按引擎分开的高级搜索串（Google、百度、B站语法不通）
   ↓
Step 5  按子模块领域生成白名单 + 动态质检规则
   ↓
Step 6  预算与止损（每模块 ≤3 条 query，15 分钟无进展换策略）
```

## 输入示例

> PC 上位机 USB 摄像头采集，PC 跑 YOLO，pyserial 串口下发指令，ESP32 Arduino 接收；排除 ESP32 本地部署 YOLO、OV2640、ESP-IDF

需要提取的 4 个字段：整体数据流、**模型部署位置**（最关键）、通信链路、框架与排除清单。

## 输出示例（节选）

```text
🔹 子模块拆分
- A. ESP32 Arduino 串口收发
- B. Python pyserial 串口通信、处理换行、乱码、帧分隔
- C. YOLO + OpenCV USB 摄像头实时检测，筛选 cat 类别
- D. 链路整合：YOLO 检测目标后串口下发指令

🔹 检索词组【模块 B】
- 精确匹配词（中）：pyserial 串口通信 收发 教程
                  （英）：pyserial readline write example
- 迁移参考词（中）：Python 串口 控制 Arduino
- 故障排坑词（英）："UnicodeDecodeError: 'gbk' codec can't decode"

🔹 搜索串
- Google / GitHub：`pyserial readline arduino -"ESP-IDF"`
- 百度：`pyserial 串口通信 教程`
- B站：`pyserial 串口 教程`
```

## 安装

### Claude Code

把仓库克隆到技能目录：

```bash
git clone https://github.com/mangohu-cyber/tech-resource-retrieval-builder.git \
  ~/.claude/skills/tech-resource-retrieval-builder
```

Windows 下（本机使用 `D:\Skills`）：

```powershell
git clone https://github.com/mangohu-cyber/tech-resource-retrieval-builder.git D:\Skills\tech-resource-retrieval-builder
```

重启会话或执行 `/skills` 确认加载。

### 其他 Agent 框架

`SKILL.md` 是纯 Markdown + YAML frontmatter，`description` 字段负责触发，正文是行为规则。任何支持按需加载指令文件的框架都可以直接使用。

## 目录结构

```text
tech-resource-retrieval-builder/
├── SKILL.md    # 技能本体：触发条件、流水线、输出格式、约束
└── README.md   # 本文件
```

## 适用边界

**适用**：多组件系统架构的检索规划、生成搜索词、排除干扰方案、学习工程资料检索方法。

**不适用**：单一 API 用法、单一报错排查、非工程技术类问题（这些直接回答更快，没必要套流程）。

判别口径：**组件数 ≥ 2，且用户要的是"去哪儿找"而不是"答案是什么"**。

## 版本

| 版本 | 变更 |
| --- | --- |
| v1.1 | 负向词局部化与正向锚定；引擎语法分栏；强制双语 query；新增接口契约步骤；质检规则动态生成；白名单按领域；新增预算止损、降级、自检 |
| v1.0 | 初版：策略生成 + 可选联网 + 可解释筛选 |
