# project-setup

# Setup de Projeto Node.js + TypeScript com Clean Architecture e TDD

## 1. Criar a pasta e inicializar o projeto

```bash
mkdir salus
cd salus
npm init
```

Durante o `npm init`, preencha assim:

| Campo | Valor |
|---|---|
| package name | salus |
| version | 1.0.0 |
| description | Multi-tenant clinic management platform with scheduling, medical records and WhatsApp notifications |
| entry point | app.ts |
| test command | (deixar em branco) |
| license | MIT |

---

## 2. Configurar o Git

```bash
git init
```

Crie o `.gitignore` na raiz:

```
node_modules
dist
.env
```

---

## 3. Instalar e configurar o TypeScript

```bash
npm install -D typescript @types/node tsx
npx tsc --init
```

Substitua o conteúdo do `tsconfig.json` por:

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",
    "rootDir": "./src",
    "outDir": "./dist",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

---

## 4. Configurar os scripts no package.json

```json
"scripts": {
  "dev": "tsx watch src/main.ts",
  "build": "tsc",
  "start": "node dist/main.js",
  "test": "vitest",
  "test:run": "vitest run"
}
```

---

## 5. Configurar variáveis de ambiente

```bash
npm install dotenv zod
```

Crie o `.env` na raiz:

```
PORT=3000
```

Crie o `.env.example` na raiz (esse vai pro git):

```
PORT=3000
```

Crie o arquivo de configuração:

```typescript
// src/config/env.ts
import "dotenv/config"
import { z } from "zod"

const envSchema = z.object({
  PORT: z.coerce.number().default(3000),
})

export const env = envSchema.parse(process.env)
```

> `z.coerce.number()` converte a string do `.env` para número automaticamente.
> Se o valor for inválido, a aplicação não sobe e já avisa o erro.

---

## 6. Instalar o Express

```bash
npm install express
npm install -D @types/express
```

---

## 7. Estrutura de pastas (Clean Architecture)

```
salus/
├── src/
│   ├── config/
│   │   └── env.ts
│   ├── domain/          # entidades, interfaces de repositório
│   ├── application/     # use cases
│   ├── infra/
│   │   ├── http/
│   │   │   ├── server.ts
│   │   │   ├── routes/
│   │   │   └── middlewares/
│   │   └── database/    # implementações dos repositórios
│   └── main.ts
├── tests/
│   └── infra/
│       └── http/
│           └── server.test.ts
├── .env
├── .env.example
├── .gitignore
├── tsconfig.json
└── package.json
```

---

## 8. Criar o servidor

```typescript
// src/infra/http/server.ts
import express from "express"

export const app = express()

app.use(express.json())

app.get("/", (req, res) => {
  res.send("Hello, World!")
})
```

```typescript
// src/main.ts
import { app } from "./infra/http/server"
import { env } from "./config/env"

app.listen(env.PORT, () => {
  console.log(`Servidor rodando na porta ${env.PORT}`)
})
```

> **Importante:** o `app` é exportado sem o `listen`.
> O `main.ts` é o único arquivo que chama `listen` — e nunca é importado nos testes.

---

## 9. Configurar os testes com Vitest e Supertest

```bash
npm install -D vitest supertest @types/supertest
```

Crie o primeiro teste:

```typescript
// tests/infra/http/server.test.ts
import { describe, it, expect } from "vitest"
import request from "supertest"
import { app } from "../../../src/infra/http/server"

describe("Server", () => {
  it("deve retornar Hello, World! na rota /", async () => {
    const response = await request(app).get("/")

    expect(response.status).toBe(200)
    expect(response.text).toBe("Hello, World!")
  })
})
```

Rode os testes:

```bash
npm test
```

---

## 10. Primeiro commit

```bash
git add .
git commit -m "chore: initial setup"
```

---

## Resumo das dependências

| Dependência | Tipo | Função |
|---|---|---|
| express | produção | framework HTTP |
| dotenv | produção | carrega o .env |
| zod | produção | validação de dados e env vars |
| typescript | dev | compilador TypeScript |
| @types/node | dev | tipos do Node.js |
| @types/express | dev | tipos do Express |
| tsx | dev | roda .ts direto sem compilar |
| vitest | dev | framework de testes |
| supertest | dev | simula requisições HTTP nos testes |
| @types/supertest | dev | tipos do supertest |

---

## Conceitos aplicados

**Clean Architecture** — o domínio não depende de nada externo. Express, Prisma e qualquer outro detalhe de infra ficam isolados na camada `infra/`. Use cases e entidades nunca importam libs externas.

**TDD** — testes escritos antes ou junto com o código. O ciclo é: escreve o teste (vermelho) → implementa o mínimo para passar (verde) → refatora.
