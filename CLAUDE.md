# CLAUDE.md

Este arquivo fornece orientações para o Claude Code (claude.ai/code) ao trabalhar com o código deste repositório.

## Projeto

Um pequeno BFF (Backend for Frontend) educacional construído com FastAPI. Ele faz proxy para a API pública [DummyJSON Recipes](https://dummyjson.com/docs/recipes), adicionando uma camada de autenticação por chave de API na frente dela. Os scripts `hello.py` e `mensagem.py` na raiz são exercícios de aprendizado sem relação com o BFF.

## Configuração do ambiente

- Requer Python 3.12+.
- O ambiente virtual fica em `venv/` (não versionado). Ative antes de executar qualquer coisa:
  - Windows: `venv\Scripts\activate` (ou `source venv/Scripts/activate` no Git Bash)
  - Linux/Mac: `source venv/bin/activate`
- Instale as dependências: `pip install -r requirements.txt` (use `requirements_win.txt` no Windows — é o mesmo conjunto, com diferenças de `uvloop` para essa plataforma).
- A chave de API é lida do `.env` (via `python-dotenv`) como `API_KEY`. Todo endpoint do BFF exige que o cliente envie essa chave no header `X-API-Key`.

## Executando o servidor

```bash
uvicorn bff.main:app --reload
```
ou, conforme o README:
```bash
dotenv run fastapi dev bff/main.py
```
O servidor roda em `http://127.0.0.1:8000`; a documentação interativa fica em `http://127.0.0.1:8000/docs`.

Não há testes automatizados, configuração de lint ou passos de build neste repositório atualmente.

## Arquitetura

Tudo está concentrado em `bff/main.py` — um app FastAPI de módulo único:

- **Autenticação**: `get_api_key` é uma dependência do FastAPI (`Depends`) que valida o header `X-API-Key` contra o `API_KEY` do ambiente, retornando `401` em caso de divergência. Ela é anexada por rota via `dependencies=[Depends(get_api_key)]`, não globalmente — novas rotas precisam declará-la explicitamente.
- **Cliente upstream**: `dummyjson_get()` é o ponto único de chamada para `https://dummyjson.com`. Ela encapsula o `httpx.AsyncClient` com timeout de 10s e traduz falhas upstream em `HTTPException` (`HTTPStatusError` → repassa o status code, `RequestError` → `502`, qualquer outra exceção → `500`). Novos endpoints que chamem o DummyJSON devem usar esse helper em vez de instanciar seu próprio cliente.
- **Endpoints**: handlers de rota enxutos que validam/restringem parâmetros de query via `Query`/`Path` do FastAPI (ex.: `limit` limitado a 50, `recipe_id` entre 1–50) e delegam para `dummyjson_get`.
