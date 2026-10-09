# ZZB 在线模型仓库 / Online Model Store

ZZB（AI 瞄准辅助）的在线模型与配置下载源。程序内「在线模型」「在线配置」两个标签页直接读取本仓库的
`models/` 与 `configs/` 目录。

> 本仓库只存放模型权重（`.onnx`）与配置文件（`.cfg`），不含任何可执行文件。

---

## 一、目录结构

```
models/            模型权重（.onnx）
configs/           配置预设（.cfg）
tools/             维护脚本
  fetch-models.ps1       从 Aimmy 官方仓库批量拉取模型
  fetch-sources.ps1      从多个社区仓库批量拉取模型
  fetch-hf.ps1           从 Hugging Face 拉取模型
  build-store.ps1        SHA256 去重合并进 models/
  build-pt-repo.ps1      组装 .pt/.engine 辅助权重仓库
  prune-models.ps1       剔除冷门/重复/垃圾条目
  gen-manifest.ps1       生成 manifest.json（含 SHA256）
  verify-store.ps1       校验本地文件与 manifest 是否一致
  check-model.ps1        解析 ONNX 输入/输出签名，判断兼容性
  push-batched-git.ps1   分批推送（绕开单次 push 体积上限）
manifest.json      文件清单（名称、大小、SHA256）
```

> 文件名含空格、`[]`、`&` 等字符，脚本一律用 `-LiteralPath` 与显式 URI 处理；
> PowerShell 默认会把 `[]` 当通配符，直接传路径会静默失败。

## 二、程序如何读取

程序按以下顺序获取列表，逐级回落：

| 顺序 | 用途 | 地址 | 限流 |
|---|---|---|---|
| 1（首选） | 列出全部文件 | `https://raw.githubusercontent.com/<owner>/<repo>/main/manifest.json` | 无 |
| 2（回落） | 列出模型 | `https://api.github.com/repos/<owner>/<repo>/contents/models` | 60 次/小时（匿名，按 IP） |
| 2（回落） | 列出配置 | `https://api.github.com/repos/<owner>/<repo>/contents/configs` | 同上 |
| 3（缓存） | 兜底 | 本地 `bin/cache/store-*.json`（30 分钟 TTL） | — |

单个文件的下载走两个源，任一成功即停止：

| 顺序 | 地址 |
|---|---|
| 1（首选） | `https://raw.githubusercontent.com/<owner>/<repo>/main/<folder>/<文件名>` |
| 2（回落） | `https://cdn.jsdelivr.net/gh/<owner>/<repo>@main/<folder>/<文件名>` |

> **为什么首选 manifest.json**：GitHub Contents API 对未认证请求限制 **60 次/小时**
> （按 IP），多人共用网络时极易触发 403，商店页面会整片空白。`manifest.json`
> 由 raw 直出，不消耗 API 配额，因此列表以它为准。
>
> **为什么下载仍要回落**：jsDelivr 对含 `[` `]` 的文件名在冷缓存下会返回
> `HTTP 200` 但**响应体为空**。程序已加入最小体积校验（<100 KB 视为失败），
> 会自动切换到另一个源，避免写入 0 字节的损坏模型。

对应实现见 ZZB 源码 `ZZB/Other/AppIdentity.cs`（地址常量）、
`ZZB/Other/GithubManager.cs`（manifest 解析 + API + 缓存）、
`ZZB/Other/FileManager.cs`（目录列举）、`ZZB/UILibrary/ADownloadGateway.xaml.cs`（下载与体积校验）。

### 运行时覆盖

不改代码即可换源：在程序目录放 `bin/store.cfg`。

```ini
owner=你的用户名
repo=你的仓库
branch=main
```

`owner` 或 `repo` 留空即关闭在线商店，仅使用本地 `bin/models`。

## 三、新增模型

1. 把 `.onnx` 放进 `models/`，`.cfg` 放进 `configs/`。
2. 文件名建议保留 `游戏 [版本] (数据量) by 作者.onnx` 格式，便于搜索过滤。
   避免纯数字命名（如 `320 (7).onnx`）—— 无法辨识，会被 `prune-models.ps1` 剔除。
3. 约束：
   - 单个模型 5 MB – 50 MB（GitHub 单文件硬上限 100 MB）
   - 单个配置 ≤ 1 MB
   - 仓库总体积建议 < 5 GB
4. 重新生成清单并校验：

```powershell
powershell -File tools/gen-manifest.ps1
powershell -File tools/verify-store.ps1
```

5. 提交推送：

```powershell
powershell -File tools/push-batched-git.ps1 -Owner <owner> -Repo <repo>
```

## 四、模型兼容性

ZZB 使用 ONNX Runtime 加载模型，要求：

- 输入：`float32`，形状 `[B, 3, H, W]`（H/W 可为动态维度 `-1`）
- 输出：`float32`，形状 `[B, >=5, N]`
- 元数据 `names` 字段（类名映射）可选，存在时 ZZB 会读取并用于「瞄准部位」勾选

不符合上述形状的模型（例如纯分类网络、非 YOLO 结构）不会被 ZZB 加载。

用 `tools/check-model.ps1` 可以离线检查任一文件的输入/输出签名：

```powershell
powershell -File tools/check-model.ps1 -Path models/AIOv10.onnx
```

## 五、许可

模型权重由社区作者训练并投稿，版权归各作者所有。若你是某个模型的作者并希望下架，
请提交 Issue 或 PR 删除对应文件。
