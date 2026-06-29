# Phân biệt MCP và Function Calling

Đây là hai khái niệm hay bị nhầm lẫn nhưng thực ra ở **hai tầng khác nhau**, và **bổ sung cho nhau** chứ không thay thế.

## Cấu trúc repo

```
day26-mcp/
├── README.md                ← Bạn đang đọc file này
├── requirements.txt         ← pip install -r requirements.txt
│
├── 01-function-calling/     ← Bước 1: Function Calling thuần (Gemini SDK)
│   └── weather_function_calling.py
│
├── 02-mcp-basics/           ← Bước 2: MCP server + client (không cần API key)
│   ├── weather_server.py
│   └── weather_client.py
│
└── 03-production/           ← Bước 3: Auth, Tool Registry, Versioning
    ├── auth_server.py
    ├── auth_client.py
    ├── registry.json
    ├── registry_client.py
    └── versioned_server.py
```

## Cách chạy nhanh

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

## Định nghĩa ngắn gọn

**Function Calling** là một *khả năng của model* (capability). Model được huấn luyện để khi bạn đưa cho nó danh sách các "công cụ" (kèm schema mô tả tham số), nó có thể tự quyết định gọi công cụ nào và sinh ra JSON tham số phù hợp. Bản thân model **không chạy** function — nó chỉ nói "hãy gọi `get_weather(city='Hanoi')`". Code của bạn mới là người thực thi.

**MCP (Model Context Protocol)** là một *giao thức chuẩn* (protocol) — giống như USB-C hay HTTP cho thế giới AI. Nó định nghĩa cách một **MCP Client** (như Claude Code, Claude Desktop) kết nối tới các **MCP Server** để khám phá và sử dụng tools, resources, prompts một cách thống nhất.

## So sánh trực tiếp

| Tiêu chí | Function Calling | MCP |
|---|---|---|
| **Bản chất** | Khả năng của LLM | Giao thức giao tiếp client–server |
| **Tầng** | Tầng model | Tầng hạ tầng/tích hợp |
| **Ai định nghĩa tool?** | Bạn hard-code trong từng app | Server tự công bố (self-describe) tool |
| **Tái sử dụng** | Phải viết lại cho mỗi app/model | Viết 1 lần, mọi MCP client dùng được |
| **Thực thi** | App của bạn tự chạy | MCP Server chạy, client điều phối |
| **Tính chuẩn hóa** | Mỗi nhà cung cấp 1 kiểu (OpenAI, Anthropic khác nhau) | Một chuẩn chung do Anthropic đề xuất |
| **Phạm vi** | Chỉ "gọi hàm" | Tools + Resources + Prompts |

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

## Ví dụ minh họa

**Chỉ Function Calling (cách cũ):** Bạn viết app, định nghĩa tool `get_weather` ngay trong code, tự gọi API thời tiết, tự xử lý. Nếu muốn dùng tool này ở app khác → copy/viết lại.

**Với MCP:** Bạn viết một **weather MCP server** một lần. Sau đó Claude Desktop, Claude Code, Cursor... đều cắm vào dùng được mà không cần sửa code. Server tự "khai báo" nó có tool gì.

## Khi nào dùng cái nào?

- **Function Calling thuần**: app đơn giản, tool gắn chặt với 1 ứng dụng, không cần chia sẻ.
- **MCP**: muốn tool/tích hợp dùng lại được trên nhiều AI client, muốn tách biệt logic tool khỏi app, hoặc xây hệ sinh thái tích hợp (DB, file, API nội bộ...).

## Minh hoạ bằng mã nguồn

Cùng một tool `get_weather`, dưới đây là hai cách triển khai để thấy rõ sự khác biệt.

### Cách 1 — Function Calling thuần (Google Gemini SDK) → `01-function-calling/`

Tool được **định nghĩa và thực thi ngay trong app**. Model chỉ quyết định gọi tool nào, app tự chạy và đưa kết quả trở lại.

```python
# pip install google-genai
from google import genai
from google.genai import types

client = genai.Client()

MODEL = "gemini-2.5-flash"

# 1. App tự định nghĩa schema của tool
get_weather_declaration = types.FunctionDeclaration(
    name="get_weather",
    description="Lấy thời tiết hiện tại của một thành phố",
    parameters=types.Schema(
        type=types.Type.OBJECT,
        properties={
            "city": types.Schema(
                type=types.Type.STRING, description="Tên thành phố"
            )
        },
        required=["city"],
    ),
)

TOOLS = [types.Tool(function_declarations=[get_weather_declaration])]


# 2. App tự thực thi tool (trong thực tế sẽ gọi API thời tiết thật)
def get_weather(city: str) -> str:
    """Trả về thời tiết (mock) của *city*. Dùng làm tool cho model."""
    mock_data = {
        "Hà Nội": {
            "nhiệt_độ": "29°C",
            "thời_tiết": "trời mưa nhẹ",
            "độ_ẩm": "82%",
            "gió": {"hướng": "Đông Nam", "tốc_độ": "12 km/h"},
        },
        "Hồ Chí Minh": {
            "nhiệt_độ": "33°C",
            "thời_tiết": "mưa rào",
            "độ_ẩm": "75%",
            "gió": {"hướng": "Tây Nam", "tốc_độ": "15 km/h"},
        },
        "Đà Nẵng": {
            "nhiệt_độ": "30°C",
            "thời_tiết": "nhiều mây",
            "độ_ẩm": "78%",
            "gió": {"hướng": "Đông", "tốc_độ": "10 km/h"},
        },
    }
    import json

    default = {"nhiệt_độ": "28°C", "thời_tiết": "không có dữ liệu chi tiết"}
    return json.dumps({"city": city, **mock_data.get(city, default)}, ensure_ascii=False)


def run(prompt: str) -> str:
    """Gửi *prompt* tới Gemini, tự động xử lý function calling và trả về câu trả lời cuối."""
    contents: list[types.Content] = [
        types.Content(role="user", parts=[types.Part.from_text(text=prompt)])
    ]

    # 3. Gọi model — model quyết định có gọi tool hay không
    resp = client.models.generate_content(
        model=MODEL,
        contents=contents,
        config=types.GenerateContentConfig(tools=TOOLS),
    )

    # 4. Vòng lặp: nếu model yêu cầu tool, app TỰ THỰC THI rồi đưa kết quả trả lại
    while resp.function_calls:
        contents.append(resp.candidates[0].content)

        function_responses = []
        for fc in resp.function_calls:
            print(f"  [model yêu cầu] {fc.name}({fc.args})")
            result = get_weather(**fc.args)  # <-- app chạy, không phải model
            print(f"  [app thực thi]  -> {result}")
            function_responses.append(
                types.Part.from_function_response(
                    name=fc.name, response={"result": result}
                )
            )

        contents.append(types.Content(role="user", parts=function_responses))
        resp = client.models.generate_content(
            model=MODEL,
            contents=contents,
            config=types.GenerateContentConfig(tools=TOOLS),
        )

    # 5. Model tổng hợp câu trả lời cuối
    return resp.text


if __name__ == "__main__":
    question = "Thời tiết Hà Nội và Đà Nẵng hôm nay thế nào?"
    print(f"User: {question}\n")
    print("Trả lời:", run(question))
```

> Nhược điểm: nếu muốn dùng `get_weather` ở một app khác, bạn phải copy lại cả schema lẫn hàm thực thi.

### Cách 2 — MCP (server tự công bố tool, mọi client dùng chung) → `02-mcp-basics/`

Tool được tách ra **một MCP server độc lập**. Server tự "khai báo" nó có tool gì; bất kỳ MCP client nào (Claude Code, Claude Desktop, Cursor...) cũng cắm vào dùng được mà không cần biết code bên trong.

**Server** — `02-mcp-basics/weather_server.py`:

```python
# pip install "mcp[cli]"
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("weather")

_MOCK_DB = {
    "Hanoi": "29°C, trời mưa",
    "Haiphong": "33°C, mưa rào",
    "Danang": "30°C, nhiều mây",
}


@mcp.tool()
def get_weather(city: str) -> str:
    """Lấy thời tiết hiện tại của một thành phố."""
    return f"{city}: {_MOCK_DB.get(city, '28°C, không có dữ liệu chi tiết')}"


if __name__ == "__main__":
    mcp.run()  # mặc định chạy qua stdio
```

**Client** — kết nối tới server và để model gọi tool qua giao thức MCP:

```python
import asyncio
import sys

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client


async def main() -> None:
    params = StdioServerParameters(command=sys.executable, args=["weather_server.py"])

    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()

            # 1. KHÁM PHÁ tool mà server công bố (không hard-code)
            tools = await session.list_tools()
            print("Tools server cung cấp:")
            for t in tools.tools:
                print(f"  - {t.name}: {t.description}")

            # 2. Gọi tool — SERVER thực thi rồi trả kết quả về qua MCP
            for city in ["Hanoi", "Danang", "Haiphong"]:
                result = await session.call_tool("get_weather", {"city": city})
                print(f"\ncall_tool get_weather(city={city!r}):")
                print("  ->", result.content[0].text)


if __name__ == "__main__":
    asyncio.run(main())
```

**Đăng ký server với Claude Code** (làm 1 lần, dùng mãi):

```bash
claude mcp add weather -- python /đường/dẫn/weather_server.py
```

Sau bước này, Claude tự `list_tools` để biết server có `get_weather`, rồi dùng **chính cơ chế Function Calling** để quyết định khi nào gọi — bạn không phải viết thêm dòng code tích hợp nào.

### Điểm khác biệt rút ra từ code

| | Function Calling thuần | MCP |
|---|---|---|
| Khai báo schema | Tự viết tay trong app | `@mcp.tool()` tự sinh từ type hints |
| Nơi thực thi tool | Trong app gọi model | Trong MCP server riêng |
| Khám phá tool | Hard-code danh sách `tools` | `session.list_tools()` tại runtime |
| Dùng lại ở app khác | Copy code | Cắm thêm client, không sửa server |
| Vai trò Function Calling | Là toàn bộ cơ chế | Là lớp model bên trong MCP |

---

## MCP trong Production → `03-production/`

Các ví dụ trên chạy tốt trên máy cá nhân, nhưng đưa vào **hệ thống production** cần giải quyết thêm ba vấn đề: **Security**, **Registry & Discovery**, và **Versioning**.

```
┌─────────────────────────────────────────────────────┐
│                  Production MCP                     │
│                                                     │
│  ┌──────────┐   ┌───────────┐   ┌───────────────┐  │
│  │ Security │   │ Registry  │   │  Versioning   │  │
│  │          │   │           │   │               │  │
│  │ • Auth   │   │ • Discover│   │ • v1 compat   │  │
│  │ • Token  │   │ • Connect │   │ • v2 features │  │
│  │ • Scopes │   │ • Health  │   │ • Deprecation │  │
│  └──────────┘   └───────────┘   └───────────────┘  │
└─────────────────────────────────────────────────────┘
```

### 1. Security — Authentication & Authorization

**Vấn đề:** Ví dụ demo chạy qua **stdio** trên cùng máy — không cần auth. Nhưng khi MCP server phục vụ qua **HTTP** cho nhiều client (nội bộ hoặc bên ngoài), ai cũng gọi được tool nếu không có xác thực.

**Giải pháp:** MCP SDK hỗ trợ sẵn cơ chế **Bearer Token** verification. Bạn chỉ cần:
1. Cấu hình `AuthSettings` trên server
2. Implement `TokenVerifier` — protocol kiểm tra token (JWT, OAuth introspection, static list, ...)
3. Client gửi header `Authorization: Bearer <token>` khi kết nối

| Tầng | Demo (stdio) | Production (HTTP) |
|---|---|---|
| Transport | stdio (cùng máy) | Streamable HTTP (qua mạng) |
| Auth | Không cần | Bearer token / OAuth / mTLS |
| Phạm vi truy cập | Toàn bộ | Scopes giới hạn từng client |

**Server** — `03-production/auth_server.py`:

```python
from mcp.server.auth.provider import AccessToken, TokenVerifier
from mcp.server.auth.settings import AuthSettings
from mcp.server.fastmcp import FastMCP

VALID_TOKENS: dict[str, str] = {
    "dev-token-abc123": "dev-user",
    "prod-key-xyz789": "prod-service",
}


class StaticTokenVerifier(TokenVerifier):
    """Production nên thay bằng JWT decode hoặc OAuth introspection."""

    async def verify_token(self, token: str) -> AccessToken | None:
        client_id = VALID_TOKENS.get(token)
        if client_id is None:
            return None
        return AccessToken(token=token, client_id=client_id, scopes=["weather:read"])


mcp = FastMCP(
    "weather-secure",
    host="0.0.0.0",
    port=8000,
    auth=AuthSettings(
        issuer_url="http://localhost:8000",
        resource_server_url="http://localhost:8000",
    ),
    token_verifier=StaticTokenVerifier(),
)

_MOCK_DB = {
    "Hanoi": "29°C, trời mưa",
    "Haiphong": "33°C, mưa rào",
    "Danang": "30°C, nhiều mây",
}


@mcp.tool()
def get_weather(city: str) -> str:
    """Lấy thời tiết hiện tại của một thành phố."""
    return f"{city}: {_MOCK_DB.get(city, '28°C, không có dữ liệu chi tiết')}"


if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

**Client** — gửi bearer token khi kết nối qua HTTP:

```python
import asyncio

import httpx

from mcp import ClientSession
from mcp.client.streamable_http import streamable_http_client

SERVER_URL = "http://localhost:8000/mcp"
TOKEN = "dev-token-abc123"


async def main() -> None:
    http_client = httpx.AsyncClient(
        headers={"Authorization": f"Bearer {TOKEN}"},
    )

    async with http_client:
        async with streamable_http_client(SERVER_URL, http_client=http_client) as (
            read,
            write,
            _get_session_id,
        ):
            async with ClientSession(read, write) as session:
                await session.initialize()

                tools = await session.list_tools()
                print("Tools (có auth):")
                for t in tools.tools:
                    print(f"  - {t.name}: {t.description}")

                result = await session.call_tool("get_weather", {"city": "Hanoi"})
                print(f"\nKết quả: {result.content[0].text}")


if __name__ == "__main__":
    asyncio.run(main())
```

> Không có token → server trả 401. Token sai → 403. Logic tool không cần biết gì về auth — SDK xử lý hết ở tầng transport.

### 2. Tool Registry & Discovery

**Vấn đề:** Trong demo, client hard-code "dùng tool `get_weather` từ file `weather_server.py`". Khi hệ thống có hàng chục MCP server, mỗi server vài tool — agent không biết tool nào tồn tại, nằm ở server nào, cần auth gì.

**Giải pháp:** Dùng một **Tool Registry** — danh mục trung tâm liệt kê **tất cả tool** từ mọi server. Agent khám phá tool tại runtime theo yêu cầu task:

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

Registry là **tool-centric**, không phải server-centric. Đơn vị khám phá là **tool** (với tag, description, parameters), không phải server.

| | Hard-code (demo) | Tool Registry (production) |
|---|---|---|
| Agent biết tool nào? | Chỉ tool được code sẵn | Tất cả tool trong registry |
| Tìm tool | Theo tên cố định | Theo tag, keyword, capability |
| Thêm tool mới | Sửa code agent | Thêm entry vào registry |
| Chọn tool | Developer quyết định | Agent tự chọn best match |

**Tool Registry** — `03-production/registry.json` (production: thay bằng DB/API với semantic search):

```json
{
  "tools": {
    "get_weather": {
      "description": "Lấy thời tiết hiện tại của một thành phố",
      "parameters": {"city": {"type": "string", "required": true}},
      "tags": ["weather", "current", "vietnam"],
      "server": "weather",
      "version": "1.0.0",
      "deprecated": false
    },
    "get_weather_v2": {
      "description": "Lấy thời tiết chi tiết — JSON, hỗ trợ forecast và đơn vị đo",
      "parameters": {
        "city": {"type": "string", "required": true},
        "include_forecast": {"type": "boolean", "required": false},
        "units": {"type": "string", "required": false}
      },
      "tags": ["weather", "forecast", "detailed", "vietnam"],
      "server": "weather-v2",
      "version": "2.0.0",
      "deprecated": false
    },
    "send_email": {
      "description": "Gửi email tới một hoặc nhiều người nhận",
      "tags": ["email", "notification"],
      "server": "email-service",
      "version": "1.0.0"
    }
  },
  "servers": {
    "weather":       {"transport": "stdio", "command": "python", "args": ["weather_server.py"]},
    "weather-v2":    {"transport": "stdio", "command": "python", "args": ["versioned_server.py"]},
    "email-service": {"transport": "streamable-http", "url": "http://internal-email:9000/mcp",
                      "auth": {"type": "bearer", "token_env": "EMAIL_MCP_TOKEN"}}
  }
}
```

**Registry Client** — agent tra cứu tool theo task requirements, tự kết nối:

```python
import json
import sys

from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client


class ToolRegistry:
    """Danh mục trung tâm — agent tra cứu tool theo tag, keyword, hoặc capability."""

    def __init__(self, path):
        with open(path) as f:
            data = json.load(f)
        self.tools = data["tools"]
        self.servers = data["servers"]

    def search(self, *, tag=None, keyword=None):
        """Tìm tool theo tag hoặc keyword trong description."""
        results = []
        for tool_name, cfg in self.tools.items():
            if tag and tag in cfg.get("tags", []):
                results.append({"tool": tool_name, **cfg, "server_cfg": self.servers[cfg["server"]]})
            elif keyword and keyword.lower() in cfg.get("description", "").lower():
                results.append({"tool": tool_name, **cfg, "server_cfg": self.servers[cfg["server"]]})
        return results

    def best_match(self, **kwargs):
        """Tool phù hợp nhất — ưu tiên version cao, không deprecated."""
        results = self.search(**kwargs)
        active = [r for r in results if not r.get("deprecated")]
        return max(active or results, key=lambda r: r["version"])


async def main():
    registry = ToolRegistry("registry.json")

    # Agent nhận task "lấy thời tiết" → tìm tool có tag "weather"
    best = registry.best_match(tag="weather")
    print(f"Best match: {best['tool']} v{best['version']} (server: {best['server']})")

    # Kết nối tới server và gọi tool — hoàn toàn dynamic
    srv = best["server_cfg"]
    params = StdioServerParameters(command=sys.executable, args=srv["args"])
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            result = await session.call_tool(best["tool"], {"city": "Hanoi"})
            print(f"Kết quả: {result.content[0].text}")
```

> Agent không biết trước "dùng tool nào, từ server nào". Nó hỏi registry theo task requirements → registry trả về best match → agent tự kết nối và gọi.

### 3. Versioning & Backward Compatibility

**Vấn đề:** Server v1 có tool `get_weather(city)` trả chuỗi đơn giản. V2 muốn trả JSON chi tiết, thêm tham số `include_forecast`, đổi format. Nếu đổi trực tiếp — **mọi client cũ sẽ break**.

**Giải pháp — 3 kỹ thuật kết hợp:**

| Kỹ thuật | Mô tả |
|---|---|
| **Tool mới song song** | Thêm `get_weather_v2` bên cạnh `get_weather`, không xoá tool cũ |
| **Tham số optional** | Tham số mới (`include_forecast`, `units`) có default → client cũ gọi vẫn đúng |
| **Server metadata** | Công bố version + deprecation notice qua MCP resource |

```
Server v2
├── get_weather(city)           ← v1, deprecated nhưng vẫn hoạt động
├── get_weather_v2(city, ...)   ← v2, thêm tính năng
└── resource server://info      ← version + migration guide
```

**Server** — `03-production/versioned_server.py`:

```python
import json
from datetime import datetime, timezone

from mcp.server.fastmcp import FastMCP

SERVER_VERSION = "2.0.0"

mcp = FastMCP(
    "weather-v2",
    instructions=f"Weather MCP Server v{SERVER_VERSION}. "
    "Hỗ trợ get_weather (v1, backward compat) và get_weather_v2 (chi tiết hơn).",
)

_MOCK_DB = {
    "Hanoi": {
        "temp": 29, "condition": "trời mưa", "humidity": 82, "wind_speed": 12,
        "forecast": [
            {"day": "tomorrow", "temp": 27, "condition": "mưa nhẹ"},
            {"day": "day_after", "temp": 31, "condition": "nắng"},
        ],
    },
}


# ── Tool v1 (giữ nguyên cho backward compatibility) ──────────────────
@mcp.tool()
def get_weather(city: str) -> str:
    """[v1] Lấy thời tiết hiện tại — trả chuỗi đơn giản. Deprecated, dùng get_weather_v2."""
    data = _MOCK_DB.get(city)
    if data:
        return f"{city}: {data['temp']}°C, {data['condition']}"
    return f"{city}: 28°C, không có dữ liệu chi tiết"


# ── Tool v2 (thêm tính năng, không break v1) ─────────────────────────
@mcp.tool()
def get_weather_v2(
    city: str,
    include_forecast: bool = False,
    units: str = "celsius",
) -> str:
    """[v2] Lấy thời tiết chi tiết — JSON, hỗ trợ forecast và đơn vị đo.

    Args:
        city: Tên thành phố (ví dụ: Hanoi, Danang)
        include_forecast: Có trả thêm dự báo 2 ngày tới không (mặc định: False)
        units: Đơn vị nhiệt độ — "celsius" hoặc "fahrenheit" (mặc định: celsius)
    """
    data = _MOCK_DB.get(city)
    if not data:
        return json.dumps({"city": city, "error": "không có dữ liệu"}, ensure_ascii=False)

    temp = data["temp"]
    if units == "fahrenheit":
        temp = round(temp * 9 / 5 + 32, 1)

    result = {
        "api_version": "2.0",
        "city": city, "temp": temp, "units": units,
        "condition": data["condition"], "humidity": data["humidity"],
        "timestamp": datetime.now(timezone.utc).isoformat(),
    }
    if include_forecast:
        result["forecast"] = data["forecast"]

    return json.dumps(result, ensure_ascii=False)


# ── Resource: server metadata (client kiểm tra version) ──────────────
@mcp.resource("server://info")
def server_info() -> str:
    """Metadata của server — version, deprecated tools, migration guide."""
    return json.dumps({
        "name": "weather-v2",
        "version": SERVER_VERSION,
        "deprecated_tools": ["get_weather"],
        "migration_guide": "Chuyển từ get_weather sang get_weather_v2. "
                           "Tham số 'city' giữ nguyên, thêm include_forecast và units.",
    }, ensure_ascii=False)


if __name__ == "__main__":
    mcp.run()
```

**Client kiểm tra version trước khi gọi:**

```python
async with ClientSession(read, write) as session:
    await session.initialize()

    # Đọc metadata server
    info = await session.read_resource("server://info")
    meta = json.loads(info.contents[0].text)
    print(f"Server: {meta['name']} v{meta['version']}")
    print(f"Deprecated: {meta['deprecated_tools']}")

    # Client thông minh: dùng v2 nếu có, fallback v1
    tools = await session.list_tools()
    tool_names = [t.name for t in tools.tools]

    if "get_weather_v2" in tool_names:
        result = await session.call_tool("get_weather_v2", {
            "city": "Hanoi", "include_forecast": True
        })
    else:
        result = await session.call_tool("get_weather", {"city": "Hanoi"})

    print(result.content[0].text)
```

> Tool cũ vẫn hoạt động cho client legacy. Tool mới thêm tính năng. Resource `server://info` giúp client tự quyết định dùng tool nào.

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

**Tóm lại bằng một câu:** Function Calling là *cơ chế model gọi công cụ*; MCP là *chuẩn để kết nối model với các công cụ đó* — và MCP thực chất dùng Function Calling làm nền tảng để hoạt động.
