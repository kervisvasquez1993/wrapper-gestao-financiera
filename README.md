# Gestão Financeira — Wrapper

Orquestra a stack completa (Postgres + Backend NestJS + Frontend React/Vite) via Docker Compose.
Os repositórios backend e frontend estão incluídos como **submódulos do git**.

---

## Quick start

```bash
# 1. Clonar o repositório
git clone <url-do-wrapper>
cd wrapper-gestao-financiera

# 2. Baixar os submódulos (backend e frontend)
git submodule update --init --recursive

# 3. Criar o .env do wrapper (editar JWT_SECRET)
cp .env.example .env

# 4. Subir tudo
docker compose up -d --build

# 5. Migrations + seed (dentro do contêiner)
docker compose exec backend node ./node_modules/typeorm/cli.js migration:run -d dist/shared/database/data-source.js
docker compose exec backend node dist/database/seed.js
```

### URLs expostas

| Serviço           | URL                              |
| ----------------- | -------------------------------- |
| 🖥️ Frontend (SPA)  | http://localhost:5173            |
| 🔌 API REST       | http://localhost:3000/api        |
| 📖 Swagger (docs) | http://localhost:3000/api/docs   |

### Credenciais de acesso

| Email                | Senha    |
| -------------------- | -------- |
| `joao@example.com`   | `123456` |
| `maria@example.com`  | `123456` |

---

## Requisitos

- Docker e Docker Compose
- Git

---

## 1. Clonar o projeto

```bash
git clone <url-do-wrapper>
cd wrapper-product
git submodule update --init --recursive
```

> O `git submodule update --init --recursive` é **obrigatório**: baixa o código do
> backend (`Gest-o-Financeira`) e do frontend (`gestoe-finaciera-frontend`) dentro
> das pastas `backend/` e `frontend/`. Sem ele, essas pastas chegam vazias e o
> `docker compose up` falha com `failed to read dockerfile`.

Atalho para clonar e baixar os submódulos em um único comando:

```bash
git clone --recurse-submodules <url-do-wrapper>
```

---

## 2. Criar o `.env`

Copie o template como está (as variáveis já vêm prontas, só defina um `JWT_SECRET`):

```bash
cp .env.example .env
```

Edite o `.env` e coloque um valor real em `JWT_SECRET`. O resto pode ficar como está para ambiente local.

```dotenv
# ===== Postgres =====
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_DB=gestao_financeira

# ===== Backend =====
PORT=3000
DB_HOST=postgres          # nome do serviço na rede Docker, NÃO localhost
DB_PORT=5432
DB_USERNAME=postgres
DB_PASSWORD=postgres
DB_NAME=gestao_financeira
JWT_SECRET=tu_secreto_aqui
JWT_EXPIRES_IN=1d
CORS_ORIGIN=http://localhost:5173

# ===== Frontend =====
VITE_API_URL=http://localhost:3000/api

# ===== Portas do host =====
POSTGRES_PORT=5432
BACKEND_PORT=3000
FRONTEND_PORT=5173
```

> **Rede Docker:** `DB_HOST=postgres` aponta para o nome do serviço dentro da rede.
> `VITE_API_URL` e `CORS_ORIGIN` usam `localhost` porque são consumidos pelo navegador do usuário, não pelo contêiner.

---

## 3. Subir a stack

```bash
docker compose up -d --build
```

Verifique que os três serviços estão de pé (postgres em `healthy`):

```bash
docker compose ps
docker compose logs -f backend
```

---

## 4. Migrations + Seed (Docker)

### Pré-requisito: fix do glob de migrations

A imagem de produção contém apenas os arquivos `.js` compilados (não há `.ts` nem `ts-node`).
Por padrão o `data-source.ts` procura as migrations como `*.ts`, então dentro do
contêiner não encontra nada ("No migrations are pending") e as tabelas não são criadas.

Edite `backend/src/shared/database/data-source.ts` para aceitar ambas as extensões:

```ts
entities: [__dirname + '/../../**/*.entity{.ts,.js}'],
migrations: [__dirname + '/../../migrations/*{.ts,.js}'],
```

Reconstrua o backend para que ele pegue a mudança:

```bash
docker compose up -d --build backend
```

### Rodar as migrations

`typeorm` é dependência de produção, então sua CLI está disponível na imagem
(sem necessidade de `ts-node`):

```bash
docker compose exec backend node ./node_modules/typeorm/cli.js migration:run -d dist/shared/database/data-source.js
```

### Rodar o seed

```bash
docker compose exec backend node dist/database/seed.js
```

Saída esperada:

```
🧹 Limpando dados existentes...
👤 Criando usuários...
📂 Criando categorias...
💰 Criando transações...
✅ Seed executado com sucesso
👤 Usuários: 2
📂 Categorias: 10
💰 Transações: 50
```

> **Nota:** não se usa `npm run migration:run` nem `npm run seed` porque esses scripts
> dependem de `ts-node` (devDependency), que não existe na imagem de produção.
> Por isso a CLI do typeorm e o seed compilado são chamados diretamente com `node`.

> O seed **apaga** os dados existentes antes de inserir. Use apenas em ambientes de teste.

---

## 5. Credenciais (pós-seed)

Ambos os usuários têm a senha **`123456`**:

| Email                | Senha    |
| -------------------- | -------- |
| `joao@example.com`   | `123456` |
| `maria@example.com`  | `123456` |

---

## 6. URLs

| Serviço         | URL                                |
| --------------- | ---------------------------------- |
| Frontend (SPA)  | http://localhost:5173              |
| API             | http://localhost:3000/api          |
| Swagger (docs)  | http://localhost:3000/api/docs     |

### Autenticar no Swagger

1. Executar `POST /auth/login` com um dos usuários do seed.
2. Copiar o `accessToken` da resposta.
3. Clicar em **Authorize** (cadeado, canto superior direito) e colar o token.
4. As rotas protegidas ficam habilitadas para teste.

---

## Comandos úteis

```bash
# Ver logs ao vivo
docker compose logs -f

# Reconstruir apenas um serviço
docker compose up -d --build backend

# Derrubar tudo (mantém os dados do volume)
docker compose down

# Derrubar tudo e apagar o banco de dados
docker compose down -v
```