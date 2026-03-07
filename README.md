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
    "rootDir": ".",
    "outDir": "./dist",
    "baseUrl": ".",
    "module": "CommonJS",
    "moduleResolution": "node",
    "target": "ES2022",
    "types": ["vitest/globals"],
    "paths": {
      "@domain/*": ["./src/domain/*"],
      "@application/*": ["./src/application/*"],
      "@infra/*": ["./src/infra/*"],
      "@config/*": ["./src/config/*"]
    },
    "sourceMap": true,
    "declaration": true,
    "declarationMap": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noPropertyAccessFromIndexSignature": true,
    "allowSyntheticDefaultImports": true,
    "esModuleInterop": true,
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src/**/*", "tests/**/*"]
}
```

> **Importante:** `rootDir: "."` permite que a pasta `tests/` fique fora do `src/` sem erro de compilação.
> `esModuleInterop` e `allowSyntheticDefaultImports` permitem usar `import express from "express"` com módulos CommonJS.
> `baseUrl` e `paths` configuram os aliases `@domain`, `@application` etc.
> Não use `"type": "module"` no `package.json` — o projeto usa CommonJS.

---

## 4. Configurar os scripts no package.json

```json
"scripts": {
  "dev": "tsx watch src/main.ts",
  "build": "tsc",
  "start": "node dist/main.js",
  "test": "vitest",
  "test:ui": "vitest --ui",
  "test:run": "vitest run",
  "lint": "eslint src tests",
  "lint:fix": "eslint src tests --fix",
  "format": "prettier --write .",
  "prepare": "husky"
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
│   ├── domain/
│   │   └── patient/
│   │       ├── patient.entity.ts
│   │       └── patient.repository.ts
│   ├── application/
│   │   └── patient/
│   │       ├── create-patient.usecase.ts
│   │       └── patient.types.ts
│   ├── infra/
│   │   ├── http/
│   │   │   ├── server.ts
│   │   │   ├── routes/
│   │   │   └── middlewares/
│   │   └── database/
│   └── main.ts
├── tests/
│   ├── application/
│   │   └── patient/
│   │       └── create-patient.usecase.test.ts
│   ├── repositories/
│   │   └── in-memory/
│   │       ├── patient.repository.ts
│   │       └── patient.repository.test.ts
│   └── infra/
│       └── http/
│           └── server.test.ts
├── .env
├── .env.example
├── .gitignore
├── tsconfig.json
├── vitest.config.ts
├── eslint.config.mjs
├── lint-staged.config.mjs
└── package.json
```

---

## 8. Criar o servidor

```typescript
// src/infra/http/server.ts
import express from "express"
import { Request, Response } from "express"

export const app = express()

app.use(express.json())

app.get("/", (req: Request, res: Response) => {
  res.send("Hello, world!")
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

Crie o `vitest.config.ts` na raiz:

```typescript
import { defineConfig } from "vitest/config"
import path from "path"

export default defineConfig({
  test: {
    globals: true,
  },
  resolve: {
    alias: {
      "@domain": path.resolve(__dirname, "./src/domain"),
      "@application": path.resolve(__dirname, "./src/application"),
      "@infra": path.resolve(__dirname, "./src/infra"),
      "@config": path.resolve(__dirname, "./src/config"),
    },
  },
})
```

> `globals: true` permite usar `describe`, `it` e `expect` sem importar em cada arquivo.
> `resolve.alias` espelha os paths do `tsconfig.json` para o Vitest resolver os aliases.

Crie o primeiro teste:

```typescript
// tests/infra/http/server.test.ts
import request from "supertest"
import { app } from "../../../src/infra/http/server"

describe("Server", () => {
  it("deve retornar Hello, world! na rota /", async () => {
    const response = await request(app).get("/")

    expect(response.status).toBe(200)
    expect(response.text).toBe("Hello, world!")
  })
})
```

Rode os testes:

```bash
npm test
```

---

## 10. Configurar ESLint e Prettier

```bash
npm install -D eslint @eslint/js typescript-eslint @vitest/eslint-plugin prettier eslint-config-prettier
```

Crie o `eslint.config.mjs` na raiz:

```javascript
import eslint from "@eslint/js"
import tseslint from "typescript-eslint"
import vitest from "@vitest/eslint-plugin"
import prettier from "eslint-config-prettier"

export default tseslint.config(
  eslint.configs.recommended,
  tseslint.configs.recommended,
  prettier,
  {
    files: ["tests/**/*.ts"],
    plugins: { vitest },
    rules: {
      ...vitest.configs.recommended.rules,
    },
  }
)
```

Crie o `.prettierrc` na raiz:

```json
{
  "semi": false,
  "singleQuote": true,
  "trailingComma": "all",
  "printWidth": 80,
  "tabWidth": 2
}
```

Crie o `.prettierignore` na raiz:

```
node_modules
dist
```

---

## 11. Configurar Husky e lint-staged

```bash
npm install -D husky lint-staged@15
npx husky init
```

> Use `lint-staged@15` — versões mais recentes podem ter incompatibilidade com Node v24.

Crie o `lint-staged.config.mjs` na raiz:

```javascript
import path from "path"

const buildEslintCommand = (filenames) =>
  `eslint --fix ${filenames
    .map((f) => path.relative(process.cwd(), f))
    .join(" ")}`

export default {
  "*.ts": [buildEslintCommand, "prettier --write"],
}
```

> Usar um arquivo separado com caminhos relativos resolve incompatibilidade do lint-staged com Node v24.

Edite o `.husky/pre-commit`:

```bash
npx lint-staged
```

---

## 12. Modelagem do domínio (Clean Architecture)

### Entidade

```typescript
// src/domain/patient/patient.entity.ts
export class Patient {
  constructor(
    public readonly id: string,
    public readonly name: string,
    public readonly phone: string,
    public readonly birthDate: Date,
  ) {}
}
```

### Interface do repositório

```typescript
// src/domain/patient/patient.repository.ts
import { Patient } from "./patient.entity"

export interface PatientRepository {
  save(patient: Patient): Promise<void>
  findById(id: string): Promise<Patient | null>
  findAll(): Promise<Patient[]>
  delete(id: string): Promise<void>
}
```

> `save` funciona como upsert — cria ou atualiza dependendo se o id já existe.

### Repositório in-memory (para testes)

```typescript
// tests/repositories/in-memory/patient.repository.ts
import { Patient } from "@domain/patient/patient.entity"
import { PatientRepository } from "@domain/patient/patient.repository"

export class InMemoryPatientRepository implements PatientRepository {
  private patients: Patient[] = []

  async save(patient: Patient): Promise<void> {
    const index = this.patients.findIndex((p) => p.id === patient.id)
    if (index >= 0) {
      this.patients[index] = patient
    } else {
      this.patients.push(patient)
    }
  }

  async findById(id: string): Promise<Patient | null> {
    return this.patients.find((p) => p.id === id) ?? null
  }

  async findAll(): Promise<Patient[]> {
    return this.patients
  }

  async delete(id: string): Promise<void> {
    this.patients = this.patients.filter((p) => p.id !== id)
  }
}
```

---

## 13. Use Cases

### Tipos de input

```typescript
// src/application/patient/patient.types.ts
export type CreatePatientInput = {
  name: string
  phone: string
  birthDate: Date
}
```

### Use case de criação

```typescript
// src/application/patient/create-patient.usecase.ts
import { Patient } from "@domain/patient/patient.entity"
import { PatientRepository } from "@domain/patient/patient.repository"
import { CreatePatientInput } from "@application/patient/patient.types"

export class CreatePatientUseCase {
  constructor(private patientRepository: PatientRepository) {}

  async execute(input: CreatePatientInput): Promise<Patient> {
    const id = crypto.randomUUID()
    const patient = new Patient(id, input.name, input.phone, input.birthDate)
    await this.patientRepository.save(patient)
    return patient
  }
}
```

> O id é gerado no use case, não na entidade.
> O repositório é recebido por injeção de dependência — nos testes passa o in-memory, em produção passa o Prisma.

---

## 14. Primeiro commit

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
| eslint | dev | análise de código e boas práticas |
| @eslint/js | dev | regras base do ESLint |
| typescript-eslint | dev | regras do ESLint para TypeScript |
| @vitest/eslint-plugin | dev | regras do ESLint para Vitest |
| eslint-config-prettier | dev | desativa regras do ESLint que conflitam com Prettier |
| prettier | dev | formatação automática de código |
| husky | dev | git hooks |
| lint-staged | dev | roda lint só nos arquivos alterados |

---

## Conceitos aplicados

**Clean Architecture** — o domínio não depende de nada externo. Express, Prisma e qualquer outro detalhe de infra ficam isolados na camada `infra/`. Use cases e entidades nunca importam libs externas.

**TDD** — testes escritos antes ou junto com o código. O ciclo é: escreve o teste (vermelho) → implementa o mínimo para passar (verde) → refatora.

**Injeção de dependência** — o use case recebe o repositório como parâmetro do construtor. Nos testes passa o in-memory, em produção passa o Prisma. O use case não sabe qual implementação está usando.

**Repositório in-memory** — implementação fake do repositório que guarda dados em array. Permite testar use cases sem banco de dados rodando.

**Multi-tenancy** — cada clínica é um tenant isolado com schema próprio no banco. O domínio não sabe disso — é resolvido na camada de infra via middleware.
