# Claude Code 图片与媒体文件内容识别机制详细分析

> 基于源码深度研究，覆盖 `src/tools/FileReadTool/`、`src/utils/imageResizer.ts`、`src/utils/imagePaste.ts`、`src/utils/imageStore.ts`、`src/utils/pdf.ts`、`src/tools/BashTool/utils.ts`、`vendor/image-processor-src/` 等模块。

---

## 目录

1. [整体架构 - 多媒体内容进入 Claude 的通路](#1-整体架构---多媒体内容进入-claude-的通路)
2. [图片处理核心管线](#2-图片处理核心管线)
3. [图片输入的五条路径](#3-图片输入的五条路径)
4. [PDF 文档处理](#4-pdf-文档处理)
5. [视频与音频处理 — 不直接支持](#5-视频与音频处理--不直接支持)
6. [二进制文件检测与分流](#6-二进制文件检测与分流)
7. [原生图片处理引擎 (image-processor-napi)](#7-原生图片处理引擎-image-processor-napi)
8. [API 限制与压缩策略](#8-api-限制与压缩策略)
9. [图片存储与生命周期管理](#9-图片存储与生命周期管理)
10. [关键设计模式总结](#10-关键设计模式总结)

---

## 1. 整体架构 - 多媒体内容进入 Claude 的通路

Claude Code 的图片/媒体识别**本质上是将文件内容转换为 Anthropic API 可接受的 ContentBlockParam**（`image` 或 `document` 类型），然后由 Claude 模型自身的视觉能力进行"识别"。Claude Code 本身不做 OCR 或视觉推理——它只做**格式转换、压缩、尺寸调整**，然后交给模型。

```
                    ┌─────────────────────────────────────────┐
                    │          用户输入（5 条通路）               │
                    ├──────────┬──────────┬──────────┬────────┬────────┤
                    │ ctrl+v   │ 拖拽文件  │ @附件    │ 工具调用│ Bash   │
                    │ 粘贴图片  │ 到终端    │ 引用图片  │ 读文件  │ 输出图片│
                    └────┬─────┴────┬─────┴────┬─────┴───┬────┴───┬────┘
                      │          │          │         │        │
                      v          v          v         v        v
                    ┌────────────────────────────────────────┐
                    │ imagePaste / usePasteHandler /         │
                    │ attachments / FileReadTool / BashTool  │
                    └──────────────────┬─────────────────────┘
                                       │
                                       v
                    ┌──────────────────────────────────────────┐
                    │          图片处理管线                      │
                    │  1. 格式检测 (magic bytes)                │
                    │  2. 尺寸/大小压缩 (sharp/napi)           │
                    │  3. Base64 编码                           │
                    │  4. API 限制校验                          │
                    └──────────────────┬───────────────────────┘
                                       │
                                       v
                    ┌──────────────────────────────────────────┐
                    │      Anthropic API (ImageBlockParam)     │
                    │      type: 'image'                       │
                    │      source: { type: 'base64', data, ... }│
                    └──────────────────────────────────────────┘
                                       │
                                       v
                    ┌──────────────────────────────────────────┐
                    │      Claude 模型 Vision 能力              │
                    │      (模型内部视觉理解，非 Claude Code)    │
                    └──────────────────────────────────────────┘
```

### 1.1 支持的媒体类型

| 类型 | 支持格式 | 处理方式 |
|------|---------|---------|
| **图片** | PNG, JPEG, GIF, WebP | 直接支持，通过 `ImageBlockParam` 发送给模型 |
| **PDF** | .pdf | 整档读取时通常作为 `DocumentBlockParam`；显式传入 `pages` 时拆成页面图片 |
| **视频** | MP4, MOV, AVI 等 | **不支持** — 作为二进制文件被 FileReadTool 拒绝 |
| **音频** | MP3, WAV 等 | **不支持** — 作为二进制文件被 FileReadTool 拒绝（但有独立的语音输入通路） |
| **BMP** | .bmp | 不是 Read 工具支持格式；仅在剪贴板/路径读取的局部路径中会转成 PNG |

> 源文件: `src/constants/files.ts` — `BINARY_EXTENSIONS` 集合定义了所有二进制扩展名。

---

## 2. 图片处理核心管线

### 2.1 核心函数: `maybeResizeAndDownsampleImageBuffer()`

> 源文件: `src/utils/imageResizer.ts`

这是整个图片处理子系统的核心，所有路径最终都调用此函数。

```
输入: (imageBuffer: Buffer, originalSize: number, ext: string)
输出: Promise<{ buffer: Buffer, mediaType: string, dimensions?: ImageDimensions }>

处理流程:
  1. 空 buffer 检查 → 抛出 ImageResizeError
  2. 通过 sharp 读取 metadata (width, height, format)
  3. 若无法获取尺寸且 > 3.75MB → 直接 JPEG 80 压缩
  4. 若尺寸 + 大小都在限制内 → 原样返回
  5. 若尺寸在限制内但大小超限:
     a. PNG: 先尝试 PNG 压缩级别 9 + 调色板
     b. 然后尝试 JPEG 质量 80→60→40→20
  6. 若尺寸超限 → 等比缩小到 2000x2000 以内
  7. 缩小后仍超大小 → 重复步骤 5
  8. 终极手段 → 缩到 1000px + JPEG quality=20
  9. catch: 检测魔数识别格式，若 base64 < 5MB 则放行
  10. 都失败 → 抛出 ImageResizeError
```

### 2.2 格式检测: Magic Bytes

> 源文件: `src/utils/imageResizer.ts` — `detectImageFormatFromBuffer()`

通过文件头魔数（而非扩展名）检测真实格式:

| 格式 | 魔数 (hex) | 说明 |
|------|-----------|------|
| PNG | `89 50 4E 47` | PNG 标准签名 |
| JPEG | `FF D8 FF` | JPEG SOI 标记 |
| GIF | `47 49 46` (GIF) | GIF87a 或 GIF89a |
| WebP | `52 49 46 46 ... 57 45 42 50` | RIFF container + WEBP |
| BMP | `42 4D` | 不属于 `detectImageFormatFromBuffer()` 的通用识别范围；仅在剪贴板/路径读取分支被上游识别后转为 PNG |

### 2.3 尺寸元数据追踪

> 源文件: `src/utils/imageResizer.ts` — `createImageMetadataText()`

每张图片处理后都会生成维度元数据，注入到对话中供模型理解坐标映射:

```typescript
type ImageDimensions = {
  originalWidth?: number    // 原始宽度
  originalHeight?: number   // 原始高度
  displayWidth?: number     // 压缩后宽度
  displayHeight?: number    // 压缩后高度
}
// 若被缩放，输出形如:
// "[Image: source: /path/to/img.png, original 4000x3000, displayed at 2000x1500.
//   Multiply coordinates by 2.00 to map to original image.]"
```

---

## 3. 图片输入的五条路径

### 3.1 路径 A: 剪贴板粘贴 (ctrl+v)

> 源文件: `src/hooks/usePasteHandler.ts`, `src/utils/imagePaste.ts`

快捷键为**平台相关**: Windows 使用 `alt+v`（避免与系统 `ctrl+v` 冲突），macOS 与 Linux 使用 `ctrl+v`。

```
用户 ctrl+v
  └→ usePasteHandler.wrappedOnInput()
      ├→ 文本为空? → checkClipboardForImage()
      │    ├→ [macOS 快速路径] 原生 NSPasteboard reader (image-processor-napi)
      │    │    └→ getNativeModule().readClipboardImage(2000, 2000)
      │    │        → 返回 { png: Buffer, originalWidth, originalHeight, width, height }
      │    │        → 若 buffer > 3.75MB → maybeResizeAndDownsampleImageBuffer()
      │    │
      │    └→ [回退路径] osascript / xclip / PowerShell
      │         → 保存到临时文件 → 读取 → sharp 处理 → 删除临时文件
      │
      └→ 文本非空? → 检查是否包含图片文件路径
           ├→ IMAGE_EXTENSION_REGEX: /\.(png|jpe?g|gif|webp)$/i
           └→ isImageFilePath() → tryReadImageFromPath()
               → 从绝对路径或剪贴板路径读取 → 处理
```

**跨平台剪贴板命令:**

| 平台 | 检查图片 | 保存图片 |
|------|---------|---------|
| macOS | `osascript -e 'the clipboard as «class PNGf»'` | AppleScript 写入临时文件 |
| Linux | `xclip -selection clipboard -t TARGETS` / `wl-paste -l` | xclip/wl-paste 输出到文件 |
| Windows | `powershell Get-Clipboard -Format Image` | PowerShell 保存为 PNG |

### 3.2 路径 B: 拖拽文件到终端

> 源文件: `src/hooks/usePasteHandler.ts`

终端模拟器将拖拽文件转为文本粘贴（路径字符串）。处理逻辑:

```
拖拽文件 → 终端粘贴路径文本
  → wrappedOnInput 检测到粘贴
  → 按行分割，每行检查 isImageFilePath()
  → 匹配的行 → tryReadImageFromPath(line)
     → 读取文件 → BMP 转 PNG → 压缩 → onImagePaste()
  → 非图片行 → 作为普通文本处理
```

### 3.3 路径 C: @ 附件引用

> 源文件: `src/utils/attachments.ts`, `src/tools/FileReadTool/FileReadTool.ts`

用户在输入框中使用 `@/path/to/image.png` 引用文件:

```
@ 附件解析
  → 识别图片扩展名 (IMAGE_EXTENSIONS: png, jpg, jpeg, gif, webp)
  → readImageWithTokenBudget(normalizedPath)
     → readFileBytes(filePath, maxBytes)    // 读取一次
     → detectImageFormatFromBuffer()        // 魔数检测格式
     → maybeResizeAndDownsampleImageBuffer()  // 标准压缩
     → 检查 token budget (base64.length * 0.125)
     → 若超预算 → compressImageBufferWithTokenLimit()
        → 计算: maxBase64Chars = maxTokens / 0.125
        →        maxBytes = maxBase64Chars * 0.75
        → 调用 compressImageBuffer() 多策略压缩
     → 最终回退: sharp resize(400, 400) + jpeg(quality=20)
```

### 3.4 路径 D: FileReadTool 工具调用

> 源文件: `src/tools/FileReadTool/FileReadTool.ts`

模型主动调用 FileReadTool 读取图片文件:

```
Claude: tool_use { name: "Read", file_path: "/path/to/screenshot.png" }
  → FileReadTool.validateInput()
     → 检查扩展名: IMAGE_EXTENSIONS.has(ext) → 跳过二进制拒绝
     → 结果: { result: true }
  → FileReadTool.call()
     → 检测扩展名 → 进入图片分支
     → readImageWithTokenBudget(resolvedFilePath, maxTokens)
     → 返回 ImageResult { type: 'image', file: { base64, type, originalSize, dimensions } }
  → mapToolResultToToolResultBlockParam()
     → 转为 ImageBlockParam { type: 'image', source: { type: 'base64', ... } }
  → 同时注入维度元数据 isMeta 消息
```

  #### 3.4.1 Read 工具调用链细节

  > 源文件: `src/tools/FileReadTool/FileReadTool.ts`, `src/Tool.ts`

  这一段不是“如何读图片”，而是 **Read 工具作为一个通用工具是如何被注册、校验、执行并回填到对话里的**。对图片/PDF 来说，真正进入模型上下文的内容，往往不只取决于 `callInner()` 读到了什么，还取决于 `mapToolResultToToolResultBlockParam()` 和 `newMessages` 如何补发。

  ```
  Claude 发起 tool_use(Read)
    → buildTool(...) 注册的 FileReadTool 接管
    → backfillObservableInput()
      → expandPath(file_path)
    → validateInput()
      → 校验 pages 参数格式
      → 检查 deny 规则
      → 拦截 UNC path 的提前 I/O
      → 通过扩展名阻止大多数二进制文件
      → 拦截会阻塞的 device file
    → checkPermissions()
      → checkReadPermissionForTool(...)
    → call()
      → 读取默认/覆写的 maxSizeBytes, maxTokens
      → readFileState 去重（仅文本/Notebook）
      → 触发基于路径的 skills 发现与激活
      → callInner() 按扩展名分流
        ├→ ipynb
        ├→ image
        ├→ pdf
        └→ text
    → mapToolResultToToolResultBlockParam()
      → 生成 tool_result
    → 若存在 newMessages
      → 额外注入 supplemental isMeta 消息
  ```

  ##### A. 工具注册层

  `FileReadTool` 通过 `buildTool({...})` 注册为只读、可并发、安全输出不落盘的工具:

  - `searchHint: 'read files, images, PDFs, notebooks'`
  - `isReadOnly() → true`
  - `isConcurrencySafe() → true`
  - `maxResultSizeChars: Infinity`

  这里的 `Infinity` 不是“无限输出”，而是**禁止走通用的结果落盘机制**。源码注释写得很明确: Read 工具如果把结果再持久化到文件，模型又可能继续用 Read 去读这个文件，形成 Read → 文件 → 再 Read 的循环。

  ##### B. validateInput 与 checkPermissions 的边界

  `Tool.ts` 的接口约束是: **只有 `validateInput()` 通过后，才会进入 `checkPermissions()`**。所以 Read 工具前半段做的是“无需 I/O 的快速拒绝”，后半段才是权限确认。

  `validateInput()` 做的事包括:

  1. `pages` 参数纯字符串校验，且单次最多 `PDF_MAX_PAGES_PER_READ = 20` 页。
  2. 先按 `expandPath()` 归一化路径，再做 deny rule 匹配。
  3. 对 UNC path 只放行到权限阶段，避免在用户授权前触发网络文件系统访问。
  4. 通过扩展名拒绝大多数二进制文件，但为 PDF 和图片扩展名保留豁免。
  5. 拦截 `/dev/zero`、`/dev/random`、`/dev/tty` 等会阻塞或无限输出的设备文件。

  `checkPermissions()` 本身相对简单，直接委托给 `checkReadPermissionForTool()`。这意味着:

  - **Read 工具路径**有完整的工具权限链路。
  - **粘贴/拖拽路径**不走这一条链路，它们属于输入层自己的处理流程。

  ##### C. call 阶段: 去重、限制与路径修复

  `call()` 不是直接读文件，而是先做三件很关键的事情:

  1. **读取限制**
    - 从 `context.fileReadingLimits` 获取 `maxSizeBytes` 与 `maxTokens`
    - 若调用方覆写了限制，会打 `tengu_file_read_limits_override` 遥测

  2. **readFileState 去重**
    - 只对“同一路径、同一 range、文件 mtime 未变化”的读取生效
    - 命中后返回 `file_unchanged`
    - 该机制**只适用于文本/Notebook**，图片与 PDF 不写入这类 dedup 状态

  3. **ENOENT 友好恢复**
    - 对 macOS 截图文件名里普通空格/窄空格差异做二次尝试
    - 失败后再给出 `Did you mean ...` 风格的相似路径建议

  ##### D. callInner 分流: 媒体路径只是四个分支之一

  `callInner()` 的分流顺序是:

  1. `ipynb` → 读 Notebook JSON，并映射为 notebook cells
  2. 图片扩展名 → `readImageWithTokenBudget()`
  3. PDF 扩展名 → 走 PDF 分支
  4. 其他 → 作为文本按 offset/limit 读取

  这解释了为什么 Read 工具在项目里不仅是“文件读取”，还是一个**多模态分发器**。

  ##### E. tool_result 与 supplemental message 的双层回填

  这是 Read 工具最容易被忽略的一点: `callInner()` 返回的不只是 `data`，还可能带 `newMessages`。最终进入模型上下文的内容分两层:

  1. **tool_result 主结果**
  2. **supplemental `isMeta` 消息**

  不同类型的行为不同:

  | 类型 | tool_result 内容 | supplemental `newMessages` |
  |------|------------------|----------------------------|
  | text | 带行号的文本 + 可选 `CYBER_RISK_MITIGATION_REMINDER` | 通常没有 |
  | image | `image` block | 若有尺寸信息，额外注入 `createImageMetadataText()` |
  | pdf | 只返回“PDF file read: ...”摘要字符串 | 实际 PDF base64 作为 `document` block 通过 `isMeta` 补发 |
  | parts | 只返回“PDF pages extracted: ...”摘要字符串 | 实际页面图片数组通过 `isMeta` 补发 |
  | file_unchanged | 固定 stub | 没有 |

  这就是为什么从 transcript 视角看，PDF 的 `tool_result` 看起来像只返回了一行摘要，但模型实际上已经收到了真正的 `document` block。

  ##### F. 文本分支的特殊安全包装

  文本分支和媒体分支还有一个明显差异:

  - 文本会经过 `formatFileLines()`，附带行号
  - 文本可能附加 `CYBER_RISK_MITIGATION_REMINDER`
  - 图片/PDF 分支**不会**附加这个提醒，而是直接发送结构化 media block

  这意味着 Read 工具虽然统一叫“Read”，但不同文件类型在最终 prompt 中的序列化形式差异很大。

### 3.5 路径 E: BashTool 命令输出图片

> 源文件: `src/tools/BashTool/utils.ts`

Shell 命令的 stdout 若以 `data:image/...;base64,` 开头，自动识别为图片:

```
Claude: tool_use { name: "Bash", command: "python plot.py" }
  → 命令输出 stdout: "data:image/png;base64,iVBOR..."
  → formatOutput(content)
     → isImageOutput(content) → true (正则: /^data:image\/[a-z0-9.+_-]+;base64,/i)
  → resizeShellImageOutput(stdout, outputFilePath, outputFileSize)
     → parseDataUri(source) → { mediaType, data }
     → Buffer.from(data, 'base64')
     → maybeResizeAndDownsampleImageBuffer(buf, buf.length, ext)
     → 重新编码为 data URI
  → buildImageToolResult(stdout, toolUseID)
     → 返回 ToolResultBlockParam { content: [{ type: 'image', ... }] }
```

**安全限制:** 从文件读取时最大 20MB (`MAX_IMAGE_FILE_SIZE`)。

---

## 4. PDF 文档处理

### 4.1 双路径处理策略

> 源文件: `src/utils/pdf.ts`, `src/utils/pdfUtils.ts`

```
┌──────────────┐
│ FileReadTool │
│ 读取 .pdf    │
└──────┬───────┘
    │
    ├── pages 参数存在?
    │    → extractPDFPages(filePath, range)
    │    → pdftoppm 拆为 JPEG 页面图片
    │    → 每页 maybeResizeAndDownsampleImageBuffer()
    │    → 作为 ImageBlockParam[] 注入消息
    │
    └── pages 参数不存在
      │
      ├── pageCount > 10?
      │    → 报错，要求改用 pages 参数分段读取
      │
      ├── 当前模型不支持 PDF document block?
      │    → 报错，提示切换模型或使用 pages 参数
      │
      └── 当前模型支持 PDF document block
        → readPDF() → base64 编码
        → 作为 supplemental DocumentBlockParam 注入消息
        → tool_result 本身只返回 PDF 元信息摘要
```

需要区分两件事:

1. `pages` 参数路径会真正把 PDF 页面渲染成图片并发送给模型。
2. 不带 `pages` 的整档读取路径，在满足 `shouldExtractPages` 时可能会真实执行一次 `extractPDFPages()`（创建输出目录并调用 `pdftoppm`）用于提取/遥测，但不会自动把这些提取结果替换成返回给模型的页面图片；支持 PDF 的模型最终仍走 `document` block，不支持的模型直接报错。

### 4.2 API 限制常量

> 源文件: `src/constants/apiLimits.ts`

| 常量 | 值 | 说明 |
|------|---|------|
| `API_IMAGE_MAX_BASE64_SIZE` | 5 MB | API 硬限（base64 字符串长度） |
| `IMAGE_TARGET_RAW_SIZE` | 3.75 MB | 客户端目标（raw bytes，编码后 ≈ 5MB） |
| `IMAGE_MAX_WIDTH` | 2000 px | 客户端最大宽度 |
| `IMAGE_MAX_HEIGHT` | 2000 px | 客户端最大高度 |
| `PDF_TARGET_RAW_SIZE` | 20 MB | PDF 最大原始大小 |
| `API_PDF_MAX_PAGES` | 100 | API 最大页数 |
| `PDF_EXTRACT_SIZE_THRESHOLD` | 3 MB | 触发 full-read 分支执行页面提取尝试与遥测的阈值，但整档读取不因此自动改走图片返回 |
| `PDF_MAX_EXTRACT_SIZE` | 100 MB | 页面提取的最大 PDF 大小 |
| `PDF_MAX_PAGES_PER_READ` | 20 | 单次 Read 工具最大页数 |
| `API_MAX_MEDIA_PER_REQUEST` | 100 | 单请求最大媒体项（图片+PDF） |

### 4.3 PDF 安全校验

```typescript
// 魔数校验: 防止非 PDF 文件（如 HTML 被重命名为 .pdf）进入对话
const header = fileBuffer.subarray(0, 5).toString('ascii')
if (!header.startsWith('%PDF-')) {
  return { success: false, error: { reason: 'corrupted', message: '...' } }
}
```

---

## 5. 视频与音频处理 — 不直接支持

### 5.1 二进制文件拒绝

> 源文件: `src/constants/files.ts`, `src/tools/FileReadTool/FileReadTool.ts`

视频和音频被归类为 `BINARY_EXTENSIONS`:

```typescript
// 视频: .mp4, .mov, .avi, .mkv, .webm, .wmv, .flv, .m4v, .mpeg, .mpg
// 音频: .mp3, .wav, .ogg, .flac, .aac, .m4a, .wma, .aiff, .opus
```

FileReadTool 的 `validateInput` 在检查扩展名时:

```typescript
if (
  hasBinaryExtension(fullFilePath) &&
  !isPDFExtension(ext) &&
  !IMAGE_EXTENSIONS.has(ext.slice(1))  // PDF 和图片豁免
) {
  return {
    result: false,
    message: `This tool cannot read binary files. The file appears to be a binary ${ext} file.`,
  }
}
```

### 5.2 间接处理方式

用户可以通过 BashTool 间接处理视频/音频:
- `ffmpeg` 提取视频帧为图片 → 通过 data URI 返回
- `ffprobe` 获取媒体元数据 → 文本输出
- 但这需要用户主动引导模型使用 Bash 命令

### 5.3 语音输入（独立通路）

> 源文件: `src/services/voice.ts`, `src/services/voiceStreamSTT.ts`

语音输入有专门的 STT (Speech-to-Text) 通路，使用 WebSocket 二进制帧传输音频到服务端转写，不经过上述图片管线。

---

## 6. 二进制文件检测与分流

> 源文件: `src/constants/files.ts`

### 6.1 两类检测能力

```
文件检测分流:
  ├── 1. 扩展名检查: hasBinaryExtension(filePath)
  │      → BINARY_EXTENSIONS 集合约 70+ 扩展名
  │
  └── 2. 内容分析: isBinaryContent(buffer)
         → 读取前 8192 字节
         → 发现 null byte (0x00) → 立即判定二进制
         → 统计非打印字符比例 > 10% → 判定二进制
```

但在当前图片/PDF 的 FileReadTool 主分流中，前置拒绝主要依赖**扩展名检查**。`isBinaryContent()` 是项目中的通用能力，并不直接参与这条媒体读取主链路。

### 6.2 豁免清单

以下二进制格式被豁免，可由 FileReadTool 直接处理:

| 格式 | 豁免原因 |
|------|---------|
| `.png`, `.jpg`, `.jpeg`, `.gif`, `.webp` | 图片 → 转为 ImageBlockParam |
| `.pdf` | PDF → 转为 DocumentBlockParam 或拆页面图片 |

---

## 7. 原生图片处理引擎 (image-processor-napi)

### 7.1 架构

> 源文件: `vendor/image-processor-src/index.ts`, `src/tools/FileReadTool/imageProcessor.ts`

```
┌────────────────────────────┐
│ getImageProcessor()        │
│                            │
│  bundled mode?             │
│   ├─ 是 → 优先尝试 image-  │
│   │      processor-napi    │
│   │      失败则回退 sharp  │
│   └─ 否 → 直接使用 sharp   │
└────────────────────────────┘

┌────────────────────────────┐
│ 剪贴板原生读取（macOS）     │
│ feature('NATIVE_CLIPBOARD_ │
│ IMAGE') + GrowthBook gate  │
│ → readClipboardImage()     │
│ → 否则回退 osascript 路径   │
└────────────────────────────┘
```

这里有两套不同条件:

1. **通用图片处理器选择**: `getImageProcessor()` 只在 bundled mode 下优先尝试原生模块，否则直接用 `sharp`。
2. **macOS 剪贴板快速路径**: 还额外受 `NATIVE_CLIPBOARD_IMAGE` 和 `tengu_collage_kaleidoscope` 控制。

### 7.2 原生模块 API

```typescript
type NativeModule = {
  processImage: (input: Buffer) => Promise<ImageProcessor>
  readClipboardImage?: (maxWidth: number, maxHeight: number) => ClipboardImageResult | null
  hasClipboardImage?: () => boolean
}

type ClipboardImageResult = {
  png: Buffer             // 已缩放的 PNG buffer
  originalWidth: number   // 原始宽度
  originalHeight: number  // 原始高度
  width: number           // 当前宽度（≤ maxWidth）
  height: number          // 当前高度（≤ maxHeight）
}
```

### 7.3 sharp 适配层

native 模块暴露了一个与 `sharp` API 兼容的适配层:

```typescript
export function sharp(input: Buffer): SharpInstance {
  // 链式操作 API，与 npm sharp 包兼容
  // .metadata() / .resize() / .jpeg() / .png() / .webp() / .toBuffer()
}
```

**重要注意:** 原生模块需要为每次操作创建新的 sharp 实例，因为 `toBuffer()` 后的实例不能正确应用格式转换（源码注释中明确记录了这个 bug）。

### 7.4 功能开关 (Feature Gate)

```
剪贴板原生路径条件:
feature('NATIVE_CLIPBOARD_IMAGE')
AND
getFeatureValue_CACHED_MAY_BE_STALE('tengu_collage_kaleidoscope', true)
→ 使用 macOS 原生 NSPasteboard 读取
→ 否则回退 osascript 路径
```

---

## 8. API 限制与压缩策略

### 8.1 多策略渐进式压缩

> 源文件: `src/utils/imageResizer.ts` — `compressImageBuffer()`

```
输入: imageBuffer + maxBytes (默认 3.75MB)
│
├── 已在限制内? → 直接返回
│
├── 策略 1: 渐进缩放 (tryProgressiveResizing)
│    按 1.0 → 0.75 → 0.5 → 0.25 比例尝试缩放
│    并尽量保留原始格式
│
├── 策略 2: PNG 调色板优化 (tryPalettePNG)
│    仅对 PNG: compressionLevel=9, palette=true, colors=64
│
├── 策略 3: JPEG 转换 (tryJPEGConversion)
│    转为 JPEG, quality=50
│
└── 策略 4: 终极压缩 (createUltraCompressedJPEG)
     缩小到 400px + JPEG quality=20
```

### 8.2 Token 预算限制

FileReadTool 读取图片时，还受 token 预算约束:

```
estimatedTokens = base64.length * 0.125  (1 token ≈ 8 base64 字符)

若 estimatedTokens > maxTokens:
  → compressImageBufferWithTokenLimit(imageBuffer, maxTokens)
  → maxBase64Chars = maxTokens / 0.125
  → maxBytes = maxBase64Chars * 0.75
  → 按该 maxBytes 限制运行压缩管线
```

### 8.3 API 边界校验 (safety net)

> 源文件: `src/utils/imageValidation.ts` — `validateImagesForAPI()`

在所有消息发送到 API 之前，做最终大小检查:

```typescript
export function validateImagesForAPI(messages: unknown[]): void {
  // 遍历所有 user 消息中的 image block
  // 检查 base64-encoded string length > 5MB → 抛出 ImageSizeError
  // 这是最后的安全网: 即使上游处理遗漏，也不会发送超大图片
}
```

---

## 9. 图片存储与生命周期管理

### 9.1 会话级图片缓存

> 源文件: `src/utils/imageStore.ts`

```
存储目录: ~/.claude/image-cache/{sessionId}/{imageId}.{ext}
  → 权限: 0o600 (仅当前用户可读写)
  → 内存缓存: Map<number, string> 最多 200 条（达到上限时删除最早插入项）
  → 清理时机: 由全局 cleanup 流程异步清理旧 session 的图片目录
```

### 9.2 用途

- **路径引用:** 将粘贴的图片持久化到磁盘，模型可以在后续对话中引用路径
- **元数据注入:** `[Image source: /path/to/stored/image.png]`
- **去重:** FileReadTool 的 readFileState 机制不缓存图片（仅文本），避免重复发送

### 9.3 历史消息处理

> 源文件: `src/history.ts`

加载历史消息时，图片内容被过滤掉:

```typescript
// Filter out images (they're stored separately in image-cache)
if (content.type === 'image') {
  // 跳过 — 图片不持久化到历史消息中
}
```

---

## 10. 关键设计模式总结

### 10.1 设计决策

| 决策 | 选择 | 原因 |
|------|------|------|
| 图片"识别" | 不做本地识别，转发给 Claude Vision | CLI 工具避免引入 ML 依赖 |
| 图片引擎 | bundled mode 下优先原生 NAPI，否则用 sharp | 原生模块可加速部分路径，但不是所有环境的默认路径 |
| 格式检测 | Magic bytes 而非扩展名 | 可靠性: 防止扩展名被篡改 |
| 压缩策略 | 渐进式多层压缩 | 优先质量: 先尝试保留原格式，最后才降级 |
| PDF 读取 | `pages` 参数走页面图片；整档读取优先走 document block | 把“显式页面读取”和“整档读取”分开，避免混淆返回路径 |
| 视频/音频 | 拒绝直读 | API 不支持，但允许通过 BashTool 间接处理 |
| 图片存储 | 会话级磁盘缓存 + 有上限的路径 Map | 降低重复引用成本，并由周期性 cleanup 清理旧会话目录 |
| Token 预算 | 图片也受 token 限制 | 防止大量图片耗尽上下文窗口 |

### 10.2 安全考量

1. **大小限制是分层的:** 图片 API 硬限是 5MB base64（客户端目标约 3.75MB raw）；PDF 整档目标是 20MB；PDF 页面提取上限是 100MB；BashTool 图片文件回读额外有 20MB 限制
2. **权限检查:** Read 工具读取图片走 FileReadTool 权限链路；粘贴/拖拽路径属于独立输入路径
3. **API 边界校验:** `validateImagesForAPI()` 作为最终安全网
4. **恶意内容提醒:** `CYBER_RISK_MITIGATION_REMINDER` 嵌入文件读取结果（图片除外）
5. **设备文件屏蔽:** `/dev/zero`, `/dev/random` 等被阻止
6. **PDF 魔数校验:** 防止伪装 PDF 的 HTML 文件破坏会话

### 10.3 性能优化

1. **Single-read 原则:** 图片文件只读取一次，后续的压缩/格式转换都在内存中操作
2. **并行处理:** 多张粘贴图片通过 `Promise.all()` 并行压缩
3. **Lazy 初始化:** sharp 模块延迟加载，避免阻塞启动
4. **原生剪贴板读取:** macOS 上 ~0.03ms (热) vs osascript ~1.5s
5. **新 sharp 实例:** 每次操作创建新实例，避免原生模块的状态泄漏 bug
