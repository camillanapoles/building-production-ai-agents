# Construa seu primeiro servidor MCP em Python em 10 minutos

> Artigo #16 da série **Building Production AI Agents** por *The Daily Agent*
>
> 📎 [Link original](https://dev.to/thedailyagent/build-your-first-mcp-server-in-python-in-10- Minutes-2eij)

---

O MCP (Model Context Protocol) ultrapassou 97 milhões de instalações mensais em março de 2026 – adoção mais rápida do que o React atingiu em seus primeiros três anos. O Chrome acaba de enviar um servidor DevTools MCP. O Microsoft Fabric tornou-se GA com suporte MCP. O ecossistema está convergindo rapidamente e se você deseja que seus agentes de IA se comuniquem com qualquer coisa — um banco de dados, uma API, uma ferramenta local — escrever um servidor MCP é a habilidade que todos precisam no momento.

Este tutorial ignora a teoria. Em menos de 800 palavras e um arquivo Python, você terá um servidor MCP funcional que seu assistente de codificação de IA pode chamar.

O que você está construindo

Um servidor MCP simples que expõe duas ferramentas:

get_weather(city) — retorna dados meteorológicos simulados para uma cidade (substitua por uma chamada de API real posteriormente)
convert_currency(amount, from_currency, to_currency) — retorna taxas de câmbio simuladas

Seu agente de IA (Claude, Cursor, Copilot ou qualquer cliente MCP) verá essas ferramentas listadas e as chamará de forma autônoma.

Pré-requisitos
instalação pip rápidamcp


É isso. FastMCP é a maneira mais rápida de escrever servidores MCP em Python. Ele envolve o SDK oficial mcp e lida com transporte stdio, JSON-RPC e registro de ferramenta automaticamente.

O Código

Crie weather_server.py:


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

Execute:


python weather_server.py


O servidor inicia em http://localhost:8000/mcp usando transporte HTTP Streamable (o padrão MCP desde v2025.03.26 da especificação).

Conecte-o ao seu cliente AI
Claude Desktop

Adicione isto à configuração do Claude Desktop (claude_desktop_config.json):


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

Adicione às suas configurações .mcp.json ou MCP:


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
Docstrings tornam-se descrições de ferramentas. A IA os lê para decidir quando chamar cada ferramenta. Escreva descrições claras e específicas — elas são os documentos da API da sua ferramenta.
As dicas de tipo tornam-se validação. Se um cliente passar uma string onde um float é esperado, o MCP retornará um erro estruturado antes mesmo de seu código ser executado.
Transporte Stdio significa que Claude/Cursor/VS Code gera seu script como um subprocesso. Sem portas de rede, sem autenticação — é o transporte mais simples e seguro para desenvolvedores locais.
Próximas etapas

Substitua os dados simulados por chamadas de API reais:


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

Você também pode adicionar ferramentas que gravam arquivos, consultam bancos de dados, chamam APIs internas ou acionam pipelines de CI/CD. O padrão é o mesmo: decore uma função, escreva uma doutrina clara e seu agente de IA ganha uma nova capacidade.

A especificação oficial do MCP e os documentos do FastMCP cobrem padrões avançados: recursos, prompts, relatórios de progresso e autenticação OAuth – tudo que vale a pena explorar quando seu primeiro servidor estiver em execução.

DR: MCP não é uma estrutura para aprender – é um protocolo para conectar. Um arquivo Python, três linhas por ferramenta e seu agente de IA de repente terá mãos.