# Orcamentos Brasil — Plano de Implementação do MVP

> **Para agentes:** SUB-SKILL OBRIGATÓRIA: use `superpowers:subagent-driven-development` (recomendado) ou `superpowers:executing-plans` para executar este plano tarefa por tarefa. Os passos usam checkbox (`- [ ]`) para acompanhamento.

**Objetivo:** Entregar um micro-SaaS multi-tenant onde prestadores de serviço cadastram clientes e itens, montam orçamentos, geram PDF, compartilham por WhatsApp/link público e acompanham aprovação.

**Arquitetura:** Monorepo com `backend/` (FastAPI + SQLAlchemy 2 + PostgreSQL, banco compartilhado com coluna `company_id` em toda entidade de tenant) e `frontend/` (SPA React). O tenant vem **sempre** do token de acesso, validado contra `company_users` a cada requisição. Em produção, um Caddy serve o build do frontend e faz proxy de `/api/*` para a API, tudo via Docker Compose numa VPS.

**Stack:** Python 3.12, FastAPI, SQLAlchemy 2.0 (sync), Alembic, Pydantic v2, psycopg 3, pwdlib[argon2], PyJWT, WeasyPrint + Jinja2, slowapi, boto3 · React 19 + Vite + TypeScript, Tailwind CSS v4, shadcn/ui, lucide-react, React Router 7, TanStack Query, react-hook-form + zod · PostgreSQL 16 · Docker Compose · Caddy · Cloudflare R2 · GitHub Actions · pytest, Vitest + Testing Library + msw, Playwright.

**Spec:** [docs/visao-do-produto.md](docs/visao-do-produto.md)

**Repositório:** https://github.com/Defendi22/orcamentosbrasil (branch padrão `main`)

---

## Decisões de arquitetura

| Tema | Decisão |
|---|---|
| Multi-tenant | Banco único, `company_id NOT NULL` + índice em toda tabela de tenant. Todo service recebe `Tenant` e filtra por `tenant.company_id`. Recurso de outra empresa → **404** (nunca 403, para não revelar existência). RLS do Postgres fica como reforço futuro. |
| Identidade do tenant | JWT de acesso (15 min, HS256) com `sub`=user_id e `cid`=company_id. `get_tenant` confere se o vínculo em `company_users` existe a cada requisição. Corpo de requisição nunca define `company_id`: todos os schemas de entrada usam `extra="forbid"`. |
| Sessão | Refresh token opaco (32 bytes) em cookie `httpOnly`, `SameSite=Lax`, `path=/api/auth`, 30 dias, guardado como SHA-256 em `refresh_tokens`, com rotação a cada uso. Reuso de token revogado revoga todos os tokens do usuário. Access token fica só em memória no frontend. |
| IDs | UUID v4 em todas as PKs. O número legível do orçamento é `ORC-0001`, sequencial **por empresa**. |
| Dinheiro | Inteiros em centavos (`*_cents BIGINT`). Quantidade `NUMERIC(12,3)`. Arredondamento `ROUND_HALF_UP` por linha e no desconto percentual. O backend é a fonte da verdade; o frontend replica o cálculo só para pré-visualização. |
| Datas | Gravadas em UTC (`timestamptz`). Validade é `date`. "Hoje", "mês corrente" e expiração usam `America/Sao_Paulo`. |
| Status | `draft, sent, viewed, approved, rejected, changes_requested, expired, canceled`. "Aguardando resposta" (spec 6.8) é um **agrupamento** (`sent` + `viewed`), não um status próprio. |
| Versões | Editar um orçamento em `sent/viewed/changes_requested/expired` incrementa `version`, guarda o snapshot anterior no evento `revised` e volta o status para `sent`. As respostas do cliente enviam a `version` que ele viu; se a versão não bate → 409. Orçamentos `approved/rejected/canceled` não podem ser editados. |
| Snapshot do cliente | Os dados do cliente são copiados para o orçamento a cada salvamento, para que editar o cadastro não altere propostas já enviadas. |
| Link público | `secrets.token_urlsafe(32)` gerado no primeiro envio, em coluna única. Revogar = gerar um novo (o antigo passa a dar 404). URL: `{PUBLIC_APP_URL}/o/{token}`. |
| Expiração | Lazy: `expire_overdue()` roda antes da listagem, do painel e da visualização pública. Não há worker no MVP. |
| PDF | WeasyPrint renderizando template Jinja2 (autoescape). Roda no container (as dependências GTK/Pango estão na imagem), então o dev no Windows usa Docker. |
| Arquivos | Interface `Storage` com implementações local (dev) e S3/R2 (prod). Chaves derivadas do servidor: `companies/{company_id}/logo.{ext}`. |
| Planos | Tabela `subscriptions` (`free`/`professional`/`business`). Plano gratuito: 5 orçamentos/mês + marca d'água no PDF, sem logo. O plano é alterado manualmente por CLI; cobrança fica fora do MVP. |
| Erros | `{"detail": {"code": "<snake_case>", "message": "<texto pt-BR>"}}`. Erros de validação do Pydantic continuam com 422 padrão. |

## Estrutura de arquivos

```
.
├── PLAN.md · README.md · .gitignore · .gitattributes · .env.example
├── docker-compose.yml               # dev: db + api
├── docker-compose.prod.yml          # prod: db + api + web(caddy)
├── Caddyfile
├── docker/postgres/init-test-db.sql # cria orcamentos_test
├── scripts/backup.sh · scripts/restore-test.sh
├── docs/visao-do-produto.md · docs/deploy.md
├── .github/workflows/ci.yml
├── backend/
│   ├── pyproject.toml · Dockerfile · alembic.ini · alembic/env.py · alembic/versions/
│   ├── app/
│   │   ├── main.py                  # create_app()
│   │   ├── cli.py                   # set-plan
│   │   ├── core/  config.py db.py errors.py security.py tenancy.py rate_limit.py
│   │   │          br.py (telefone, CPF/CNPJ, formatação) clock.py security_headers.py
│   │   ├── storage/ __init__.py base.py local.py s3.py
│   │   └── modules/
│   │       ├── auth/       models.py schemas.py service.py router.py
│   │       ├── companies/  models.py schemas.py service.py router.py
│   │       ├── clients/    models.py schemas.py service.py router.py
│   │       ├── catalog/    models.py schemas.py service.py router.py templates.py
│   │       ├── quotes/     models.py schemas.py service.py router.py calc.py
│   │       │               numbering.py status.py lifecycle.py events.py
│   │       ├── pdf/        renderer.py templates/quote.html templates/pdf.css
│   │       ├── public/     schemas.py service.py router.py
│   │       ├── dashboard/  schemas.py service.py router.py
│   │       └── account/    service.py router.py
│   └── tests/ conftest.py factories.py test_*.py
└── frontend/
    ├── Dockerfile · vite.config.ts · components.json · e2e/happy-path.spec.ts
    └── src/
        ├── main.tsx · App.tsx
        ├── lib/ api.ts auth.tsx format.ts whatsapp.ts
        ├── components/ui/* (shadcn) · components/layout/AppShell.tsx
        └── features/
            ├── auth/ company/ clients/ catalog/ dashboard/ legal/
            ├── quotes/ api.ts calc.ts QuotesListPage QuoteEditorPage QuoteDetailPage
            │           ItemsEditor ClientPicker CatalogPicker DiscountField
            │           StatusBadge QuoteTimeline ShareDialog
            └── public/ PublicQuotePage ApproveDialog RejectDialog RequestChangesDialog
```

## Restrições globais

- Nome do produto em toda a UI, no PDF e no título das páginas: **Orcamentos Brasil**.
- Idioma da interface: pt-BR. Moeda BRL formatada como `R$ 1.234,56`. Datas como `dd/mm/aaaa`. Fuso `America/Sao_Paulo`.
- Python ≥ 3.12, Node 22 LTS, PostgreSQL 16.
- Nunca aceitar `company_id` vindo do cliente. Todos os schemas Pydantic de entrada usam `model_config = ConfigDict(extra="forbid")`.
- Todo endpoint de tenant tem teste provando que outra empresa recebe 404.
- Sem impostos nem cálculo fiscal. O produto não se apresenta como sistema fiscal ou contábil.
- Fórmulas: `subtotal = soma(quantidade × valor_unitário)`, `total = subtotal − desconto`. Desconto em valor ou percentual, nunca maior que o subtotal.
- Plano Gratuito: até **5 orçamentos/mês** (`FREE_PLAN_MONTHLY_QUOTES=5`) e PDF com a marca Orcamentos Brasil. Preços de referência: Profissional R$ 29,90/mês, Empresa R$ 59,90/mês. Sem cobrança no MVP e sem armazenar dados de cartão.
- Mensagem de WhatsApp exata: `Olá! Preparei seu orçamento. Você pode visualizar os detalhes e responder por este link: {link}`
- A página pública e o PDF nunca exibem `internal_notes` nem dados de outros clientes.
- Senhas com argon2. Mínimo de 8 caracteres.
- Desenvolvimento em branch (`feat/<tarefa>`), PR para `main` com CI verde.

## Foco de revisão

Situações que a spec implica, que um usuário real vai encontrar e que precisam de teste explícito:

1. **Telefones BR em formatos variados** (`(11) 98765-4321`, `+55 11 98765-4321`, `11987654321`, fixo de 10 dígitos): devem virar `5511987654321`, e o link `wa.me` deve funcionar. Número sem DDD → erro claro. *(Teste na Tarefa 4 e na Tarefa 16.)*
2. **Quantidade decimal e vírgula** (`2,5` m × `R$ 37,33`): o total da linha deve ser `R$ 93,33` (half-up), igual no frontend e no backend. *(Tarefas 2, 10 e 13.)*
3. **Desconto inválido** (valor maior que o subtotal, percentual > 100 ou negativo): deve dar 422 com mensagem, nunca um total negativo. *(Tarefas 10 e 13.)*
4. **Cliente respondendo um link "velho"**: aprovar duas vezes, aprovar depois de expirado, aprovar uma versão já substituída ou usar um token regenerado. Deve dar 409/410/404 com mensagem compreensível. *(Tarefa 15.)*
5. **Numeração por empresa**: duas empresas criando orçamentos devem ter sequências independentes (`ORC-0001` em cada uma), sem duplicar número. *(Tarefa 11.)*

## Cronograma (7 dias, agressivo)

| Dia | Tarefas |
|---|---|
| 1 | 1 Fundação backend · 2 Fundação frontend · 3 Auth/tenant · 4 Empresa · 5 Front auth/empresa |
| 2 | 6 Clientes API · 7 Clientes UI |
| 3 | 8 Catálogo API · 9 Catálogo UI |
| 4 | 10 Cálculo · 11 Orçamentos API · 12 Ciclo de vida · 13 Editor UI |
| 5 | 14 PDF · parte de compartilhamento da 16 |
| 6 | 15 Página pública API · 16 Página pública UI |
| 7 | 17 Painel/histórico · 18 Segurança/LGPD · 19 Deploy/backup/E2E |

## Como rodar os testes

- Backend: `docker compose run --rm api pytest -q` (usa o banco `orcamentos_test`; migrações aplicadas uma vez por sessão; cada teste roda dentro de um SAVEPOINT e é revertido).
- Frontend: `cd frontend && npm test -- --run`, mais `npm run typecheck` e `npm run lint`.

---

### Tarefa 1: Fundação do backend (FastAPI, Postgres, Alembic, pytest, CI)

**Arquivos:**
- Criar: `docker-compose.yml`, `.env.example`, `docker/postgres/init-test-db.sql`, `backend/pyproject.toml`, `backend/Dockerfile`, `backend/alembic.ini`, `backend/alembic/env.py`, `backend/app/main.py`, `backend/app/core/{config,db,errors}.py`, `backend/tests/conftest.py`, `.github/workflows/ci.yml`
- Teste: `backend/tests/test_health.py`

**Interfaces:**
- Produz:
  - `create_app() -> FastAPI`. Rotas montadas sob `/api`.
  - `Settings` (pydantic-settings) com os campos `DATABASE_URL`, `TEST_DATABASE_URL`, `JWT_SECRET`, `ACCESS_TOKEN_TTL_MIN=15`, `REFRESH_TOKEN_TTL_DAYS=30`, `PUBLIC_APP_URL`, `COOKIE_SECURE=True`, `CORS_ORIGINS`, `STORAGE_BACKEND="local"`, `STORAGE_LOCAL_DIR="/data/storage"`, `S3_ENDPOINT_URL`, `S3_BUCKET`, `S3_ACCESS_KEY_ID`, `S3_SECRET_ACCESS_KEY`, `FREE_PLAN_MONTHLY_QUOTES=5`, `APP_TIMEZONE="America/Sao_Paulo"`. Acesso via `get_settings() -> Settings` (com `lru_cache`).
  - `Base(DeclarativeBase)` e `IdTimestampMixin`: `id: UUID` (PK, default `uuid4`), `created_at`, `updated_at` (`timestamptz`, `server_default=now()`, `onupdate`).
  - `get_db() -> Iterator[Session]`.
  - `AppError(status_code: int, code: str, message: str)`, com handler que devolve `{"detail": {"code", "message"}}`.
  - Fixtures de pytest: `db: Session` e `api: TestClient` (com `get_db` sobrescrito para usar `db`).

- [ ] **Passo 1: Escrever o teste que falha**

```python
def test_health_reports_db_ok(api):
    r = api.get("/api/health")
    assert r.status_code == 200
    assert r.json() == {"status": "ok", "db": "ok"}

def test_app_error_shape(api):
    r = api.get("/api/_test/error")  # rota registrada só no conftest
    assert r.status_code == 409
    assert r.json() == {"detail": {"code": "conflict_x", "message": "Mensagem"}}
```

- [ ] **Passo 2: Rodar e confirmar que falha**: `docker compose run --rm api pytest tests/test_health.py -v`. Esperado: FAIL (app inexistente).
- [ ] **Passo 3: Implementar**
  - `docker-compose.yml`: serviço `db` (`postgres:16-alpine`, healthcheck `pg_isready`, monta `docker/postgres/init-test-db.sql` em `/docker-entrypoint-initdb.d/`) e serviço `api` (build `./backend`, volume `./backend:/app`, `env_file: .env`, `uvicorn app.main:app --reload --host 0.0.0.0`, porta 8000, `depends_on: db: condition: service_healthy`).
  - Dockerfile: `python:3.12-slim` + `libpango-1.0-0 libpangoft2-1.0-0 libharfbuzz0b fonts-dejavu-core`.
  - Dependências de dev: `pytest`, `httpx`, `ruff`, `freezegun`.
  - `conftest.py`: aplica `alembic upgrade head` no banco de teste uma vez por sessão; por teste, abre uma conexão com transação externa e uma `Session(join_transaction_mode="create_savepoint")`, e faz rollback ao final.
  - CI, job `backend`: serviço `postgres:16`, `apt-get install` das libs do Pango, `pip install -e ".[dev]"`, `ruff check .`, `pytest -q`.
- [ ] **Passo 4: Rodar e confirmar que passa**: o mesmo comando do Passo 2. Esperado: 2 passed.
- [ ] **Passo 5: Commit**: `git commit -m "chore: backend skeleton with health check, test db and CI"`

---

### Tarefa 2: Fundação do frontend (Vite, Tailwind, shadcn, layout responsivo)

**Arquivos:**
- Criar: `frontend/` via `npm create vite@latest frontend -- --template react-ts`, `frontend/vite.config.ts` (proxy `/api` → `http://localhost:8000`, Vitest com `jsdom`), `src/lib/format.ts`, `src/lib/api.ts`, `src/components/layout/AppShell.tsx`, `src/App.tsx`. Job `frontend` em `.github/workflows/ci.yml`.
- Teste: `src/lib/format.test.ts`, `src/lib/api.test.ts`

**Interfaces:**
- Produz:
  - `formatBRL(cents: number): string`, `parseBRNumber(input: string): number | null`, `reaisToCents(input: string): number | null`, `formatDate(isoDate: string): string` (`dd/mm/aaaa`), `formatDateTime(iso: string): string` (no fuso America/Sao_Paulo).
  - `apiFetch<T>(path: string, init?: RequestInit): Promise<T>`. Sempre usa `credentials: "include"` e `Authorization: Bearer` quando houver token. Em 401, tenta `POST /api/auth/refresh` **uma** vez e repete a chamada. Lança `ApiError`.
  - `class ApiError extends Error { status: number; code: string; fields?: Record<string,string> }`.
  - `setAccessToken(token: string | null): void`.
  - `AppShell`: sidebar a partir de `md`, bottom nav abaixo de `md`. Itens: Painel `/app`, Orçamentos `/app/orcamentos`, Clientes `/app/clientes`, Catálogo `/app/catalogo`, Empresa `/app/empresa`.

- [ ] **Passo 1: Escrever os testes que falham**

```ts
expect(formatBRL(123456)).toBe("R$ 1.234,56");
expect(formatBRL(0)).toBe("R$ 0,00");
expect(parseBRNumber("2,5")).toBe(2.5);
expect(parseBRNumber("1.234,56")).toBe(1234.56);
expect(parseBRNumber("abc")).toBeNull();
expect(parseBRNumber("")).toBeNull();
expect(reaisToCents("37,33")).toBe(3733);
expect(reaisToCents("0,1")).toBe(10);
expect(formatDate("2026-12-02")).toBe("02/12/2026");
// api.test.ts (msw): 401 → refresh 200 → repete a chamada e resolve; refresh 401 → lança ApiError status 401
```

- [ ] **Passo 2: Rodar e confirmar que falha**: `cd frontend && npm test -- --run`. Esperado: FAIL.
- [ ] **Passo 3: Implementar.** Instalar `tailwindcss @tailwindcss/vite`, rodar `npx shadcn@latest init` e adicionar `button input label card dialog table badge dropdown-menu sonner select textarea`. Instalar `lucide-react react-router-dom @tanstack/react-query react-hook-form zod @hookform/resolvers`; de dev, `vitest jsdom @testing-library/react @testing-library/user-event msw`. Scripts: `test`, `typecheck` (`tsc -b --noEmit`), `lint`. Usar `Intl.NumberFormat("pt-BR", {style:"currency", currency:"BRL"})`.
- [ ] **Passo 4: Rodar e confirmar que passa.** Também `npm run typecheck && npm run build`.
- [ ] **Passo 5: Commit**: `git commit -m "chore: frontend skeleton with BR formatting, api client and app shell"`

---

### Tarefa 3: Autenticação, empresas e contexto de tenant

**Arquivos:**
- Criar: `backend/app/modules/auth/{models,schemas,service,router}.py`, `backend/app/modules/companies/models.py`, `backend/app/core/{security,tenancy,rate_limit}.py`, `backend/alembic/versions/0001_auth_companies.py`, `backend/tests/factories.py`
- Teste: `backend/tests/test_auth.py`

**Interfaces:**
- Consome: `Base`, `IdTimestampMixin`, `get_db`, `AppError`, `get_settings` (Tarefa 1).
- Produz:
  - Modelos:
    - `User(email citext/lower unique, name, password_hash, terms_accepted_at)`.
    - `Company(name, trade_name, document, phone, email, address, city, state, logo_key, default_validity_days=7, default_payment_terms, default_notes, next_quote_number=1)`.
    - `CompanyUser(company_id, user_id, role: "owner"|"member")`, com UNIQUE `(company_id, user_id)`.
    - `Subscription(company_id unique, plan: "free"|"professional"|"business" = "free", status="active")`.
    - `RefreshToken(user_id, company_id, token_hash unique, expires_at, revoked_at)`.
  - `hash_password(raw) -> str`, `verify_password(raw, hashed) -> bool`, `create_access_token(user_id: UUID, company_id: UUID) -> str`, `decode_access_token(token) -> AccessClaims(user_id, company_id)`, `new_refresh_token() -> tuple[str, str]` (cru, sha256).
  - `@dataclass(frozen=True) class Tenant: company_id: UUID; user_id: UUID; role: str` e a dependência `get_tenant(...) -> Tenant`, que dá 401 `unauthenticated` se o token for inválido ou o vínculo não existir.
  - `limiter` (slowapi), com chave por IP.
  - Endpoints:
    - `POST /api/auth/signup {name, email, password, company_name, accept_terms: true}` → 201 `AuthOut{access_token, user{id,name,email}, company{id,name}}` + cookie `refresh_token`.
    - `POST /api/auth/login {email, password}` → 200 `AuthOut`. Limite de 5/min por IP+e-mail.
    - `POST /api/auth/refresh` (cookie) → 200 `AuthOut` + cookie rotacionado.
    - `POST /api/auth/logout` → 204 e revoga o token.
    - `GET /api/auth/me` → `{user, company, role, plan}`.
  - Factory de teste: `signup(api, *, email=None, company_name="Empresa A") -> AuthCtx(headers: dict, company_id: UUID, user_id: UUID)`.

- [ ] **Passo 1: Escrever os testes que falham**

```python
def test_signup_creates_user_company_owner_and_free_subscription(api, db):
    r = api.post("/api/auth/signup", json={"name": "Ana", "email": "ana@x.com", "password": "senha1234",
                                           "company_name": "Refrigeração Silva", "accept_terms": True})
    assert r.status_code == 201
    assert r.json()["company"]["name"] == "Refrigeração Silva" and r.json()["access_token"]
    assert "refresh_token=" in r.headers["set-cookie"] and "HttpOnly" in r.headers["set-cookie"]
    cid = r.json()["company"]["id"]
    assert db.scalar(select(Subscription.plan).where(Subscription.company_id == cid)) == "free"
    assert db.scalar(select(CompanyUser.role).where(CompanyUser.company_id == cid)) == "owner"

def test_signup_email_is_case_insensitive_unique(api): ...      # 2º "ANA@x.com" → 409 code "email_taken"
def test_signup_requires_terms_and_8_char_password(api): ...    # accept_terms False → 422; "1234567" → 422
def test_login_wrong_password_and_unknown_email_same_error(api): ...  # ambos 401 code "invalid_credentials"
def test_me_requires_token(api): ...                            # 401
def test_refresh_rotates_and_reuse_revokes_all(api): ...        # 2º uso do cookie antigo → 401; o cookie novo também passa a dar 401
def test_token_rejected_when_membership_removed(api, db): ...   # apaga CompanyUser → /me 401
def test_login_rate_limited(api): ...                           # 6ª tentativa em 1 min → 429
```

- [ ] **Passo 2: Rodar e confirmar que falha**: `docker compose run --rm api pytest tests/test_auth.py -v`
- [ ] **Passo 3: Implementar.** O signup roda em uma única transação. O e-mail é normalizado com `strip().lower()`. O cookie usa `secure=settings.COOKIE_SECURE`, `samesite="lax"` e `path="/api/auth"`. O limiter é desativado nos testes por padrão (fixture `limiter.enabled = False`) e reativado só no teste de rate limit.
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(auth): signup, login, refresh rotation and tenant context"`

---

### Tarefa 4: Perfil da empresa, utilidades BR e logotipo

**Arquivos:**
- Criar: `backend/app/core/br.py`, `backend/app/storage/{__init__,base,local,s3}.py`, `backend/app/modules/companies/{schemas,service,router}.py`
- Teste: `backend/tests/test_br.py`, `backend/tests/test_company.py`

**Interfaces:**
- Consome: `Tenant`, `get_tenant`, `Company`, `signup` (Tarefa 3).
- Produz:
  - `normalize_br_phone(raw: str | None) -> str | None`: devolve só dígitos com `55`; `None` se vazio; lança `InvalidPhone` caso contrário.
  - `is_valid_cpf(s) -> bool`, `is_valid_cnpj(s) -> bool`, `normalize_document(raw) -> str | None` (só dígitos; lança `InvalidDocument`).
  - `format_brl(cents: int) -> str` (`"R$ 1.234,56"`, com espaço comum) e `format_date_br(d: date) -> str`.
  - `class Storage(Protocol): put(key, data: bytes, content_type: str) -> None; get(key) -> bytes; delete(key) -> None`, `get_storage() -> Storage` e `logo_key_for(company_id: UUID, ext: str) -> str`.
  - Endpoints:
    - `GET /api/company` e `PUT /api/company` (`CompanyIn`: name, trade_name, document, phone, email, address, city, state (UF de 2 letras), default_validity_days 1..365, default_payment_terms, default_notes).
    - `POST /api/company/logo` (multipart `file`; PNG/JPEG/WebP detectado pelos *magic bytes*; ≤ 2 MB).
    - `GET /api/company/logo` e `DELETE /api/company/logo`.

- [ ] **Passo 1: Escrever os testes que falham**

```python
@pytest.mark.parametrize("raw,expected", [
    ("(11) 98765-4321", "5511987654321"), ("+55 11 98765-4321", "5511987654321"),
    ("11987654321", "5511987654321"), ("1133334444", "551133334444"), ("", None), (None, None)])
def test_normalize_br_phone(raw, expected): assert normalize_br_phone(raw) == expected

@pytest.mark.parametrize("raw", ["98765-4321", "123", "5511"])
def test_normalize_br_phone_invalid(raw):
    with pytest.raises(InvalidPhone): normalize_br_phone(raw)

def test_documents():
    assert is_valid_cpf("529.982.247-25") and not is_valid_cpf("111.111.111-11")
    assert is_valid_cnpj("11.222.333/0001-81") and not is_valid_cnpj("11.222.333/0001-80")

@pytest.mark.parametrize("cents,txt", [(0, "R$ 0,00"), (123456, "R$ 1.234,56"), (100000000, "R$ 1.000.000,00")])
def test_format_brl(cents, txt): assert format_brl(cents) == txt

def test_update_company_normalizes_phone(api): ...        # PUT phone "(11) 3333-4444" → GET phone "551133334444"
def test_company_body_cannot_set_ids(api): ...            # PUT com "company_id" ou "id" → 422
def test_logo_upload_validates_magic_bytes_and_size(api): ...  # texto com content-type image/png → 422 "invalid_logo"; 2MB+1 → 422 "logo_too_large"
def test_logo_roundtrip(api): ...                          # PNG válido → GET /api/company/logo devolve os mesmos bytes
```

- [ ] **Passo 2: Rodar e confirmar que falha**: `docker compose run --rm api pytest tests/test_br.py tests/test_company.py -v`
- [ ] **Passo 3: Implementar.** Os erros de `InvalidPhone`/`InvalidDocument` viram 422 com `code` `invalid_phone`/`invalid_document`, aplicados via validators Pydantic que chamam `br.py`. `LocalStorage` grava em `STORAGE_LOCAL_DIR`; `S3Storage` usa boto3 com `endpoint_url` (R2). Trocar o logo apaga a chave antiga.
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(company): profile, BR phone/document utils and logo storage"`

---

### Tarefa 5: Frontend — autenticação, onboarding, empresa e páginas legais

**Arquivos:**
- Criar: `src/lib/auth.tsx`, `src/features/auth/{LoginPage,SignupPage,RequireAuth}.tsx`, `src/features/company/{api.ts,CompanySettingsPage.tsx}`, `src/features/legal/{PrivacyPage,TermsPage}.tsx`
- Teste: `src/features/auth/auth.test.tsx`, `src/features/company/company.test.tsx`

**Interfaces:**
- Consome: `apiFetch`, `setAccessToken`, `ApiError`, `AppShell` (Tarefa 2) e os endpoints `/api/auth/*` e `/api/company*` (Tarefas 3–4).
- Produz:
  - `AuthProvider` e `useAuth(): { status: "loading"|"authenticated"|"anonymous"; user; company; plan; login(email, password); signup(data); logout() }`. Ao montar, chama `/api/auth/refresh` para restaurar a sessão.
  - Rotas: `/entrar`, `/cadastro`, `/privacidade`, `/termos`, `/app/*` (protegidas por `RequireAuth`) e `/app/empresa`.

- [ ] **Passo 1: Escrever os testes que falham** (com msw):
  - Signup com senha `"123"` mostra `A senha deve ter pelo menos 8 caracteres` e não chama a API.
  - Signup sem marcar `Li e aceito os Termos de Uso e a Política de Privacidade` bloqueia o envio.
  - Login com 401 `invalid_credentials` mostra `E-mail ou senha incorretos`.
  - `RequireAuth` redireciona para `/entrar` quando o refresh responde 401.
  - Depois do signup, navega para `/app/empresa?onboarding=1` e mostra `Complete os dados da sua empresa para aparecerem nos seus orçamentos.`
  - Na página da empresa, o upload do logo mostra o preview, e o erro `invalid_logo` vira `Envie uma imagem PNG, JPG ou WebP de até 2 MB.`
- [ ] **Passo 2: Rodar e confirmar que falha**: `npm test -- --run src/features/auth src/features/company`
- [ ] **Passo 3: Implementar** com react-hook-form + zod. As páginas legais mostram o aviso `Rascunho — revisar com assessoria jurídica antes do lançamento` e cobrem quais dados são coletados, a finalidade, a retenção, a exclusão de conta e o contato (LGPD).
- [ ] **Passo 4: Rodar e confirmar que passa.** Verificação manual: `docker compose up` + `npm run dev`; cadastrar, recarregar a página e confirmar que a sessão continua ativa.
- [ ] **Passo 5: Commit**: `git commit -m "feat(web): auth flow, onboarding, company settings and legal pages"`

---

### Tarefa 6: Clientes (API)

**Arquivos:**
- Criar: `backend/app/modules/clients/{models,schemas,service,router}.py`, `backend/alembic/versions/0002_clients.py`
- Teste: `backend/tests/test_clients.py`. Adicionar `create_client(api, auth, **overrides) -> dict` em `factories.py`.

**Interfaces:**
- Consome: `Tenant`, `get_tenant`, `normalize_br_phone`, `normalize_document`.
- Produz:
  - `Client(company_id, name, document, phone, email, address, internal_notes, archived_at)`, com índice `(company_id, name)`.
  - `get_client(db, tenant, client_id) -> Client` (404 `client_not_found` se pertencer a outra empresa).
  - Endpoints: `GET /api/clients?q=&include_archived=false&page=1&page_size=20` → `{items, total}`, `POST /api/clients`, `GET/PUT /api/clients/{id}`, `POST /api/clients/{id}/archive` e `/unarchive`.
  - `ClientIn`: name (1..200, obrigatório) e os demais campos opcionais (email validado).

- [ ] **Passo 1: Escrever os testes que falham**: `test_create_client_normalizes_phone` (`"(11) 98765-4321"` → `"5511987654321"`), `test_invalid_document_422` (code `invalid_document`), `test_list_excludes_archived_by_default`, `test_search_case_insensitive_by_name_and_phone` (`"silva"` encontra `"João Silva"`), `test_list_only_own_company`, `test_other_company_404_on_get_put_archive`.
- [ ] **Passo 2: Rodar e confirmar que falha**: `docker compose run --rm api pytest tests/test_clients.py -v`
- [ ] **Passo 3: Implementar** a busca com `ILIKE` sobre `name`, `phone` e `document`, e a ordenação por `name`.
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(clients): tenant-scoped CRUD, search and archive"`

---

### Tarefa 7: Clientes (UI)

**Arquivos:**
- Criar: `src/features/clients/{api.ts,ClientsPage.tsx,ClientFormDialog.tsx}`
- Teste: `src/features/clients/clients.test.tsx`

**Interfaces:**
- Consome: `/api/clients*` (Tarefa 6) e `apiFetch`.
- Produz: os hooks `useClients(params)`, `useCreateClient()`, `useUpdateClient()` e `useArchiveClient()`, além de `ClientFormDialog({ client?, onSaved(client) })`, que a Tarefa 13 reutiliza para o cadastro inline.

- [ ] **Passo 1: Escrever os testes que falham**:
  - Estado vazio mostra `Nenhum cliente cadastrado` e o botão `Cadastrar cliente`.
  - Um 422 `invalid_document` aparece no campo como `CPF/CNPJ inválido`.
  - Arquivar pede confirmação e remove o item da lista.
  - O toggle `Mostrar arquivados` exibe os arquivados.
- [ ] **Passo 2: Rodar e confirmar que falha.**
- [ ] **Passo 3: Implementar.** Tabela a partir de `md` e cards no mobile, busca com debounce de 300 ms, máscara visual de telefone.
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(web): clients list, search, form and archive"`

---

### Tarefa 8: Catálogo (API) e modelos por segmento

**Arquivos:**
- Criar: `backend/app/modules/catalog/{models,schemas,service,router,templates}.py`, `backend/alembic/versions/0003_catalog.py`
- Teste: `backend/tests/test_catalog.py`. Adicionar `create_catalog_item(...)` em `factories.py`.

**Interfaces:**
- Produz:
  - `CatalogItem(company_id, name, kind: "product"|"service", unit, default_price_cents ≥ 0, active=True)`.
  - `get_catalog_item(db, tenant, item_id) -> CatalogItem` (404 `catalog_item_not_found`).
  - Endpoints: `GET /api/catalog-items?q=&kind=&active=`, `POST`, `PUT /api/catalog-items/{id}` e `POST /api/catalog-items/import-template {segment: "ar_condicionado"|"eletrica"}` → `{created: int}`.
  - `TEMPLATES: dict[str, list[tuple[name, kind, unit]]]`, todos com preço 0 (o usuário define os próprios preços; o sistema não inventa preços de mercado):
    - `ar_condicionado`: Instalação de ar-condicionado split até 12.000 BTUs (service, un) · Instalação de ar-condicionado split 18.000 a 24.000 BTUs (service, un) · Manutenção preventiva / limpeza completa (service, un) · Carga de gás refrigerante (service, un) · Visita técnica (service, un) · Tubulação de cobre – par (product, m) · Suporte para condensadora (product, un) · Mangueira de dreno (product, m)
    - `eletrica`: Mão de obra elétrica (service, h) · Instalação de tomada (service, un) · Instalação de interruptor (service, un) · Instalação de chuveiro elétrico (service, un) · Troca de disjuntor (service, un) · Montagem de quadro de distribuição (service, un) · Cabo flexível 2,5 mm² (product, m) · Disjuntor (product, un)

- [ ] **Passo 1: Escrever os testes que falham**: `test_create_and_filter_active`, `test_negative_price_422`, `test_import_template_creates_8_and_is_idempotent` (1ª chamada `{"created": 8}`, 2ª `{"created": 0}`, comparando nomes sem diferenciar maiúsculas), `test_unknown_segment_422`, `test_other_company_404`.
- [ ] **Passo 2: Rodar e confirmar que falha**: `docker compose run --rm api pytest tests/test_catalog.py -v`
- [ ] **Passo 3: Implementar.**
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(catalog): items CRUD and segment templates"`

---

### Tarefa 9: Catálogo (UI)

**Arquivos:**
- Criar: `src/features/catalog/{api.ts,CatalogPage.tsx,CatalogItemDialog.tsx}`
- Teste: `src/features/catalog/catalog.test.tsx`

**Interfaces:**
- Produz: `useCatalogItems(params)` e `CatalogItemDialog`. A Tarefa 13 consome `useCatalogItems({ active: true })`.

- [ ] **Passo 1: Escrever os testes que falham**:
  - Digitar o preço `"1.250,00"` envia `default_price_cents: 125000`.
  - O estado vazio mostra `Importar modelo: Ar-condicionado` e `Importar modelo: Elétrica`, e importar mostra o toast `8 itens adicionados`.
  - O filtro por tipo funciona.
  - Desativar um item o tira da lista "Ativos".
- [ ] **Passo 2: Rodar e confirmar que falha.**
- [ ] **Passo 3: Implementar.** Unidades sugeridas no select (com opção de texto livre): `un, m, m², h, kit, serviço`.
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(web): catalog management with segment templates"`

---

### Tarefa 10: Cálculo de orçamento (módulo puro, backend + espelho no frontend)

**Arquivos:**
- Criar: `backend/app/modules/quotes/calc.py`, `frontend/src/features/quotes/calc.ts`
- Teste: `backend/tests/test_quote_calc.py`, `frontend/src/features/quotes/calc.test.ts`

**Interfaces:**
- Produz (Python):
  - `LineInput(quantity: Decimal, unit_price_cents: int)` (frozen).
  - `Discount` (frozen) com os construtores `Discount.none()`, `Discount.amount(cents: int)` e `Discount.percent(p: Decimal)`; campos `type`, `amount_cents`, `percent`.
  - `Totals(line_totals_cents: list[int], subtotal_cents, discount_cents, total_cents)`.
  - `compute_totals(lines: Sequence[LineInput], discount: Discount) -> Totals`, que lança `InvalidLine` e `InvalidDiscount`.
  - Limites: `0 < quantity ≤ 1_000_000` com no máximo 3 casas decimais; `0 ≤ unit_price_cents ≤ 1_000_000_000`; no máximo 100 linhas; percentual entre 0 e 100 com até 2 casas; desconto em valor entre 0 e o subtotal.
- Produz (TS): `computeTotals(lines: {quantity: number; unitPriceCents: number}[], discount: {type:"none"} | {type:"amount"; amountCents:number} | {type:"percent"; percent:number}): {lineTotalsCents:number[]; subtotalCents:number; discountCents:number; totalCents:number; error?: "invalid_line"|"invalid_discount"}`. Usa **BigInt**: `qtyMilli = round(q*1000)` e `line = roundHalfUp(qtyMilli*price/1000)`.
- Algoritmo (idêntico nos dois lados): `line = (q × price).quantize(1, ROUND_HALF_UP)`; `subtotal = Σ line`; `discount = amount` ou `(subtotal × p / 100).quantize(1, ROUND_HALF_UP)`; `total = subtotal − discount`.

- [ ] **Passo 1: Escrever os testes que falham** (os mesmos casos em Python e em TS)

```python
def test_line_total_rounds_half_up():
    t = compute_totals([LineInput(Decimal("2.5"), 3733)], Discount.none())
    assert (t.line_totals_cents, t.subtotal_cents, t.total_cents) == ([9333], 9333, 9333)

def test_percent_discount():
    t = compute_totals([LineInput(Decimal("1"), 10000), LineInput(Decimal("2"), 2550)], Discount.percent(Decimal("10")))
    assert (t.subtotal_cents, t.discount_cents, t.total_cents) == (15100, 1510, 13590)

def test_percent_discount_rounds_half_up():   # 999 × 12,5% = 124,875 → 125
    assert compute_totals([LineInput(Decimal("1"), 999)], Discount.percent(Decimal("12.5"))).total_cents == 874

def test_amount_discount():
    assert compute_totals([LineInput(Decimal("1"), 15100)], Discount.amount(2000)).total_cents == 13100

@pytest.mark.parametrize("d", [Discount.amount(15101), Discount.amount(-1),
                               Discount.percent(Decimal("100.01")), Discount.percent(Decimal("-1"))])
def test_invalid_discount(d):
    with pytest.raises(InvalidDiscount): compute_totals([LineInput(Decimal("1"), 15100)], d)

@pytest.mark.parametrize("q", [Decimal("0"), Decimal("-1"), Decimal("1.2345")])
def test_invalid_quantity(q):
    with pytest.raises(InvalidLine): compute_totals([LineInput(q, 100)], Discount.none())

def test_no_lines_is_zero():
    assert compute_totals([], Discount.none()).total_cents == 0
```

- [ ] **Passo 2: Rodar e confirmar que falha**: `docker compose run --rm api pytest tests/test_quote_calc.py -v` e `npm test -- --run src/features/quotes/calc`
- [ ] **Passo 3: Implementar** os dois módulos.
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(quotes): pure totals calculation with half-up rounding (py + ts)"`

---

### Tarefa 11: Orçamentos (API): CRUD, numeração, snapshot e cota do plano

**Arquivos:**
- Criar: `backend/app/modules/quotes/{models,schemas,service,router,numbering,events}.py`, `backend/app/core/clock.py`, `backend/app/cli.py`, `backend/alembic/versions/0004_quotes.py`
- Teste: `backend/tests/test_quotes.py`. Adicionar `create_quote(api, auth, client_id, items=None, **overrides) -> dict` e `set_plan(db, company_id, plan)` em `factories.py`.

**Interfaces:**
- Consome: `compute_totals`, `LineInput`, `Discount` (Tarefa 10), `get_client` (Tarefa 6), `get_catalog_item` (Tarefa 8), `Subscription` e `Company.next_quote_number` (Tarefa 3).
- Produz:
  - `Quote`:
    - Identificação: `company_id`, `number` (UNIQUE `(company_id, number)`), `client_id`, `created_by`.
    - Snapshot do cliente: `client_name`, `client_phone`, `client_document`, `client_email`, `client_address`.
    - Estado e conteúdo: `status="draft"`, `version=1`, `valid_until`, `payment_terms`, `execution_time`, `notes`, `terms`, `internal_notes`.
    - Desconto e totais: `discount_type`, `discount_amount_cents`, `discount_percent`, `subtotal_cents`, `discount_cents`, `total_cents`.
    - Link e datas: `public_token` (unique, nullable), `sent_at`, `responded_at`.
  - `QuoteItem(quote_id, position, catalog_item_id (FK SET NULL), kind, description, unit, quantity NUMERIC(12,3), unit_price_cents, line_total_cents)`.
  - `QuoteEvent(company_id, quote_id, type, actor: "user"|"client"|"system", user_id, version, payload JSONB, created_at)` e `record_event(db, quote, type: EventType, actor, *, user_id=None, payload=None) -> QuoteEvent`. `EventType(StrEnum)`: `created, updated, sent, viewed, revised, approved, rejected, changes_requested, expired, canceled, link_regenerated`.
  - `now_utc() -> datetime`, `today_local() -> date` e `month_bounds_utc(d: date) -> tuple[datetime, datetime]`, todos usando `APP_TIMEZONE`.
  - `allocate_quote_number(db, company_id) -> int`: `SELECT ... FOR UPDATE` em `companies`, devolve `next_quote_number` e incrementa. Helper `display_number(n) -> f"ORC-{n:04d}"`.
  - `check_quota(db, tenant) -> None`: no plano free, se já houver ≥ `FREE_PLAN_MONTHLY_QUOTES` orçamentos criados no mês local → `AppError(403, "plan_limit_reached", "Você atingiu o limite de 5 orçamentos do plano Gratuito neste mês.")`.
  - `get_quote(db, tenant, quote_id) -> Quote` (404 `quote_not_found`).
  - Endpoints:
    - `POST /api/quotes` com `QuoteIn{client_id, valid_until?, payment_terms?, execution_time?, notes?, terms?, internal_notes?, discount: {type:"none"} | {type:"amount", amount_cents} | {type:"percent", percent}, items: [{catalog_item_id?, kind, description(1..500), unit, quantity, unit_price_cents}]}` → 201 `QuoteOut`.
    - `GET /api/quotes?status=&q=&page=` → `{items: QuoteSummary[], total}`.
    - `GET /api/quotes/{id}`, `PUT /api/quotes/{id}` (nesta tarefa, só em `draft`; nos demais status → 409 `quote_locked`) e `DELETE /api/quotes/{id}` (só `draft`, 204).
  - `QuoteSummary{id, number, display_number, client_name, status, total_cents, valid_until, created_at}`. `QuoteOut` acrescenta todos os campos, `items` com `line_total_cents`, `version` e `public_url` (null até o envio).
  - Defaults: `valid_until = today_local() + company.default_validity_days`; `payment_terms`/`notes` vêm dos defaults da empresa quando omitidos. `valid_until < today_local()` → 422 `invalid_valid_until` (vale também para revisões na Tarefa 12, inclusive ao reabrir um orçamento `expired`).
  - CLI: `python -m app.cli set-plan <company_id> <free|professional|business>`.

- [ ] **Passo 1: Escrever os testes que falham**
  - `test_create_quote_totals_computed_on_backend`: itens `(2.5, 3733)` e `(1, 10000)`, desconto de 10% → `subtotal 19333`, `discount 1933`, `total 17400`.
  - `test_numbers_sequential_per_company`: A cria 2 (`ORC-0001`, `ORC-0002`); B cria 1 (`ORC-0001`).
  - `test_client_of_other_company_422`: code `invalid_client`.
  - `test_catalog_item_of_other_company_422`: code `invalid_catalog_item`.
  - `test_editing_quote_item_does_not_touch_catalog`: catálogo a 5000, item no orçamento a 4500 → o catálogo continua 5000.
  - `test_client_snapshot_survives_client_rename`.
  - `test_archived_client_quote_still_readable`.
  - `test_free_plan_limit`: 5 criados; o 6º dá 403 `plan_limit_reached`; com `freeze_time` no mês seguinte → 201; com o plano `professional`, 7 criados → todos 201.
  - `test_put_non_draft_409_quote_locked`: força `status="sent"` no db.
  - `test_delete_only_draft`.
  - `test_other_company_404_get_put_delete`.
  - `test_list_filter_by_status_and_search_by_client_name`.
  - `test_create_records_created_event`.
- [ ] **Passo 2: Rodar e confirmar que falha**: `docker compose run --rm api pytest tests/test_quotes.py -v`
- [ ] **Passo 3: Implementar.** O PUT substitui todos os itens (apaga e recria) e recalcula os totais. O snapshot do cliente é refeito a cada save. Erros de `InvalidLine`/`InvalidDiscount` viram 422 com `code` `invalid_line`/`invalid_discount`.
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(quotes): CRUD, per-company numbering, client snapshot and free-plan quota"`

---

### Tarefa 12: Ciclo de vida: status, versões, envio, cancelamento e expiração

**Arquivos:**
- Criar: `backend/app/modules/quotes/{status,lifecycle}.py`
- Modificar: `backend/app/modules/quotes/{service,router}.py` (o PUT passa a revisar orçamentos não-draft)
- Teste: `backend/tests/test_quote_status.py`, `backend/tests/test_quote_lifecycle.py`

**Interfaces:**
- Consome: `Quote`, `record_event`, `EventType`, `today_local`, `get_quote` (Tarefa 11).
- Produz:
  - `QuoteStatus(StrEnum)` com os 8 status, `ALLOWED: dict[QuoteStatus, frozenset[QuoteStatus]]` e `transition(current, target) -> QuoteStatus` (lança `InvalidTransition` → 409 `invalid_transition`).
  - Tabela `ALLOWED` exata:
    - `draft` → `sent`, `canceled`
    - `sent` → `sent`, `viewed`, `approved`, `rejected`, `changes_requested`, `expired`, `canceled`
    - `viewed` → `sent`, `approved`, `rejected`, `changes_requested`, `expired`, `canceled`
    - `changes_requested` → `sent`, `approved`, `rejected`, `canceled`
    - `expired` → `sent`, `canceled`
    - `approved`, `rejected`, `canceled` → nenhum (terminais)
  - `send_quote(db, tenant, quote_id) -> Quote` (gera token se não houver, `sent_at`, evento `sent`), `public_url(quote) -> str | None`, `cancel_quote(...)`, `regenerate_public_link(...)` (evento `link_regenerated`), `get_quote_by_token(db, token) -> Quote | None`, `expire_overdue(db, company_id: UUID, today: date) -> int` (`sent`/`viewed` com `valid_until < today` → `expired` + evento `expired` com ator `system`).
  - A revisão no PUT de um orçamento em `sent`/`viewed`/`changes_requested`/`expired` faz:
    1. Evento `revised` com `payload={"previous": <QuoteOut serializado>}`.
    2. `version += 1`.
    3. `transition(..., sent)`.
  - Endpoints: `POST /api/quotes/{id}/send` → `{quote: QuoteOut, public_url}`, `POST /api/quotes/{id}/cancel`, `POST /api/quotes/{id}/regenerate-link` → `{public_url}` e `GET /api/quotes/{id}/events` → `[{type, actor, version, payload, created_at}]` em ordem cronológica.

- [ ] **Passo 1: Escrever os testes que falham**
  - `test_transition_table` (parametrizado): `draft→approved` lança erro, `approved→sent` lança erro, `viewed→approved` é ok, `expired→sent` é ok, `canceled→sent` lança erro.
  - `test_send_generates_unguessable_token`: `len(token) >= 43`; `public_url == f"{PUBLIC_APP_URL}/o/{token}"`; status `sent`; evento `sent`.
  - `test_resend_keeps_same_token`.
  - `test_edit_draft_keeps_version_1`.
  - `test_edit_sent_quote_creates_version_2`: muda o preço → `version == 2`, `status == "sent"`, evento `revised` com `payload["previous"]["total_cents"]` igual ao total antigo.
  - `test_edit_approved_409`.
  - `test_regenerate_link_invalidates_old_token`: `get_quote_by_token(db, old) is None`.
  - `test_expire_overdue_only_sent_or_viewed`: draft vencido continua draft; sent vencido vira expired com evento `system`.
  - `test_cancel_then_send_409`.
  - `test_events_chronological`: `created`, `sent`, `revised`.
- [ ] **Passo 2: Rodar e confirmar que falha**: `docker compose run --rm api pytest tests/test_quote_status.py tests/test_quote_lifecycle.py -v`
- [ ] **Passo 3: Implementar.** Toda mudança de status passa por `transition()` e registra um evento na mesma transação.
- [ ] **Passo 4: Rodar e confirmar que passa.** Rodar também `pytest -q` (a suíte inteira, para pegar regressões da Tarefa 11).
- [ ] **Passo 5: Commit**: `git commit -m "feat(quotes): status machine, versioned revisions, send/cancel/expire and events"`

---

### Tarefa 13: Telas de orçamento (lista, editor, detalhe)

**Arquivos:**
- Criar: `src/features/quotes/{api.ts,QuotesListPage.tsx,QuoteEditorPage.tsx,ItemsEditor.tsx,ClientPicker.tsx,CatalogPicker.tsx,DiscountField.tsx,StatusBadge.tsx,QuoteDetailPage.tsx}`
- Teste: `src/features/quotes/quotes.test.tsx`

**Interfaces:**
- Consome: `computeTotals` (Tarefa 10), `useClients` e `ClientFormDialog` (Tarefa 7), `useCatalogItems` (Tarefa 9) e os endpoints das Tarefas 11–12.
- Produz:
  - Rotas: `/app/orcamentos`, `/app/orcamentos/novo`, `/app/orcamentos/:id` e `/app/orcamentos/:id/editar`.
  - `STATUS_LABELS`: `draft` "Rascunho", `sent` "Enviado", `viewed` "Visualizado", `approved` "Aprovado", `rejected` "Recusado", `changes_requested` "Alteração solicitada", `expired` "Expirado", `canceled` "Cancelado".
  - `QuoteDetailPage` tem uma área de ações (`Enviar`, `Editar`, `Cancelar`, `Baixar PDF`) que as Tarefas 14, 16 e 17 preenchem.

- [ ] **Passo 1: Escrever os testes que falham**
  - No `ItemsEditor`, ao adicionar "Instalação de tomada" do catálogo (preço 0), digitar o preço `350,00` e a quantidade `2` mostra a linha `R$ 700,00` e o total `R$ 700,00`.
  - A quantidade `2,5` com o preço `37,33` mostra `R$ 93,33`.
  - O desconto percentual `120` mostra `Desconto não pode passar de 100%` e desativa `Salvar`.
  - Um 403 `plan_limit_reached` mostra a mensagem da API e o CTA `Conhecer planos`.
  - Editar um orçamento `sent` na versão 1 mostra `Este orçamento já foi enviado. Salvar criará a versão 2 e o cliente verá a versão atualizada no mesmo link.`
  - Orçamentos `approved` não exibem o botão `Editar`.
- [ ] **Passo 2: Rodar e confirmar que falha.**
- [ ] **Passo 3: Implementar.** O editor usa react-hook-form com `useFieldArray`. No mobile, os itens aparecem como cards empilhados. O `ClientPicker` tem busca e a opção `+ Novo cliente`, que abre `ClientFormDialog`. Depois de salvar, exibe os totais devolvidos pelo backend.
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(web): quotes list, editor with live totals and detail page"`

---

### Tarefa 14: Geração de PDF

**Arquivos:**
- Criar: `backend/app/modules/pdf/{renderer.py,templates/quote.html,templates/pdf.css}`
- Modificar: `backend/app/modules/quotes/router.py` (adicionar `GET /api/quotes/{id}/pdf`) e `src/features/quotes/QuoteDetailPage.tsx` (botão `Baixar PDF`)
- Teste: `backend/tests/test_pdf.py`

**Interfaces:**
- Consome: `Quote`, `Company`, `Subscription.plan`, `get_storage`, `format_brl`, `format_date_br`, `display_number`.
- Produz: `render_quote_html(quote, company, plan: str, logo_data_uri: str | None) -> str` e `render_quote_pdf(quote, company, plan, logo: bytes | None) -> bytes`. A resposta usa `application/pdf` com `Content-Disposition: inline; filename="orcamento-ORC-0001.pdf"`. A Tarefa 15 reutiliza `render_quote_pdf`.
- Conteúdo (spec 6.5), em A4 com fonte DejaVu Sans:
  - Cabeçalho: logo (só para planos pagos), dados e contatos da empresa.
  - Cliente.
  - Número, data, versão e validade.
  - Tabela de itens (descrição, unidade, quantidade, unitário, total).
  - Subtotal, desconto e total.
  - Condições de pagamento, prazo de execução, observações e termos.
  - Rodapé com contatos e, no plano free, o texto `Gerado com Orcamentos Brasil`.

- [ ] **Passo 1: Escrever os testes que falham**
  - `test_html_has_required_fields`: contém o nome da empresa, o nome do cliente, `ORC-0001`, `R$ 1.234,56`, a validade `02/12/2026` e as condições de pagamento; **não** contém o texto de `internal_notes`.
  - `test_free_plan_watermark_and_no_logo`, mais o caso `professional`: tem `<img` e não tem `Gerado com Orcamentos Brasil`.
  - `test_user_text_is_escaped`: `notes="<script>x</script>"` aparece como `&lt;script&gt;`.
  - `test_pdf_endpoint`: 200, `content-type` `application/pdf`, corpo começando com `b"%PDF"`.
  - `test_pdf_other_company_404`.
- [ ] **Passo 2: Rodar e confirmar que falha**: `docker compose run --rm api pytest tests/test_pdf.py -v`
- [ ] **Passo 3: Implementar.** Jinja2 `Environment(autoescape=True)`. O logo entra como data URI. As quantidades são exibidas sem zeros à direita (`2,5`, não `2,500`).
- [ ] **Passo 4: Rodar e confirmar que passa.** Verificação manual: abrir o PDF gerado no celular (pinch-zoom legível, tabela sem corte).
- [ ] **Passo 5: Commit**: `git commit -m "feat(pdf): professional quote PDF with plan-based branding"`

---

### Tarefa 15: Página pública (API) e respostas do cliente

**Arquivos:**
- Criar: `backend/app/modules/public/{schemas,service,router}.py`
- Teste: `backend/tests/test_public.py`

**Interfaces:**
- Consome: `get_quote_by_token`, `transition`, `record_event`, `expire_overdue`, `today_local` (Tarefa 12), `render_quote_pdf` (Tarefa 14) e `limiter` (Tarefa 3).
- Produz:
  - `GET /api/public/quotes/{token}` (60/min/IP) → `PublicQuoteOut`. Campos exatos: `display_number, version, status, can_respond, created_at, valid_until, responded_at, company{name, trade_name, phone, email, address, city, state, has_logo}, client_name, items[{description, unit, quantity, unit_price_cents, line_total_cents}], subtotal_cents, discount_cents, total_cents, payment_terms, execution_time, notes, terms`.
  - `can_respond = status in {sent, viewed, changes_requested}`.
  - Efeito do GET: expira se vencido; se `sent`, passa para `viewed` e registra o evento `viewed` (no máximo um por versão).
  - `POST /api/public/quotes/{token}/approve {version, signer_name (2..120)}`, `/reject {version, reason? (≤2000)}` e `/request-changes {version, message (1..2000)}`, todos com limite de 10/min/IP. O payload do evento guarda `signer_name`/`reason`/`message`, `ip` e `user_agent`; `responded_at = now_utc()`.
  - `GET /api/public/quotes/{token}/pdf` e `GET /api/public/quotes/{token}/logo` (404 se o plano for free ou não houver logo).
  - Erros:
    - Token desconhecido → 404 `quote_not_found`.
    - Versão divergente → 409 `version_mismatch`.
    - Status que não aceita resposta → 409 `cannot_respond`.
    - Vencido → 410 `quote_expired`.

- [ ] **Passo 1: Escrever os testes que falham**
  - `test_public_payload_has_exact_keys`: `set(body) == EXPECTED_KEYS`; sem `internal_notes`, `company_id`, `client_id`, `client_phone`, `client_document` nem `public_token`.
  - `test_first_view_marks_viewed_once`: dois GETs → status `viewed` e exatamente 1 evento `viewed`.
  - `test_unknown_token_404`.
  - `test_approve_records_signer_ip_and_time`.
  - `test_approve_twice_409_cannot_respond`.
  - `test_stale_version_409_version_mismatch`: o cliente vê a v1, o dono edita (v2) e o cliente aprova com `version=1`.
  - `test_request_changes_requires_message_422`, e o caso válido → `changes_requested`, com a mensagem aparecendo em `GET /api/quotes/{id}/events`.
  - `test_reject_with_optional_reason`.
  - `test_expired_quote_410`: `freeze_time` depois de `valid_until`.
  - `test_old_token_404_after_regenerate`.
  - `test_canceled_quote_visible_but_cannot_respond`.
  - `test_public_post_rate_limited_429`: 11º POST em 1 min.
  - `test_public_pdf_ok`.
- [ ] **Passo 2: Rodar e confirmar que falha**: `docker compose run --rm api pytest tests/test_public.py -v`
- [ ] **Passo 3: Implementar.** O IP vem de `X-Forwarded-For`, aceito apenas quando a requisição chega do proxy confiável (o Caddy na rede do Docker); caso contrário, usa `request.client.host`.
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(public): public quote view, approve/reject/request-changes with versions"`

---

### Tarefa 16: Envio por WhatsApp e página pública (UI)

**Arquivos:**
- Criar: `src/lib/whatsapp.ts`, `src/features/quotes/ShareDialog.tsx`, `src/features/public/{api.ts,PublicQuotePage.tsx,ApproveDialog.tsx,RejectDialog.tsx,RequestChangesDialog.tsx}`
- Modificar: `src/features/quotes/QuoteDetailPage.tsx` (botão `Enviar` abre o `ShareDialog`) e `src/App.tsx` (rota pública `/o/:token`, fora do `RequireAuth`)
- Teste: `src/lib/whatsapp.test.ts`, `src/features/public/public.test.tsx`

**Interfaces:**
- Consome: `POST /api/quotes/{id}/send` (Tarefa 12) e `/api/public/*` (Tarefa 15).
- Produz:
  - `shareMessage(publicUrl: string): string`, que devolve exatamente `Olá! Preparei seu orçamento. Você pode visualizar os detalhes e responder por este link: ${publicUrl}`.
  - `buildWhatsAppLink(phone: string | null, message: string): string`: com telefone → `https://wa.me/${phone}?text=${encodeURIComponent(message)}`; sem telefone → `https://wa.me/?text=...`.
  - O `ShareDialog` oferece `Enviar por WhatsApp` (chama `send` e abre o link em nova aba usando o telefone do snapshot do cliente), `Copiar link` e `Baixar PDF`.

- [ ] **Passo 1: Escrever os testes que falham**
  - `buildWhatsAppLink("5511987654321", shareMessage("https://app/o/abc"))` começa com `https://wa.me/5511987654321?text=` e, decodificado, é igual à mensagem exata.
  - Com telefone `null`, gera `https://wa.me/?text=`.
  - Na página pública com `can_respond: false`, os botões `Aprovar`, `Solicitar alteração` e `Recusar` não aparecem, e o banner mostra o status.
  - O `ApproveDialog` exige nome com pelo menos 2 caracteres (`Informe seu nome`).
  - Um 409 `version_mismatch` mostra `Este orçamento foi atualizado. Recarregue a página para ver a versão mais recente.` com o botão `Recarregar`.
  - Um 410 mostra `Este orçamento expirou. Fale com o prestador pelo WhatsApp.`
  - O botão `Falar no WhatsApp` usa `buildWhatsAppLink(company.phone, "Olá! Tenho uma dúvida sobre o orçamento ORC-0001.")`.
- [ ] **Passo 2: Rodar e confirmar que falha.**
- [ ] **Passo 3: Implementar.** A página é mobile-first, com o total em destaque e os botões fixos no rodapé no mobile. `document.title` = `Orçamento ORC-0001 — {empresa}`. A meta `robots` é `noindex`.
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(web): WhatsApp share and public quote page with client responses"`

---

### Tarefa 17: Painel e histórico

**Arquivos:**
- Criar: `backend/app/modules/dashboard/{schemas,service,router}.py`, `src/features/dashboard/DashboardPage.tsx`, `src/features/quotes/QuoteTimeline.tsx`
- Modificar: `src/features/quotes/QuoteDetailPage.tsx` (incluir o `QuoteTimeline`)
- Teste: `backend/tests/test_dashboard.py`, `src/features/dashboard/dashboard.test.tsx`

**Interfaces:**
- Consome: `expire_overdue`, `today_local`, `month_bounds_utc`, `GET /api/quotes/{id}/events`.
- Produz: `GET /api/dashboard` → `DashboardOut`:

```
{
  counts: {total, draft, awaiting_response, approved, rejected, changes_requested, expired, canceled},
  negotiating_total_cents,
  approved_total_cents,
  quotes_this_month,
  plan,
  monthly_quota,        // null quando ilimitado
  recent: QuoteSummary[10]
}
```

  - `awaiting_response` = `sent` + `viewed`.
  - `negotiating_total_cents` = soma de `total_cents` nos status `sent`, `viewed` e `changes_requested`.
  - `approved_total_cents` = soma de `total_cents` dos `approved`.
  - `total` exclui `canceled`.

- [ ] **Passo 1: Escrever os testes que falham**
  - Backend, `test_dashboard_counts_and_totals`: monta 1 draft, 2 sent (10000 e 5000), 1 viewed (2500), 1 approved (123456), 1 rejected e 1 canceled. Espera `awaiting_response == 3`, `negotiating_total_cents == 17500`, `approved_total_cents == 123456` e `total == 6`.
  - Backend, `test_dashboard_isolated`: a empresa B vê tudo zerado.
  - Backend, `test_dashboard_expires_before_counting`.
  - Backend, `test_free_quota_fields`: `monthly_quota == 5` e `quotes_this_month` correto; no plano `professional`, `monthly_quota is None`.
  - Frontend: os cards mostram `R$ 1.234,56` em "Aprovado"; no plano free aparece `3 de 5 orçamentos usados este mês`; o CTA `Novo orçamento` está presente.
  - Frontend: o `QuoteTimeline` renderiza `Alteração solicitada` com a mensagem do cliente e `Aprovado por Maria` com data e hora em America/Sao_Paulo.
- [ ] **Passo 2: Rodar e confirmar que falha.**
- [ ] **Passo 3: Implementar** com uma única query agregada (`GROUP BY status`) filtrada por `company_id`.
- [ ] **Passo 4: Rodar e confirmar que passa.**
- [ ] **Passo 5: Commit**: `git commit -m "feat(dashboard): KPIs, quota indicator and quote timeline"`

---

### Tarefa 18: Segurança, LGPD e varredura de isolamento

**Arquivos:**
- Criar: `backend/app/core/security_headers.py`, `backend/app/modules/account/{service,router}.py`, `src/features/company/DeleteAccountSection.tsx`
- Modificar: `backend/app/main.py` (middlewares, CORS restrito a `CORS_ORIGINS`, logging que nunca registra `Authorization`, cookies, senhas ou tokens públicos)
- Teste: `backend/tests/test_tenant_isolation.py`, `backend/tests/test_security.py`, `backend/tests/test_account.py`

**Interfaces:**
- Produz:
  - `DELETE /api/account {password, confirm: "EXCLUIR"}`, só para `owner`. Apaga a empresa e, em cascata, clientes, catálogo, orçamentos, itens, eventos, assinatura, refresh tokens e o logo no storage. O usuário também é apagado se não tiver outros vínculos. Resposta 204, com o cookie limpo.
  - Na varredura de isolamento, `TENANT_ROUTES`, `PUBLIC_ROUTES` e `AUTH_ROUTES` são conjuntos de `(method, path)` declarados no teste.

- [ ] **Passo 1: Escrever os testes que falham**
  - `test_every_api_route_is_classified`: percorre `app.routes` e falha se alguma rota `/api/*` não estiver em nenhum dos três conjuntos. Isso força a classificação de rotas novas.
  - `test_tenant_routes_return_404_for_other_company` (parametrizado em `TENANT_ROUTES` com `{id}`): a empresa A cria cliente, item e orçamento; a empresa B chama cada rota com os IDs de A → 404.
  - `test_lists_never_leak_other_company`: as listagens de B não contêm nenhum ID de A.
  - `test_security_headers`: `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `Referrer-Policy: strict-origin-when-cross-origin`.
  - `test_cors_rejects_unknown_origin`.
  - `test_delete_account_removes_all_company_rows_and_logo`: em cada tabela de tenant, `count(*) WHERE company_id = A` é 0; os dados de B ficam intactos; `storage.get(logo_key)` lança erro.
  - `test_delete_account_requires_password_and_confirm`.
- [ ] **Passo 2: Rodar e confirmar que falha**: `docker compose run --rm api pytest tests/test_tenant_isolation.py tests/test_security.py tests/test_account.py -v`
- [ ] **Passo 3: Implementar.** FKs com `ON DELETE CASCADE` a partir de `companies` (migração `0005`). A UI fica em "Empresa → Zona de perigo" e pede para digitar `EXCLUIR`.
- [ ] **Passo 4: Rodar e confirmar que passa.** Suíte completa: `pytest -q` e `npm test -- --run`.
- [ ] **Passo 5: Commit**: `git commit -m "feat(security): isolation sweep, security headers, CORS and account deletion"`

---

### Tarefa 19: Deploy, backups testados e smoke test E2E

**Arquivos:**
- Criar: `docker-compose.prod.yml`, `Caddyfile`, `frontend/Dockerfile`, `backend/entrypoint.sh` (`alembic upgrade head` seguido de `uvicorn --workers 2 --proxy-headers --forwarded-allow-ips=<rede do docker>`), `scripts/backup.sh`, `scripts/restore-test.sh`, `docs/deploy.md`, `frontend/e2e/happy-path.spec.ts`, `frontend/playwright.config.ts`, `README.md` (seção de dev atualizada)
- Modificar: `.github/workflows/ci.yml` (job `e2e`, que sobe o compose e roda o Playwright)

**Interfaces:**
- Produz:
  - Caddy em `{$DOMAIN}`: HTTPS automático, `reverse_proxy /api/* api:8000`, SPA com `try_files {path} /index.html`, HSTS e `X-Robots-Tag: noindex` em `/o/*`.
  - `backup.sh`: `pg_dump -Fc`, depois envio para `s3://$BACKUP_BUCKET/db/YYYY-MM-DD.dump` via `aws s3 cp --endpoint-url $S3_ENDPOINT_URL`. Retenção de 30 dias por regra de lifecycle no R2. Cron diário às 03:00.
  - `restore-test.sh`: baixa o dump mais recente, restaura num container `postgres:16` descartável e confere `SELECT count(*) FROM companies` > 0. Retorna código ≠ 0 em falha. Cron semanal.

- [ ] **Passo 1: Escrever o teste E2E que falha** (`happy-path.spec.ts`):
  1. Cadastro.
  2. Importar o modelo de ar-condicionado.
  3. Criar um cliente com telefone `(11) 98765-4321`.
  4. Criar um orçamento com 2 itens do catálogo (preços `350,00` × 1 e `45,00` × `3,5`), e verificar o total `R$ 507,50`.
  5. Enviar e capturar o link em `Copiar link`.
  6. Abrir o link num contexto anônimo, verificar o total `R$ 507,50`, aprovar como `Maria Souza` e ver `Orçamento aprovado`.
  7. Voltar ao painel e verificar `Aprovado: 1` e `R$ 507,50`.
- [ ] **Passo 2: Rodar e confirmar que falha**: `cd frontend && npx playwright test`
- [ ] **Passo 3: Implementar** os arquivos de produção e os scripts. Em `docs/deploy.md`, documentar:
  - Preparo da VPS Ubuntu: Docker, firewall (portas 22/80/443) e usuário não-root.
  - DNS.
  - Variáveis de `.env.prod`.
  - `docker compose -f docker-compose.prod.yml up -d --build`.
  - Cron dos backups.
  - Procedimento de restauração.
  - Rollback por tag de imagem.
- [ ] **Passo 4: Verificar**
  - `npx playwright test` → passed.
  - Localmente, com `DOMAIN=localhost`: `docker compose -f docker-compose.prod.yml up -d --build` e `curl -k https://localhost/api/health` → `{"status":"ok","db":"ok"}`.
  - `./scripts/backup.sh && ./scripts/restore-test.sh` → `restore OK: companies=N`.
- [ ] **Passo 5: Commit**: `git commit -m "chore(deploy): production compose, Caddy, tested backups and E2E smoke test"`

---

## Checklist de liberação (spec §12, critérios mínimos)

| Critério | Garantido por |
|---|---|
| Uma empresa não consulta dados de outra | Tarefas 6, 8, 11, 14 e a varredura da Tarefa 18 |
| Totais e descontos calculados no backend | Tarefas 10 e 11 |
| Orçamento enviado não muda silenciosamente | Versões e `version_mismatch` (Tarefas 12 e 15) |
| Links públicos não permitem descobrir outros orçamentos | Token de 256 bits, rate limit, payload mínimo (Tarefas 12 e 15) |
| PDFs corretos e legíveis no celular | Tarefa 14 + verificação manual |
| Aprovação, recusa e alteração registradas | Tarefas 15 e 17 |
| Backups e recuperação testados | Tarefa 19 |

## Fora deste plano (versões futuras, conforme a spec §11)

Cobrança recorrente (Asaas/Mercado Pago) · multiusuário do plano Empresa (a tabela `company_users` já suporta) · WhatsApp Business API · notificações por e-mail/WhatsApp · IA para descrições · assinatura eletrônica avançada · NF-e · financeiro · app nativo · estoque · relatórios avançados · RLS no Postgres · métricas de produto (ativação = empresa que cria e compartilha o primeiro orçamento; dá para derivar de `quote_events.sent`).

## Atividades não-código em paralelo (spec §13 e §17)

Entrevistar 10 prestadores (roteiro na spec §13) · recrutar 3–5 pilotos (liberar com `python -m app.cli set-plan <id> professional`) · revisar Termos e Privacidade com assessoria jurídica · definir identidade visual.
