# 基于网络嗅探的流量分析与安全检测系统

本项目实现了一个“抓包 -> 规则分析/可选AI -> 可视化展示”的本地安全检测系统：

- **C++ Producer**：基于 Npcap 抓包并输出标准 `.pcap` 文件，支持自动轮转。
- **Python Consumer**：监听并解析 `.pcap`，执行多种规则检测，生成报告与告警。
- **Web UI**：统一控制抓包/分析进程，展示报告列表、评分趋势、攻击分布并支持筛选。

---

## 目录结构

```text
security_test/
├─ cpp_producer/
│  ├─ CMakeLists.txt
│  └─ main.cpp
├─ python_consumer/
│  ├─ consumer.py
│  └─ requirements.txt
├─ web_ui/
│  ├─ app.py
│  ├─ templates/index.html
│  └─ static/{style.css, app.js}
├─ captures/      # 抓包文件目录（自动创建）
└─ reports/       # 分析报告目录（自动创建）
```

---

## 核心能力（当前版本）

### 抓包能力（C++）

- 基于 Npcap 抓取指定网卡流量
- 支持 BPF 过滤（例如 `tcp or udp`）
- `.pcap` 自动轮转：
  - 文件大小达到 **50MB**
  - 或写入时长达到 **60秒**

### 规则分析能力（Python）

- 协议与特征提取：TCP/UDP、源/目的 IP、端口、大小、TCP Flags、payload 样本
- 规则检测（示例）：
  - `syn_flood_suspected`
  - `port_scan_suspected`（TCP/UDP）
  - `dns_tunnel_suspected`
  - `data_exfil_suspected`
  - `c2_channel_suspected`（已做更严格阈值控制）
  - Web 攻击载荷：
    - `xss_suspected`
    - `sqli_suspected`（含 `1=1`、`'1'='1'`、不完整引号变体等）
    - `command_injection_suspected`
    - `path_traversal_suspected`
    - `ssrf_suspected`
    - `malicious_upload_suspected`
- 输出报告：`reports/*.analysis.json`
- 告警日志：`reports/alerts.jsonl`（存在告警时追加）

### AI 研判能力（Qwen）

- 支持自动 AI（分析进程启动时开启）
- 支持手动 AI（Web 报告列表“AI分析”按钮）
- 模型返回异常时自动降级为文本分析，不中断流程
- 威胁评分趋势图标注为：**AI分析专用**

### Web UI 能力

- 抓包/分析进程启动、停止、状态监控
- 报告列表与详情查看
- 可视化图表：
  - 威胁评分趋势（AI分析专用）
  - 攻击类型分布饼图
  - 攻击源 IP 分布饼图
- 筛选器（联动表格和图表）：
  - 时间范围：最近5分钟 / 1小时 / 24小时 / 全部 / 自定义（近N分钟）
  - 攻击类型、AI状态、评分区间、文件名关键词
  - 告警源IP / 告警目的IP（支持模糊匹配与多值）
  - 本机IP排除（仅用于“攻击源IP分布”饼图）

---

## 环境要求

- Windows 10/11
- Python 3.10+
- Npcap（勾选 WinPcap API Compatible Mode）
- CMake + 支持 C++17 的编译器（MSVC）

---

## 1) 构建并运行 C++ 抓包端

在 `cpp_producer` 目录执行：

```powershell
cmake -S . -B build -DNPCAP_ROOT="C:/Program Files/Npcap"
cmake --build build --config Release
```

运行示例：

```powershell
.\build\Release\producer.exe "<设备名>" "..\captures" "tcp or udp"
```

说明：

- `<设备名>` 为空时会打印可用网卡列表
- 输出文件示例：`capture_20260428_102458_001.pcap`

---

## 2) 安装并运行 Python 分析端

安装依赖：

```powershell
pip install -r python_consumer\requirements.txt
```

可选设置环境变量（自动 AI 需要）：

```powershell
setx QWEN_API_KEY "你的API_KEY"
setx QWEN_BASE_URL "https://dashscope.aliyuncs.com/compatible-mode/v1"
```

运行示例：

```powershell
python python_consumer\consumer.py --capture-dir ".\captures" --report-dir ".\reports" --qwen-model "qwen3.6-plus"
```

常用参数（节选）：

- `--enable-ai`：开启自动 AI
- `--tcp-port-scan-min-unique-ports` / `--udp-port-scan-min-unique-ports`
- `--c2-channel-min-packets`
- `--c2-channel-min-forward-bytes`
- `--c2-channel-max-rev-to-fwd-ratio`

---

## 3) 运行 Web 控制台

启动：

```powershell
python web_ui\app.py
```

浏览器访问：

`http://127.0.0.1:7860`

---

## 使用建议

- 抓包前先确认 `device_name` 正确（如 `\\Device\\NPF_{...}`）
- 若 `producer.exe` 路径不同，请在 UI 中修改
- 改动规则后需重启分析进程，旧报告不会自动回溯重算
- 若你只想看规则分析，可不开启自动 AI

---

## 报告文件结构（简要）

`*.analysis.json` 主要包含：

- `input_summary`：流量统计、规则命中、样本包
- `llm_assessment`：AI 结果（或降级说明）
- `alerts`：规则告警聚合结果（含 `severity/confidence/evidence`）
- `ai`：AI执行元信息（是否执行、触发方式、模型）
