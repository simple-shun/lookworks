# Ollama 与 qwen3:8b 安装验证指南

适用设备：MacBook Pro，Apple M4 Pro，48GB 统一内存，macOS 26.2。文档核对日期：2026 年 10 月 21 日。

目标：手动安装 Ollama，下载 `qwen3:8b`，分别验证命令行、本地服务、模型文件和实际推理。本文是待执行的操作指南；生成文档时没有替你安装软件或下载模型。

## 一 先理解要安装什么

- **Ollama**：运行模型的软件，提供命令行和本地 HTTP 接口。
- **qwen3:8b**：本次使用的模型。`qwen3` 是模型名称，`8b` 是规格标签。
- **本地服务**：你的程序通过 `http://localhost:11434` 调用 Ollama。

你的设备满足 Ollama 对 macOS 14 及以上、Apple M 系列芯片的要求。采用 macOS 原生应用安装即可。[系统要求](https://docs.ollama.com/macos)

该模型当前默认下载包约 5.2GB，量化格式为 Q4_K_M。文件大小不等于运行内存占用，运行时还有上下文缓存等开销。建议为这次练习预留至少 15GB 磁盘空间；此前检查本机约有 210GiB 可用空间。[模型详情](https://ollama.com/library/qwen3:8b)

本地调用不需要购买云端 API 或设置 API Key；下载应用和模型需要网络。[官方快速入门](https://docs.ollama.com/quickstart)

## 二 安装 Ollama

1. 打开 [Ollama 官方 macOS 下载页面](https://ollama.com/download/mac)。
2. 下载 macOS 安装包，打开下载的磁盘映像，将 **Ollama** 拖入 **Applications（应用程序）**。
3. 从“应用程序”启动 Ollama，按照首次启动提示完成设置。
4. 如果提示安装命令行或创建 `/usr/local/bin/ollama` 链接，允许该操作。
5. 打开一个新的“终端”窗口，进行下面的验证。

上述应用安装与命令行链接方式来自 [macOS 安装说明](https://docs.ollama.com/macos)。首次使用先保持 Ollama 应用运行。

### 验证命令行可用

在终端执行：

```bash
command -v ollama
ollama --version
```

成功标志：第一条输出可执行文件路径，第二条输出版本号。具体路径和版本号可以与示例不同。

如果出现 `command not found`，先查看第七节；如果提示无法连接服务，继续下一步检查服务。

### 验证本地服务可连接

```bash
curl --fail --show-error http://localhost:11434/api/tags
```

成功标志：返回 JSON。尚未下载模型时，可以是：

```json
{"models": []}
```

这表示服务正常，但还没有模型。`/api/tags` 用于列出本地模型。[接口文档](https://docs.ollama.com/api/tags)

## 三 下载 qwen3:8b

保持 Ollama 运行，在终端执行：

```bash
ollama pull qwen3:8b
```

等待下载和校验完成，通常最后显示 `success`。耗时取决于网络速度；这一步是在下载模型权重，不是在训练模型。

下载完成后执行：

```bash
ollama ls
```

成功标志：列表中出现 `qwen3:8b`，大小约 5.2GB。模型 ID、更新时间和精确大小以本机输出为准。[命令行参考](https://docs.ollama.com/cli)

模型默认存储在用户目录的 `~/.ollama` 下，不会因为你从当前项目目录执行命令，就下载到项目中。[存储位置说明](https://docs.ollama.com/macos)

## 四 验证模型真的能回答

在终端执行：

```bash
ollama run qwen3:8b
```

出现交互输入提示后，输入：

```text
请用中文简短介绍什么是大语言模型，并计算 12 + 30。
```

成功标志：生成非空回答，能进行中文解释，并给出计算结果 42。回答措辞不必完全相同；如果只有个别知识回答错误，不能据此判定安装失败。

首次请求需要加载权重，可能比后续请求慢。模型也可能先输出思考内容，等待最终回答即可。

退出对话时，在交互提示处输入：

```text
/bye
```

回到终端后，马上执行：

```bash
ollama ps
```

该命令显示当前加载的模型。留意 `PROCESSOR` 一列是否显示 GPU 使用情况；它有助于检查加速情况。若等待较久后列表为空，模型可能已自动卸载，重新运行一次对话后再查看。`ollama ls` 查看已下载模型，`ollama ps` 查看当前加载模型，两者含义不同。[命令行参考](https://docs.ollama.com/cli)

## 五 验证应用程序可以调用模型

这一步验证以后写 RAG 和 Agent 时会用到的本地 HTTP 调用。将整段命令复制到终端执行：

```bash
curl --fail --show-error http://localhost:11434/api/chat \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen3:8b",
    "messages": [
      {"role": "user", "content": "请用中文一句话解释什么是 RAG。"}
    ],
    "stream": false,
    "think": false
  }'
```

参数含义：`model` 指定模型；`messages` 提供对话；`stream: false` 等待完整 JSON 返回；`think: false` 请求关闭思考输出，让这次验证更直接。[聊天接口](https://docs.ollama.com/api/chat)、[思考开关](https://docs.ollama.com/capabilities/thinking)

成功返回的结构大致如下，实际回答和其他字段可能不同：

```json
{
  "model": "qwen3:8b",
  "message": {
    "role": "assistant",
    "content": "RAG 是先检索相关资料，再结合资料生成回答的方法。"
  },
  "done": true
}
```

检查三件事：没有 HTTP 错误；`message.content` 非空；`done` 为 `true`。这说明本地服务已完成一次模型生成请求，不等于证明所有回答都正确。[返回字段说明](https://docs.ollama.com/api/chat)

## 六 安装完成检查表

逐项完成后勾选：

- [ ] `command -v ollama` 能找到命令。
- [ ] `ollama --version` 能显示版本。
- [ ] `/api/tags` 返回 JSON。
- [ ] `ollama ls` 包含 `qwen3:8b`。
- [ ] `ollama run qwen3:8b` 能生成中文回答。
- [ ] `/api/chat` 返回非空 `message.content` 和 `done: true`。
- [ ] 已查看 `ollama ps`，了解模型是否在使用 GPU。

前六项通过即可开始本地应用练习。最后一项用于了解运行状态与性能。

可在这里记录本机结果：

| 项目 | 你的记录 |
| --- | --- |
| 安装日期 | |
| Ollama 版本 | |
| 模型名称和 ID | |
| 命令行对话结果 | |
| API 调用结果 | |
| PROCESSOR 显示 | |
| 遇到的问题 | |

## 七 常见问题

### 找不到 ollama 命令

重新打开 Ollama 和新的终端窗口，确认完成了命令行安装提示。仍失败时执行：

```bash
/Applications/Ollama.app/Contents/Resources/ollama --version
```

如果绝对路径能运行，说明应用已安装，问题在命令行链接或 PATH。可以先用该绝对路径替换本文命令里的 `ollama` 完成练习，再修复命令行入口。如果绝对路径也不存在，检查应用是否确实放在“应用程序”目录。[CLI 文件位置](https://docs.ollama.com/macos)

### 无法连接 localhost 的 11434 端口

先从“应用程序”打开 Ollama，再重试服务验证。也可以用终端启动应用：

```bash
open -a Ollama
```

如果应用仍未启动服务，可在单独的终端窗口执行：

```bash
ollama serve
```

此时保留该窗口，在另一个终端进行验证。若提示 `address already in use`，通常已有进程占用端口；先重试 `/api/tags`，不要重复启动服务。[服务启动命令](https://docs.ollama.com/cli)

### 模型下载失败或中断

检查网络连接和磁盘剩余空间，恢复后重新执行 `ollama pull qwen3:8b`。保留具体错误信息，区分网络超时、证书问题和磁盘不足；不要因为一次失败就删除整个模型目录。

### 提示模型不存在

先运行 `ollama ls`，检查是否已经下载，以及模型名称是否为 `qwen3:8b`。没有该项就重新执行下载命令，再重试调用。

### 首次回答慢或一直输出思考

先等待权重加载完成，再用第五节关闭思考的 API 示例测试。关闭暂时不用的大型程序，用短问题验证，并查看 `ollama ps`。不要把首轮加载时间当成每次请求的固定耗时。

### API 返回很多行 JSON

检查请求中是否包含 `"stream": false`。默认流式模式会分多次返回内容；第五节使用非流式输出便于人工核对。[stream 参数说明](https://docs.ollama.com/api/chat)

### 需要查看服务错误日志

```bash
tail -n 80 ~/.ollama/logs/server.log
```

如果文件不存在，先确认 Ollama 已启动。macOS 应用日志通常位于 `~/.ollama/logs`。[日志位置](https://docs.ollama.com/macos)

## 八 学习结束后如何释放内存

```bash
ollama stop qwen3:8b
```

这会卸载当前运行的模型以释放内存，保留已下载文件。下次执行 `ollama run qwen3:8b` 即可重新加载。[命令行参考](https://docs.ollama.com/cli)

完成安装后，适合做的第一个小实验是：用同一个问题调用三次 `/api/chat`，观察回答变化，并区分“服务调用成功”和“回答事实正确”这两个验收标准。
