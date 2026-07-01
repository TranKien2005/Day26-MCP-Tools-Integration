# Phân biệt MCP và Function Calling

Đây là hai khái niệm hay bị nhầm lẫn nhưng thực ra ở **hai tầng khác nhau**, và **bổ sung cho nhau** chứ không thay thế.

## Cấu trúc repo

```
day26-mcp/
├── README.md                ← Bạn đang đọc file này
├── requirements.txt         ← pip install -r requirements.txt
│
├── 01-function-calling/     ← Bước 1: Function Calling thuần (Gemini SDK)
│   ├── README.md
│   └── weather_function_calling.py
│
├── 02-mcp-basics/           ← Bước 2: MCP server + client (không cần API key)
│   ├── README.md
│   ├── weather_server.py
│   └── weather_client.py
│
└── 03-production/           ← Bước 3: Auth, Tool Registry, Versioning
    ├── README.md
    ├── auth_server.py
    ├── auth_client.py
    ├── registry.json
    ├── registry_client.py
    └── versioned_server.py
```

## Quick start

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# MCP demo (không cần API key)
cd 02-mcp-basics && python weather_client.py

# Function Calling (cần Gemini API key)
export GEMINI_API_KEY=...
cd 01-function-calling && python weather_function_calling.py

# Production — Auth (2 terminal)
cd 03-production
python auth_server.py              # terminal 1
python auth_client.py              # terminal 2

# Production — Tool Registry
cd 03-production && python registry_client.py
```

---

## Định nghĩa ngắn gọn

**Function Calling** là một *khả năng của model* (capability). Model được huấn luyện để khi bạn đưa cho nó danh sách các "công cụ" (kèm schema mô tả tham số), nó có thể tự quyết định gọi công cụ nào và sinh ra JSON tham số phù hợp. Bản thân model **không chạy** function — nó chỉ nói "hãy gọi `get_weather(city='Hanoi')`". Code của bạn mới là người thực thi.

**MCP (Model Context Protocol)** là một *giao thức chuẩn* (protocol) — giống như USB-C hay HTTP cho thế giới AI. Nó định nghĩa cách một **MCP Client** (như Claude Code, Claude Desktop) kết nối tới các **MCP Server** để khám phá và sử dụng tools, resources, prompts một cách thống nhất.

---

## Ví dụ minh hoạ trực quan


### Ví dụ 1 — USB-C: hiểu MCP qua phép so sánh

```
TRƯỚC MCP (mỗi thiết bị 1 cổng riêng)      SAU MCP (1 chuẩn cho tất cả)
──────────────────────────────────           ─────────────────────────────

  App 1 ──[format A]──▶ Tool 1                App 1 ─┐
  App 1 ──[format B]──▶ Tool 2                App 2 ─┼── MCP ──┬── Tool 1
  App 2 ──[format C]──▶ Tool 1                App 3 ─┘         ├── Tool 2
  App 2 ──[format D]──▶ Tool 2                                 └── Tool 3
  App 3 ──[format E]──▶ Tool 1
  App 3 ──[format F]──▶ Tool 2         Viết tool 1 lần → mọi app dùng được

  6 kết nối cho 3 app × 2 tool         3 app + 3 tool = chỉ 6 kết nối MCP
  Thêm 1 tool = viết thêm 3 format    Thêm 1 tool = 0 thay đổi ở app
```

### Ví dụ 2 — Cùng một câu hỏi, hai cách xử lý

User hỏi: **"Thời tiết Hà Nội hôm nay thế nào?"**

```
╔══════════════════════════════════════════════════════════════════════╗
║  CÁCH 1: Function Calling thuần                                      ║
║                                                                      ║
║  weather_app.py (MỘT file làm hết)                                   ║
║  ┌──────────────────────────────────────────────────┐                ║
║  │ # Bước 1: Viết schema THỦ CÔNG (30 dòng)         │                ║
║  │ schema = {                                       │                ║
║  │   "name": "get_weather",                         │                ║
║  │   "parameters": {                                │                ║
║  │     "city": {"type": "string"} ...               │                ║
║  │   }                                              │                ║
║  │ }                                                │                ║
║  │                                                  │                ║
║  │ # Bước 2: Viết hàm thực thi                      │                ║
║  │ def get_weather(city): ...                       │                ║
║  │                                                  │                ║
║  │ # Bước 3: Gửi cho model                          │                ║
║  │ response = model.generate(prompt, tools=[schema])│                ║
║  │                                                  │                ║
║  │ # Bước 4: App tự chạy hàm                        │                ║
║  │ result = get_weather("Hà Nội")  ← APP chạy       │                ║
║  │                                                  │                ║
║  │ # Bước 5: Đưa kết quả lại cho model              │                ║
║  │ final = model.generate(prompt + result)          │                ║
║  └──────────────────────────────────────────────────┘                ║
║                                                                      ║
║  ⚠️ Muốn dùng tool này ở app khác?                                   ║
║     → Copy cả schema + hàm sang app mới                              ║
║     → Đổi sang OpenAI? Viết lại schema theo format OpenAI            ║
╚══════════════════════════════════════════════════════════════════════╝

╔══════════════════════════════════════════════════════════════════════╗
║  CÁCH 2: MCP                                                         ║
║                                                                      ║
║  weather_server.py (TÁCH RIÊNG)    Bất kỳ MCP client nào             ║
║  ┌─────────────────────────┐       ┌──────────────────────┐          ║
║  │ @mcp.tool()             │       │ Claude Code          │          ║
║  │ def get_weather(city):  │◀──────│ Cursor               │          ║
║  │     ...                 │  MCP  │ Gemini CLI           │          ║
║  │                         │       │ App tự viết          │          ║
║  │ # Schema? TỰ ĐỘNG sinh │       │ ...                  │           ║
║  │ # từ type hints!        │       │                      │          ║
║  └─────────────────────────┘       └──────────────────────┘          ║
║         SERVER chạy                   CLIENT chỉ điều phối           ║
║                                                                      ║
║  ✅ Viết 1 lần server → Claude, Cursor, app tự viết đều dùng được     ║
║     Thêm tool mới? Thêm 1 hàm @mcp.tool(), không sửa client          ║
╚══════════════════════════════════════════════════════════════════════╝
```

### Ví dụ 4 — Dòng thời gian một request

```
Thời gian ──────────────────────────────────────────────────────▶

FUNCTION CALLING:
  User        App                    Model
   │──"HN?"──▶│                        │
   │           │──prompt + schema────▶ │
   │           │◀──"gọi get_weather"── │    Model CHỈ ra lệnh
   │           │                        │
   │           │ get_weather("HN") ◀── │    App TỰ CHẠY hàm
   │           │──────kết quả────────▶ │
   │           │◀──"HN 29°C, mưa"──── │    Model tổng hợp
   │◀──────────│                        │
   │  "Hà Nội 29°C, mưa nhẹ, nhớ mang ô nhé!"

MCP:
  User     Client           MCP Protocol          Server
   │──"HN?"──▶│                  │                   │
   │           │──list_tools───▶ │ ────────────────▶ │  Khám phá
   │           │◀──tool list──── │ ◀──────────────── │
   │           │                  │                   │
   │           │  (Model quyết định gọi get_weather)  │  Function Calling
   │           │                  │                   │    bên trong
   │           │──call_tool────▶ │ ────────────────▶ │
   │           │                  │       SERVER chạy hàm  ← Server chạy
   │           │◀──result──────── │ ◀──────────────── │
   │           │                  │                   │
   │           │  (Model tổng hợp)                    │
   │◀──────────│                  │                   │
   │  "Hà Nội 29°C, mưa nhẹ, nhớ mang ô nhé!"
```

---

## So sánh trực tiếp

| Tiêu chí | Function Calling | Model Context Protocol (MCP) |
|---|---|---|
| **Bản chất** | Tính năng của mô hình (Model capability) | Giao thức giao tiếp client–server |
| **Ai định nghĩa tool?** | Bạn hard-code trong từng app | Server tự công bố (self-describe) tool |
| **Tái sử dụng** | Phải viết lại cho mỗi app/model | Viết 1 lần, mọi MCP client dùng được |
| **Thực thi** | App của bạn tự chạy | MCP Server chạy, client điều phối |
| **Tính chuẩn hóa** | Mỗi nhà cung cấp 1 kiểu (OpenAI, Anthropic khác nhau) | Một chuẩn chung do Anthropic đề xuất |
| **Hệ sinh thái** | Khó chia sẻ dạng module đóng gói sẵn | Dễ dàng chia sẻ và tải về các "MCP Servers" mã nguồn mở |

## Quan hệ giữa chúng

Điểm quan trọng nhất: **MCP dùng Function Calling bên dưới**. Chúng không loại trừ nhau.

```
User hỏi
   │
   ▼
LLM (dùng Function Calling để quyết định gọi tool nào)
   │
   ▼
MCP Client  ──[giao thức MCP]──►  MCP Server (thực thi tool thật)
   │                                   │
   ◄───────────── kết quả ─────────────┘
   ▼
LLM tổng hợp câu trả lời
```

## Khi nào dùng cái nào?

- **Function Calling thuần**: app đơn giản, tool gắn chặt với 1 ứng dụng, không cần chia sẻ.
- **MCP**: muốn tool/tích hợp dùng lại được trên nhiều AI client, muốn tách biệt logic tool khỏi app, hoặc xây hệ sinh thái tích hợp (DB, file, API nội bộ...).

---

## Minh hoạ bằng mã nguồn

Cùng một tool `get_weather`, dưới đây là hai cách triển khai để thấy rõ sự khác biệt.

### [Cách 1 — Function Calling thuần (Google Gemini SDK)](01-function-calling/)

Tool được **định nghĩa và thực thi ngay trong app**. Model chỉ quyết định gọi tool nào, app tự chạy và đưa kết quả trở lại.

```
User hỏi → Model quyết định gọi get_weather("Hà Nội")
                    │
                    ▼
             App TỰ THỰC THI hàm get_weather
                    │
                    ▼
             Model tổng hợp câu trả lời
```

> Nhược điểm: schema viết tay, tool gắn chặt trong app — muốn dùng lại ở app khác phải copy cả schema lẫn hàm.

Chi tiết + code: xem [`01-function-calling/README.md`](01-function-calling/README.md)

### [Cách 2 — MCP (server tự công bố tool, mọi client dùng chung)](02-mcp-basics/)

Tool được tách ra **một MCP server độc lập**. Server tự "khai báo" nó có tool gì; bất kỳ MCP client nào (Claude Code, Claude Desktop, Cursor...) cũng cắm vào dùng được mà không cần biết code bên trong.

```
weather_client.py                       weather_server.py
┌─────────────┐    giao thức MCP    ┌─────────────────┐
│  list_tools │ ──────────────────▶ │ @mcp.tool()     │
│  call_tool  │ ◀────────────────── │ get_weather()   │
└─────────────┘     stdio           └─────────────────┘
```

Chi tiết + code: xem [`02-mcp-basics/README.md`](02-mcp-basics/README.md)

### Điểm khác biệt rút ra từ code

| | Function Calling thuần | MCP |
|---|---|---|
| Khai báo schema | Tự viết tay trong app | `@mcp.tool()` tự sinh từ type hints |
| Nơi thực thi tool | Trong app gọi model | Trong MCP server riêng |
| Khám phá tool | Hard-code danh sách `tools` | `session.list_tools()` tại runtime |
| Dùng lại ở app khác | Copy code | Cắm thêm client, không sửa server |
| Vai trò Function Calling | Là toàn bộ cơ chế | Là lớp model bên trong MCP |

---

## [MCP trong Production](03-production/)

Các ví dụ trên chạy tốt trên máy cá nhân, nhưng đưa vào **hệ thống production** cần giải quyết thêm ba vấn đề:

```
┌─────────────────────────────────────────────────────┐
│                  Production MCP                     │
│                                                     │
│  ┌──────────┐   ┌───────────┐   ┌───────────────┐   │
│  │ Security │   │ Registry  │   │  Versioning   │   │
│  │          │   │           │   │               │   │
│  │ • Auth   │   │ • Discover│   │ • v1 compat   │   │
│  │ • Token  │   │ • Connect │   │ • v2 features │   │
│  │ • Scopes │   │ • Health  │   │ • Deprecation │   │
│  └──────────┘   └───────────┘   └───────────────┘   │
└─────────────────────────────────────────────────────┘
```

### 1. Security — Authentication & Authorization

MCP server phục vụ qua **HTTP** cho nhiều client → cần xác thực. MCP SDK hỗ trợ sẵn **Bearer Token** verification:

- Server: cấu hình `AuthSettings` + implement `TokenVerifier` protocol
- Client: gửi header `Authorization: Bearer <token>` qua `httpx.AsyncClient`
- Không có token → 401, token sai → 403, logic tool không biết gì về auth

| Tầng | Demo (stdio) | Production (HTTP) |
|---|---|---|
| Transport | stdio (cùng máy) | Streamable HTTP (qua mạng) |
| Auth | Không cần | Bearer token / OAuth / mTLS |
| Phạm vi truy cập | Toàn bộ | Scopes giới hạn từng client |

### 2. Tool Registry & Discovery

Agent **không hard-code** tool nào. Nó hỏi **Tool Registry** — danh mục trung tâm liệt kê tất cả tool từ mọi server — theo yêu cầu task:

```
Agent nhận task "lấy thời tiết Hà Nội"
   │
   ▼
Tool Registry: "tool nào có tag 'weather'?"
   │
   ├── get_weather v1.0 → server: weather (stdio)
   └── get_weather_v2 v2.0 → server: weather-v2 (stdio)
   │
   ▼
Agent chọn best match (v2.0, không deprecated)
   │
   ▼
Kết nối tới server weather-v2, gọi get_weather_v2(city="Hanoi")
```

Registry là **tool-centric** — đơn vị khám phá là **tool** (tag, description, parameters), không phải server.

| | Hard-code (demo) | Tool Registry (production) |
|---|---|---|
| Agent biết tool nào? | Chỉ tool được code sẵn | Tất cả tool trong registry |
| Tìm tool | Theo tên cố định | Theo tag, keyword, capability |
| Thêm tool mới | Sửa code agent | Thêm entry vào registry |
| Chọn tool | Developer quyết định | Agent tự chọn best match |

### 3. Versioning & Backward Compatibility

Server v1 có `get_weather(city)` trả chuỗi đơn giản. V2 muốn trả JSON chi tiết, thêm `include_forecast`. Nếu đổi trực tiếp → mọi client cũ break. Giải pháp — 3 kỹ thuật kết hợp:

| Kỹ thuật | Mô tả |
|---|---|
| **Tool mới song song** | `get_weather_v2` tồn tại bên cạnh `get_weather` — không xoá tool cũ |
| **Tham số optional** | `include_forecast`, `units` có default → client cũ gọi vẫn đúng |
| **Server metadata** | Resource `server://info` công bố version + deprecation notice |

Chi tiết + code cho cả 3 phần: xem [`03-production/README.md`](03-production/README.md)

### Tổng kết Production Checklist

| Khía cạnh | Dev/Demo | Production |
|---|---|---|
| **Transport** | stdio (cùng máy) | HTTP/SSE (qua mạng) |
| **Auth** | Không | Bearer token, OAuth, mTLS |
| **Discovery** | Hard-code tool/server | Tool Registry — agent tìm tool theo task |
| **Versioning** | 1 tool duy nhất | Tool v1 + v2 song song, deprecation notice |
| **Health** | Không | Health check, retry, circuit breaker |
| **Logging** | `print()` | Structured logging, tracing (OpenTelemetry) |

---

**Tóm lại:** Function Calling là *cơ chế model gọi công cụ*; MCP là *chuẩn để kết nối model với các công cụ đó* — và MCP thực chất dùng Function Calling làm nền tảng để hoạt động.
