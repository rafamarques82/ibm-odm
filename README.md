# Decision Center MCP

Servidor **MCP (Model Context Protocol)** para integrar clientes MCP com o **IBM ODM Decision Center** via REST API.  
Este projeto expõe ferramentas para listar Decision Services e seus artefatos, executar test suites, acompanhar relatórios e realizar deploy de RuleApps no RES.

> **Autor:** Rafael Eduardo Marques — IBM Hybrid Cloud Specialist  
> **Local:** São Paulo, SP

---

## Sumário

- [Visão geral](#-visão-geral)
- [Arquitetura e funcionamento](#-arquitetura-e-funcionamento)
- [Pré-requisitos](#-pré-requisitos)
- [Instalação](#-instalação)
- [Configuração](#-configuração)
- [Execução](#-execução)
- [Ferramentas MCP disponíveis](#-ferramentas-mcp-disponíveis)
- [Fluxo recomendado (Gate de Teste)](#-fluxo-recomendado-gate-de-teste)
- [Deploy com Gate de Teste (exemplo de código)](#-deploy-com-gate-de-teste-exemplo-de-código)
- [Erros, Timeouts e Retries](#-erros-timeouts-e-retries)
- [Paginação](#-paginação)
- [Observabilidade e Logs](#-observabilidade-e-logs)
- [Segurança](#-segurança)
- [Troubleshooting](#-troubleshooting)
- [Roadmap](#-roadmap)
- [Contribuição](#-contribuição)
- [Licença](#-licença)

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

## 🔒 Deploy com Gate de Teste (exemplo de código)

> Ferramenta MCP que **executa o test suite**, faz **poll** até estado final e só **deploy** se `PASSED`.  
> Ajuste `poll_interval_sec` e `max_attempts` conforme o tempo típico de execução no seu ambiente.

```python
import time
from typing import Dict, Any, Optional

@mcp.tool()
def deploy_with_test_gate(
    decision_service_id: str,
    test_suite_id: str,
    deployment_id: str,
    poll_interval_sec: int = 2,
    max_attempts: int = 60
) -> Dict[str, Any]:
    """
    Executa o test suite (gate), espera concluir e só faz deploy se o status for PASSED.
    Retorna um resumo com testReportId, testStatus e deployResult (se aplicável).
    """

    # 1) Executa test suite
    run = dc_post(f"testsuites/{test_suite_id}/run")
    report_id = (run or {}).get("testReportId")
    if not report_id:
        return {"error": "Test suite não retornou testReportId", "runResult": run}

    # 2) Poll do test report até finalizar
    status: Optional[str] = None
    attempts = 0
    last_report: Dict[str, Any] = {}
    terminal_statuses = {"PASSED", "FAILED", "ERROR", "CANCELED"}
    while attempts < max_attempts:
        rep = dc_get(f"testreports/{report_id}")
        last_report = rep if isinstance(rep, dict) else {}
        status = last_report.get("status")
        if status in terminal_statuses:
            break
        time.sleep(poll_interval_sec)
        attempts += 1

    if status is None:
        return {"error": "Falha ao obter status do relatório de teste", "testReportId": report_id}

    # 3) Gate: só faz deploy se PASSED
    if status != "PASSED":
        return {
            "decisionServiceId": decision_service_id,
            "testReportId": report_id,
            "testStatus": status,
            "message": "Gate reprovado: deploy não executado."
        }

    # 4) Deploy
    deploy_res = dc_post(f"deployments/{deployment_id}/deploy")

    return {
        "decisionServiceId": decision_service_id,
        "testReportId": report_id,
        "testStatus": status,
        "deployResult": deploy_res
    }
```

---

## 🧪 Erros, Timeouts e Retries

- `dc_get`: timeout padrão **60s**.  
- `dc_post`: timeout padrão **120s**.
- Em erros de rede/HTTP, retorna `{ "error": "<mensagem>" }`.

**Melhorias recomendadas:**
- Adicionar **retries** com backoff exponencial (`urllib3.Retry`) para timeouts/5xx.
- Diferenciar erros **4xx** (cliente/configuração) de **5xx** (servidor).
- Permitir configurar **timeouts** via variáveis de ambiente (p.ex. `DC_GET_TIMEOUT`, `DC_POST_TIMEOUT`).

---

## 📄 Paginação

Alguns endpoints (`decisionservices`, etc.) podem **paginar** (`elements`, `offset`, `limit`).  
Implemente loop de paginação quando necessário para coletar todas as páginas e agregar `elements`.

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

## 📈 Roadmap

- [ ] `deploy_with_test_gate` com retries e timeouts configuráveis
- [ ] Paginação automática de `elements`
- [ ] HTTPS com `sess.verify` e CA interno
- [ ] Observabilidade (latência, correlation-id)
- [ ] Validação adicional de pré-condições de deploy (branch/labels)

---

## 🤝 Contribuição

1. Crie um branch: `feature/minha-melhoria`  
2. Commit: `git commit -m "Implementa X"`  
3. Push: `git push origin feature/minha-melhoria`  
4. Abra um **Pull Request** com descrição e exemplos.

---

## 📜 Licença

Defina a licença conforme as políticas da sua organização (ex.: `MIT`, `Apache-2.0` ou **proprietária** para uso interno).
