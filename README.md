# Decision Center MCP

Servidor **MCP (Model Context Protocol)** para integrar clientes MCP com o **IBM ODM Decision Center** via REST API.  
Este projeto expõe ferramentas para listar Decision Services e seus artefatos, executar test suites, acompanhar relatórios e realizar deploy de RuleApps no RES.

> **Autor:** Rafael Eduardo Marques 
---

## Sumário

- [Visão geral](#-visão-geral)
- [Arquitetura e funcionamento](#-arquitetura-e-funcionamento)
- [Pré-requisitos](#-pré-requisitos)
- [Instalação](#-instalação)
- [Configuração](#-configuração)
- [Execução](#-execução)
- [Ferramentas MCP disponíveis](#-ferramentas-mcp-disponíveis)
- [Fluxo recomendado (Gate de Teste)](#-ferramentas-mcp-disponíveis)
- [Exemplo de código](#-erros-timeouts-e-retries)
- [Observabilidade e Logs](#-observabilidade-e-logs)
- [Segurança](#-segurança)
- [Troubleshooting](#-troubleshooting)


---

## 🎯 Visão geral

Este servidor MCP permite que ferramentas/clients compatíveis com MCP (p.ex., Claude Desktop) consumam operações do **Decision Center** sem precisar chamar diretamente a REST API.  
As principais funcionalidades são:

- Listar **Decision Services**, **branches** e **test suites**;
- Executar test suites e consultar **test reports**;
- Listar **targets/servers** para deploy (RES);
- Realizar **deploy** de RuleApps (recomendado com **gate de teste**).

---

## 🏗️ Arquitetura e funcionamento

- O servidor MCP é criado com `FastMCP("Decision Center MCP")`.
- As ferramentas MCP (`@mcp.tool()`) chamam helpers HTTP (`dc_get` / `dc_post`) que usam `requests.Session` com **HTTP Basic Auth**.
- O servidor pode operar em dois modos:
  - **STDIO** (ideal para clientes MCP que usam canal padrão de I/O);
  - **HTTP/ASGI** (via Uvicorn), expondo um endpoint web para consumo.

**Detalhes técnicos do código:**
- Logging é direcionado para **STDERR** (para não poluir o framing JSON no STDOUT).
- STDOUT é reconfigurado para **CRLF + flush imediato** em ambientes sensíveis a framing.
- `build_session()` injeta `Accept: application/json` e credenciais de **Decision Center**.
- `_url()` garante composição adequada do caminho (`base.rstrip('/') + path.lstrip('/')`).

---

## ✅ Pré-requisitos

- Python **3.10+**
- Pacotes:
  - `requests`
  - `uvicorn` (apenas para modo HTTP/ASGI)
  - SDK MCP: `mcp.server.fastmcp` (disponível na sua distribuição/cliente MCP)

---

## 📥 Instalação

```bash
# (Opcional) Ambiente virtual
python -m venv .venv
source .venv/bin/activate   # Linux/macOS
# .venv\Scripts\activate    # Windows

# Instalar dependências
pip install requests uvicorn
# O SDK MCP (mcp.server.fastmcp) deve estar presente no seu ambiente MCP
```

> Se o SDK MCP não estiver instalado no seu Python, ele provavelmente é fornecido/embutido pelo cliente MCP (ex.: Claude Desktop). Execute o servidor a partir do ambiente correto.

---

## ⚙️ Configuração

Defina variáveis de ambiente (recomendado) ou ajuste no código:

```env
DC_BASE_URL=http://my-odm.ibm.com:9060/decisioncenter-api/v1
DC_USERNAME=odmAdmin
DC_PASSWORD=odmAdmin

# MCP
MCP_TRANSPORT=stdio    # ou "http"
HTTP_HOST=127.0.0.1
HTTP_PORT=8000
JSON_RESPONSE=false    # "true" para forçar respostas JSON-only no MCP
```

> **Boas práticas de segurança**
> - Use variáveis de ambiente (evite credenciais hardcoded).
> - Em HTTPS interno, configure verificação de certificado (`sess.verify`) com CA corporativo.
> - Restrinja o acesso se expuser o modo HTTP (rede interna, autenticação adicional).

---

## 🚀 Execução

### Modo STDIO (padrão; ideal para clientes MCP)
```bash
python app.py
```

### Modo HTTP/ASGI (expondo endpoint MCP)
```bash
export MCP_TRANSPORT=http
python app.py
# Aplicação disponível em: http://127.0.0.1:8000/mcp
```

O código tenta `mcp.streamable_http_app()` (SDK mais novo) e faz fallback para `mcp.http_app()` se necessário.

---

## 🧰 Ferramentas MCP disponíveis

Todas abaixo são funções anotadas com `@mcp.tool()`:

- `about()`  
  Retorna informações do endpoint `/about` do Decision Center.

- `list_decision_services()`  
  Lista os **Decision Services** do repositório.

- `list_branches(decision_service_id: str)`  
  Lista **branches** de um Decision Service.

- `list_test_suites(decision_service_id: str)`  
  Lista **test suites** associados a um Decision Service.

- `run_test_suite(test_suite_id: str)`  
  Executa um test suite; normalmente retorna `testReportId` e `status`.

- `get_test_report(test_report_id: str)`  
  Consulta **detalhes/estado** do test report (`PASSED`, `FAILED`, etc.).

- `list_servers()`  
  Lista **targets/servidores RES** disponíveis para deploy.

- `deploy_ruleapp(deployment_id: str)` ⚠️  
  Executa **deploy** da RuleApp para o **deployment configuration ID** indicado.  
  > **Atenção:** a função **não valida** resultados de testes. Para conformidade, use **gate de teste** (veja abaixo).

- `find_decision_service_id_by_name(name: str) -> Optional[str>`  
  Busca o **ID** de um Decision Service pelo **nome** (retorna ID ou `null`).

- `get_deployments(decision_service_id: str)`  
  Lista **deployment configurations** de um Decision Service.

- `get_branches_and_test_suites(name: str)`  
  Conveniência: dado o **nome** do DS, retorna **branches** e **test suites**.

---

## ✅ Fluxo recomendado (Gate de Teste)

1. **Localizar o Decision Service**:
   ```python
   find_decision_service_id_by_name("Meu Decision Service")
   ```
2. **Listar test suites**:
   ```python
   list_test_suites(decision_service_id="...")
   ```
3. **Executar o test suite**:
   ```python
   run_test_suite(test_suite_id="...")
   # → retorna algo como {"testReportId": "...", "status": "RUNNING"}
   ```
4. **Consultar o test report**:
   ```python
   get_test_report(test_report_id="...")
   # Aguardar status "PASSED" para prosseguir
   ```
5. **Listar deployments**:
   ```python
   get_deployments(decision_service_id="...")
   # Selecionar deployment_id apropriado
   ```
6. **Somente se PASSED**, **executar deploy**:
   ```python
   deploy_ruleapp(deployment_id="...")
   ```

> **Regra de ouro:** Não faça deploy sem validação de teste **PASSED**.  
> O snippet de **Gate de Teste** abaixo automatiza essa validação.

---

## 🔒 Exemplo de código

> Ferramenta MCP que **executa o test suite**, faz **poll** até estado final e só **deploy** se `PASSED`.  
> Ajuste `poll_interval_sec` e `max_attempts` conforme o tempo típico de execução no seu ambiente.

```python

import os
import sys
import logging
from typing import Dict, Any, Optional

import requests
from requests.auth import HTTPBasicAuth
from mcp.server.fastmcp import FastMCP

# ------------------------------------------------------------------
# Logging: somente em STDERR (nada em stdout além de mensagens JSON)
# ------------------------------------------------------------------
logging.basicConfig(
    stream=sys.stderr,
    level=logging.INFO,
    format='%(asctime)s %(levelname)s %(name)s: %(message)s'
)
log = logging.getLogger("decision-center-mcp")

# Em STDIO, alguns stacks são sensíveis a framing; CRLF + flush imediato
try:
    sys.stdout.reconfigure(newline='\r\n', write_through=True)
except Exception:
    pass

# ------------------------------------------------------------------
# Configuração (validação adiada por ferramenta)
# ------------------------------------------------------------------
# --- Configuration ---
DC_BASE_URL = "http://my-odm.ibm.com:9060/decisioncenter-api/v1"
DC_USERNAME ="odmAdmin"
DC_PASSWORD = "odmAdmin"

MCP_TRANSPORT = os.getenv("MCP_TRANSPORT", "stdio").strip().lower()
HTTP_HOST = os.getenv("HTTP_HOST", "127.0.0.1").strip()
HTTP_PORT = int(os.getenv("HTTP_PORT", "8000"))
JSON_RESPONSE = os.getenv("JSON_RESPONSE", "false").lower() == "true"

def ensure_config() -> Optional[Dict[str, Any]]:
    missing = []
    if not DC_BASE_URL: missing.append("DC_BASE_URL")
    if not DC_USERNAME: missing.append("DC_USERNAME")
    if not DC_PASSWORD: missing.append("DC_PASSWORD")
    if missing:
        msg = f"Decision Center config missing: {', '.join(missing)}"
        log.error(msg)
        return {"error": msg}
    return None

def build_session() -> Optional[requests.Session]:
    err = ensure_config()
    if err:
        return None
    sess = requests.Session()
    sess.auth = HTTPBasicAuth(DC_USERNAME, DC_PASSWORD)
    sess.headers.update({"Accept": "application/json"})
    return sess

def _url(path: str) -> str:
    base = DC_BASE_URL.rstrip('/')
    path = path.lstrip('/')
    return f"{base}/{path}"

def dc_get(path: str, params: Optional[Dict[str, Any]] = None) -> Dict[str, Any]:
    sess = build_session()
    if sess is None:
        return {"error": "Configuration missing"}
    try:
        r = sess.get(_url(path), params=params, timeout=60)
        r.raise_for_status()
        return r.json()
    except requests.exceptions.RequestException as e:
        log.error(f"GET {path} failed: {e}")
        return {"error": str(e)}
    except ValueError:
        log.error(f"GET {path} returned non-JSON body")
        return {"error": "non-json-response"}

def dc_post(path: str, json: Optional[Dict[str, Any]] = None) -> Dict[str, Any]:
    sess = build_session()
    if sess is None:
        return {"error": "Configuration missing"}
    try:
        r = sess.post(_url(path), json=json, timeout=120)
        r.raise_for_status()
        try:
            return r.json()
        except ValueError:
            return {"status": r.status_code}
    except requests.exceptions.RequestException as e:
        log.error(f"POST {path} failed: {e}")
        return {"error": str(e)}

# ------------------------------------------------------------------
# MCP Server (SDK oficial)
# ------------------------------------------------------------------
mcp = FastMCP("Decision Center MCP", json_response=JSON_RESPONSE)

@mcp.tool()
def about() -> Dict[str, Any]:
    """Informações do endpoint /about do Decision Center REST API."""
    return dc_get("about")

@mcp.tool()
def list_decision_services() -> Dict[str, Any]:
    """Lista Decision Services do repositório."""
    return dc_get("decisionservices")

@mcp.tool()
def list_branches(decision_service_id: str) -> Dict[str, Any]:
    """Lista branches de um Decision Service (por ID)."""
    return dc_get(f"decisionservices/{decision_service_id}/branches")

@mcp.tool()
def list_test_suites(decision_service_id: str) -> Dict[str, Any]:
    """Lista test suites de um Decision Service (por ID)."""
    return dc_get(f"decisionservices/{decision_service_id}/testsuites")

@mcp.tool()
def run_test_suite(test_suite_id: str) -> Dict[str, Any]:
    """Executa test suite por ID; retorna testReportId/status."""
    return dc_post(f"testsuites/{test_suite_id}/run")

@mcp.tool()
def get_test_report(test_report_id: str) -> Dict[str, Any]:
    """Busca status/detalhes de um test report por ID."""
    return dc_get(f"testreports/{test_report_id}")

@mcp.tool()
def list_servers() -> Dict[str, Any]:
    """Lista servidores disponíveis para deploy (RES targets)."""
    return dc_get("servers")

@mcp.tool()
def deploy_ruleapp(deployment_id: str) -> Dict[str, Any]:
    """Faz deploy de RuleApp ao RES para um deployment configuration ID. antes de fazer o deploy, verificar se tem um relatorio de teste associado e se o mesmo foi executado com sucesso. se não foi executado com sucesso, não iniciar o deploy."""
    return dc_post(f"deployments/{deployment_id}/deploy")

@mcp.tool()
def find_decision_service_id_by_name(name: str) -> Optional[str]:
    """Localiza o ID de um Decision Service pelo nome. Retorna ID ou null."""
    data = dc_get("decisionservices")
    elems = data.get("elements", []) if isinstance(data, dict) else []
    for el in elems:
        if el.get("name") == name:
            return el.get("id")
    return None

@mcp.tool()
def get_deployments(decision_service_id: str) -> Dict[str, Any]:
    """Lista deployment configurations de um Decision Service (se disponível)."""
    return dc_get(f"decisionservices/{decision_service_id}/deployments")

@mcp.tool()
def get_branches_and_test_suites(name: str) -> Dict[str, Any]:
    """Convenience: dado o nome, retorna branches e test suites."""
    dsid = find_decision_service_id_by_name(name)
    if not dsid:
        return {"error": f"Decision Service '{name}' not found"}
    branches = dc_get(f"decisionservices/{dsid}/branches")
    testsuites = dc_get(f"decisionservices/{dsid}/testsuites")
    return {"decisionServiceId": dsid, "branches": branches, "testsuites": testsuites}

# ------------------------------------------------------------------
# Inicialização do servidor — STDIO ou HTTP/ASGI
#   - No SDK oficial, `run()` não aceita host/port; para HTTP use ASGI
#   - Para STDIO (Claude Desktop), use run(transport="stdio")
# ------------------------------------------------------------------
if __name__ == "__main__":
    try:
        if MCP_TRANSPORT == "http":
            # Modo HTTP via ASGI (Uvicorn) — JSON-only recomendado para teste
            try:
                # Preferível quando disponível na sua versão do SDK
                app = mcp.streamable_http_app()
            except AttributeError:
                # Fallback em versões que expõem apenas http_app()
                app = mcp.http_app()
            log.info(f"Starting ASGI (HTTP) at http://{HTTP_HOST}:{HTTP_PORT}/mcp")
            import uvicorn
            uvicorn.run(app, host=HTTP_HOST, port=HTTP_PORT)
        else:
            log.info("Starting MCP server (STDIO)")
            mcp.run(transport="stdio")
    except Exception as e:
        log.exception(f"Fatal error in MCP server: {e}")

```

---


## 📊 Observabilidade e Logs

- Logs vão **somente para STDERR**, preservando o **STDOUT** para mensagens JSON do MCP.
- Recomenda-se:
  - Incluir **correlation-id** por requisição;
  - Logar **latência** de chamadas (`GET/POST`);
  - Padronizar mensagens e níveis (`INFO`, `ERROR`).

---

## 🔐 Segurança

- **Credenciais** (`DC_USERNAME`, `DC_PASSWORD`): sempre via **variáveis de ambiente** ou secrets (não hardcode).
- **HTTPS interno**: configure `sess.verify` com CA corporativo para evitar MITM e assegurar trust chain.
- **Exposição HTTP**: se usar `MCP_TRANSPORT=http`, preferir rede interna e, se possível, **autenticação** adicional no front.

---

## 🛠️ Troubleshooting

- **`Configuration missing`**: verifique `DC_BASE_URL`, `DC_USERNAME`, `DC_PASSWORD`.
- **`non-json-response`**: endpoint retornou corpo não-JSON; confirme cabeçalhos `Accept` e versão da API.
- **Timeouts**: ajuste timeouts; adicione retries; valide conectividade/latência até o DC.
- **Deploy sem validação**: prefira `deploy_with_test_gate` para bloquear promoção sem testes aprovados.

---
