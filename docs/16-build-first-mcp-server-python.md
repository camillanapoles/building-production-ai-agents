# Build Your First MCP Server in Python in 10 Minutes

> Artigo #16 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/build-your-first-mcp-server-in-python-in-10-minutes-2eij)

---

MCP (Model Context Protocol) crossed 97 million monthly installs in March 2026 — faster adoption than React hit in its first three years. Chrome just shipped a DevTools MCP server. Microsoft Fabric went GA with MCP support. The ecosystem is converging fast, and if you want your AI agents to talk to anything — a database, an API, a local tool — writing an MCP server is the skill everyone needs right now.

This tutorial skips the theory. In under 800 words and one Python file, you'll have a working MCP server that your AI coding assistant can call.

What You're Building

A simple MCP server that exposes two tools:

get_weather(city) — returns mock weather data for a city (replace with a real API call later)
convert_currency(amount, from_currency, to_currency) — returns mock exchange rates

Your AI agent (Claude, Cursor, Copilot, or any MCP client) will see these tools listed and call them autonomously.

Prerequisites
pip install fastmcp


That's it. FastMCP is the fastest way to write MCP servers in Python. It wraps the official mcp SDK and handles stdio transport, JSON-RPC, and tool registration automatically.

The Code

Create weather_server.py:


```python
from fastmcp import FastMCP

# Initialize the server
mcp = FastMCP("weather-tools", port=8000)

# Mock data — swap these out for real API calls
WEATHER = {
"london": {"temp_c": 12, "condition": "cloudy", "wind_kmh": 18},
"tokyo": {"temp_c": 22, "condition": "clear", "wind_kmh": 8},
"new york": {"temp_c": 18, "condition": "rainy", "wind_kmh": 25},
}

RATES = {
("usd", "eur"): 0.92,
("usd", "gbp"): 0.79,
("eur", "usd"): 1.09,
("gbp", "usd"): 1.27,
}

# Tool 1: Weather lookup
@mcp.tool()
def get_weather(city: str) -> str:
"""Get current weather for a city. Returns temperature in Celsius, condition, and wind speed."""
data = WEATHER.get(city.lower())
if not data:
return f"No weather data for {city}. Supported cities: {', '.join(WEATHER.keys())}"
return f"{city.title()}: {data['temp_c']}\u00b0C, {data['condition']}, wind {data['wind_kmh']} km/h"

# Tool 2: Currency converter
@mcp.tool()
def convert_currency(amount: float, from_currency: str, to_currency: str) -> str:
"""Convert an amount between currencies. Supports USD, EUR, GBP."""
key = (from_currency.lower(), to_currency.lower())
rate = RATES.get(key)
if not rate:
return f"No rate for {from_currency.upper()} \u2192 {to_currency.upper()}. Supported: USD, EUR, GBP"
result = amount * rate
return f"{amount} {from_currency.upper()} = {result:.2f} {to_currency.upper()}"

if __name__ == "__main__":
mcp.run()

```

Run it:


python weather_server.py


The server starts on http://localhost:8000/mcp using Streamable HTTP transport (the MCP default since v2025.03.26 of the spec).

Connect It to Your AI Client
Claude Desktop

Add this to your Claude Desktop config (claude_desktop_config.json):


{
```python
"mcpServers": {
"weather-tools": {
"command": "python",
"args": ["/path/to/weather_server.py"]
}
}
}


Restart Claude Desktop. When you ask "What's the weather in Tokyo?" or "Convert 100 USD to EUR," Claude will see the tools and call them directly.

Cursor / VS Code Copilot
```

Add to your .mcp.json or MCP settings:


{
```python
"mcpServers": {
"weather-tools": {
"command": "python",
"args": ["${workspaceFolder}/weather_server.py"]
}
}
}

Programmatic (Python client)
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client

async def main():
params = StdioServerParameters(
command="python", args=["weather_server.py"]
)
async with stdio_client(params) as streams:
async with ClientSession(*streams) as session:
await session.initialize()
tools = await session.list_tools()
print([t.name for t in tools.tools])
# ['get_weather', 'convert_currency']

import asyncio
asyncio.run(main())

What Just Happened
@mcp.tool() decorator registers each function as an MCP tool. FastMCP auto-generates the JSON Schema from Python type hints — no manual tool definitions needed.
```
Docstrings become tool descriptions. The AI reads these to decide when to call each tool. Write clear, specific descriptions — they're your tool's API docs.
Type hints become validation. If a client passes a string where a float is expected, MCP returns a structured error before your code even runs.
Stdio transport means Claude/Cursor/VS Code spawn your script as a subprocess. No network ports, no auth — it's the simplest and most secure transport for local dev.
Next Steps

Replace the mock data with real API calls:


```python
import httpx

@mcp.tool()
def get_weather(city: str) -> str:
"""Get current weather for a city using OpenWeatherMap API."""
resp = httpx.get(
f"https://api.openweathermap.org/data/2.5/weather",
params={"q": city, "appid": "YOUR_API_KEY", "units": "metric"}
)
data = resp.json()
return f"{city.title()}: {data['main']['temp']}\u00b0C, {data['weather'][0]['description']}"

```

You can also add tools that write files, query databases, call internal APIs, or trigger CI/CD pipelines. The pattern is the same: decorate a function, write a clear docstring, and your AI agent gains a new capability.

The official MCP specification and FastMCP docs cover advanced patterns: resources, prompts, progress reporting, and OAuth authentication — all worth exploring once your first server is running.

TL;DR: MCP isn't a framework to learn — it's a protocol to connect. One Python file, three lines per tool, and your AI agent suddenly has hands.