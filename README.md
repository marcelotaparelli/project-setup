# Setup de Projeto Node.js + TypeScript com Clean Architecture e TDD

---

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

> `rootDir: "."` permite que a pasta `tests/` fique fora do `src/` sem erro de compilação.
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
DATABASE_URL="postgresql://salus:salus@localhost:5432/salus?schema=public"
```

Crie o `.env.example` na raiz (esse vai pro git):

```
PORT=3000
DATABASE_URL="postgresql://user:password@localhost:5432/dbname?schema=public"
```

Crie o arquivo de configuração:

```typescript
// src/config/env.ts
import "dotenv/config"
import { z } from "zod"

const envSchema = z.object({
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string(),
})

export const env = envSchema.parse(process.env)
```

> `z.coerce.number()` converte a string do `.env` para número automaticamente.
> Se qualquer variável for inválida, a aplicação não sobe e já avisa o erro.

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
│   │       ├── patient.types.ts
│   │       ├── create-patient.usecase.ts
│   │       ├── list-patients.usecase.ts
│   │       ├── get-patient.usecase.ts
│   │       ├── update-patient.usecase.ts
│   │       └── delete-patient.usecase.ts
│   ├── infra/
│   │   ├── http/
│   │   │   ├── server.ts
│   │   │   ├── routes/
│   │   │   │   └── patient.routes.ts
│   │   │   └── controllers/
│   │   │       └── patient.controller.ts
│   │   ├── database/
│   │   │   └── prisma/
│   │   │       ├── client.ts
│   │   │       └── patient.repository.ts
│   │   └── factories/
│   │       └── patient.factory.ts
│   └── main.ts
├── tests/
│   ├── application/
│   │   └── patient/
│   │       ├── create-patient.usecase.test.ts
│   │       ├── list-patients.usecase.test.ts
│   │       ├── get-patient.usecase.test.ts
│   │       ├── update-patient.usecase.test.ts
│   │       └── delete-patient.usecase.test.ts
│   ├── repositories/
│   │   └── in-memory/
│   │       ├── patient.repository.ts
│   │       └── patient.repository.test.ts
│   └── infra/
│       └── http/
│           ├── server.test.ts
│           └── routes/
│               └── patient.routes.test.ts
├── prisma/
│   ├── schema.prisma
│   └── migrations/
├── .env
├── .env.example
├── .gitignore
├── tsconfig.json
├── vitest.config.ts
├── eslint.config.mjs
├── lint-staged.config.mjs
├── prisma.config.ts
└── package.json
```

---

## 8. Criar o servidor

```typescript
// src/infra/http/server.ts
import express from "express"
import { Request, Response } from "express"
import { patientRouter } from "./routes/patient.routes"

export const app = express()

app.use(express.json())
app.use(patientRouter)

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

> O `app` é exportado sem o `listen`.
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
import { app } from "@infra/http/server"

describe("Server", () => {
  it("deve retornar Hello, world! na rota /", async () => {
    const response = await request(app).get("/")

    expect(response.status).toBe(200)
    expect(response.text).toBe("Hello, world!")
  })
})
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

## 12. Primeiro commit

```bash
git add .
git commit -m "chore: initial setup"
```

---

## 13. Modelagem do domínio

### Entidade

```typescript
// src/domain/patient/patient.entity.ts
export class Patient {
  constructor(
    public readonly id: string,
    public readonly name: string,
    public readonly cpf: string,
    public readonly phone: string,
    public readonly birthDate: Date,
    public readonly createdAt?: Date,
  ) {
    this.validate()
  }

  private validate(): void {
    if (!this.name || this.name.trim().length < 2)
      throw new Error("Nome inválido")
    if (!this.isValidCpf(this.cpf))
      throw new Error("CPF inválido")
    if (!this.phone || this.phone.trim().length < 8)
      throw new Error("Telefone inválido")
    if (this.birthDate > new Date())
      throw new Error("Data de nascimento não pode ser futura")
  }

  private isValidCpf(cpf: string): boolean {
    const cleaned = cpf.replace(/\D/g, "")
    if (cleaned.length !== 11) return false
    if (/^(\d)\1+$/.test(cleaned)) return false
    const calc = (mod: number) =>
      cleaned
        .slice(0, mod - 1)
        .split("")
        .reduce((sum, d, i) => sum + Number(d) * (mod - i), 0)
    const d1 = ((calc(10) * 10) % 11) % 10
    const d2 = ((calc(11) * 10) % 11) % 10
    return d1 === Number(cleaned[9]) && d2 === Number(cleaned[10])
  }

  static create(name: string, cpf: string, phone: string, birthDate: Date): Patient {
    return new Patient(crypto.randomUUID(), name, cpf, phone, birthDate)
  }
}
```

> `static create` é o método de fábrica — gera o UUID e instancia a entidade.
> `createdAt` é opcional — não existe ao criar, só ao buscar do banco.
> As validações ficam na entidade — é responsabilidade do domínio garantir dados válidos.

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
> A interface vive no domínio — é o contrato que a infra deve implementar.

---

## 14. Repositório in-memory (para testes)

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

> O repositório in-memory vive em `tests/` — é exclusivo para testes, nunca vai para produção.
> Permite testar use cases sem banco de dados rodando.

---

## 15. Use Cases

### Tipos de input

```typescript
// src/application/patient/patient.types.ts
export type CreatePatientInput = {
  name: string
  cpf: string
  phone: string
  birthDate: Date
}
```

### Create

```typescript
// src/application/patient/create-patient.usecase.ts
import { Patient } from "@domain/patient/patient.entity"
import { PatientRepository } from "@domain/patient/patient.repository"
import { CreatePatientInput } from "./patient.types"

export class CreatePatientUseCase {
  constructor(private patientRepository: PatientRepository) {}

  async execute(input: CreatePatientInput): Promise<Patient> {
    const patient = Patient.create(input.name, input.cpf, input.phone, input.birthDate)
    await this.patientRepository.save(patient)
    return patient
  }
}
```

### List

```typescript
// src/application/patient/list-patients.usecase.ts
import { Patient } from "@domain/patient/patient.entity"
import { PatientRepository } from "@domain/patient/patient.repository"

export class ListPatientsUseCase {
  constructor(private patientRepository: PatientRepository) {}

  async execute(): Promise<Patient[]> {
    return this.patientRepository.findAll()
  }
}
```

### Get

```typescript
// src/application/patient/get-patient.usecase.ts
import { Patient } from "@domain/patient/patient.entity"
import { PatientRepository } from "@domain/patient/patient.repository"

export class GetPatientUseCase {
  constructor(private patientRepository: PatientRepository) {}

  async execute(id: string): Promise<Patient | null> {
    return this.patientRepository.findById(id)
  }
}
```

### Update

```typescript
// src/application/patient/update-patient.usecase.ts
import { Patient } from "@domain/patient/patient.entity"
import { PatientRepository } from "@domain/patient/patient.repository"

export class UpdatePatientUseCase {
  constructor(private patientRepository: PatientRepository) {}

  async execute(patient: Patient): Promise<void> {
    const found = await this.patientRepository.findById(patient.id)
    if (!found) throw new Error("Paciente não encontrado")
    await this.patientRepository.save(patient)
  }
}
```

### Delete

```typescript
// src/application/patient/delete-patient.usecase.ts
import { PatientRepository } from "@domain/patient/patient.repository"

export class DeletePatientUseCase {
  constructor(private patientRepository: PatientRepository) {}

  async execute(id: string): Promise<void> {
    const found = await this.patientRepository.findById(id)
    if (!found) throw new Error("Paciente não encontrado")
    await this.patientRepository.delete(id)
  }
}
```

> Cada use case tem uma única responsabilidade.
> `update` e `delete` validam se o paciente existe antes de operar.
> O repositório é recebido por injeção de dependência — nos testes passa o in-memory, em produção passa o Prisma.

---

## 16. Configurar o PostgreSQL com Docker

```bash
docker run --name salus-db \
  -e POSTGRES_USER=salus \
  -e POSTGRES_PASSWORD=salus \
  -e POSTGRES_DB=salus \
  -p 5432:5432 \
  -d postgres
```

Verificar se está rodando:

```bash
docker ps
```

---

## 17. Configurar o Prisma

```bash
npm install -D prisma
npm install @prisma/client @prisma/adapter-pg pg
npm install -D @types/pg
npx prisma init
```

Configure o `prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
}
```

> No Prisma v7 a `url` não fica mais no `schema.prisma` — fica no `prisma.config.ts`.

Configure o `prisma.config.ts` na raiz:

```typescript
import "dotenv/config"
import { defineConfig } from "prisma/config"

export default defineConfig({
  schema: "prisma/schema.prisma",
  migrations: {
    path: "prisma/migrations",
  },
  datasource: {
    url: process.env['DATABASE_URL'],
  },
})
```

Adicione o model Patient no `schema.prisma`:

```prisma
model Patient {
  id        String   @id @default(uuid())
  name      String
  cpf       String   @unique
  phone     String
  birthDate DateTime
  createdAt DateTime @default(now())
}
```

Rode a migration:

```bash
npx prisma migrate dev --name init
npx prisma generate
```

---

## 18. Criar o Prisma Client

```typescript
// src/infra/database/prisma/client.ts
import { PrismaClient } from "@prisma/client"
import { PrismaPg } from "@prisma/adapter-pg"

const adapter = new PrismaPg({
  connectionString: process.env['DATABASE_URL']!,
})

export const prisma = new PrismaClient({ adapter })
```

---

## 19. Implementar o PrismaPatientRepository

```typescript
// src/infra/database/prisma/patient.repository.ts
import { PatientRepository } from "@domain/patient/patient.repository"
import { Patient } from "@domain/patient/patient.entity"
import { prisma } from "./client"

export class PrismaPatientRepository implements PatientRepository {
  async save(patient: Patient): Promise<void> {
    await prisma.patient.upsert({
      where: { id: patient.id },
      create: {
        id: patient.id,
        name: patient.name,
        cpf: patient.cpf,
        phone: patient.phone,
        birthDate: patient.birthDate,
      },
      update: {
        name: patient.name,
        cpf: patient.cpf,
        phone: patient.phone,
        birthDate: patient.birthDate,
      },
    })
  }

  async findAll(): Promise<Patient[]> {
    const result = await prisma.patient.findMany()
    return result.map(
      (p) => new Patient(p.id, p.name, p.cpf, p.phone, p.birthDate, p.createdAt)
    )
  }

  async findById(id: string): Promise<Patient | null> {
    const result = await prisma.patient.findUnique({ where: { id } })
    if (!result) return null
    return new Patient(result.id, result.name, result.cpf, result.phone, result.birthDate, result.createdAt)
  }

  async delete(id: string): Promise<void> {
    await prisma.patient.delete({ where: { id } })
  }
}
```

> `save` usa `upsert` — cria ou atualiza dependendo se o id já existe.
> O mapeamento `new Patient(...)` converte o objeto plano do Prisma para instância da entidade.
> Os testes de use case continuam usando o `InMemoryPatientRepository` — o Prisma só é usado em produção.

---

## 20. Factory

```typescript
// src/infra/factories/patient.factory.ts
import { PrismaPatientRepository } from "@infra/database/prisma/patient.repository"
import { CreatePatientUseCase } from "@application/patient/create-patient.usecase"
import { GetPatientUseCase } from "@application/patient/get-patient.usecase"
import { ListPatientsUseCase } from "@application/patient/list-patients.usecase"
import { UpdatePatientUseCase } from "@application/patient/update-patient.usecase"
import { DeletePatientUseCase } from "@application/patient/delete-patient.usecase"

const repository = new PrismaPatientRepository()

export function makePatientUseCases() {
  return {
    createUseCase: new CreatePatientUseCase(repository),
    getUseCase: new GetPatientUseCase(repository),
    listUseCase: new ListPatientsUseCase(repository),
    updateUseCase: new UpdatePatientUseCase(repository),
    deleteUseCase: new DeletePatientUseCase(repository),
  }
}
```

> O repositório é instanciado fora da função — todos os use cases compartilham a mesma instância.
> Para trocar de banco, basta trocar o repositório aqui — o resto do código não muda.

---

## 21. Controller

```typescript
// src/infra/http/controllers/patient.controller.ts
import { makePatientUseCases } from "@infra/factories/patient.factory"
import { Request, Response, NextFunction } from "express"
import { Patient } from "@domain/patient/patient.entity"

export class PatientController {
  constructor(private useCases: ReturnType<typeof makePatientUseCases>) {}

  async create(req: Request, res: Response, next: NextFunction) {
    try {
      const response = await this.useCases.createUseCase.execute({
        name: req.body.name,
        cpf: req.body.cpf,
        phone: req.body.phone,
        birthDate: new Date(req.body.birthDate),
      })
      res.status(201).json(response)
    } catch (error) {
      next(error)
    }
  }

  async getAll(req: Request, res: Response, next: NextFunction) {
    try {
      const response = await this.useCases.listUseCase.execute()
      res.status(200).json(response)
    } catch (error) {
      next(error)
    }
  }

  async getById(req: Request, res: Response, next: NextFunction) {
    try {
      const id = req.params['id'] as string
      const response = await this.useCases.getUseCase.execute(id)
      if (!response)
        return res.status(404).json({ message: "Paciente não encontrado" })
      res.status(200).json(response)
    } catch (error) {
      next(error)
    }
  }

  async update(req: Request, res: Response, next: NextFunction) {
    try {
      const id = req.params['id'] as string
      const updatedPatient = new Patient(
        id,
        req.body.name,
        req.body.cpf,
        req.body.phone,
        new Date(req.body.birthDate),
      )
      await this.useCases.updateUseCase.execute(updatedPatient)
      res.status(200).json(updatedPatient)
    } catch (error) {
      if (error instanceof Error && error.message === "Paciente não encontrado")
        return res.status(404).json({ message: error.message })
      next(error)
    }
  }

  async delete(req: Request, res: Response, next: NextFunction) {
    try {
      const id = req.params['id'] as string
      await this.useCases.deleteUseCase.execute(id)
      res.status(200).json({ message: "Paciente deletado com sucesso" })
    } catch (error) {
      if (error instanceof Error && error.message === "Paciente não encontrado")
        return res.status(404).json({ message: error.message })
      next(error)
    }
  }
}
```

> O controller recebe os use cases por injeção de dependência no construtor.
> O `birthDate` vem como string no body HTTP — converte para `Date` com `new Date()`.
> Erros de negócio conhecidos retornam 404 — erros desconhecidos passam para o `next`.

---

## 22. Rotas

```typescript
// src/infra/http/routes/patient.routes.ts
import { Router } from "express"
import { PatientController } from "../controllers/patient.controller"
import { makePatientUseCases } from "@infra/factories/patient.factory"

const patientRouter = Router()
const controller = new PatientController(makePatientUseCases())

patientRouter.post("/patients", controller.create.bind(controller))
patientRouter.get("/patients", controller.getAll.bind(controller))
patientRouter.get("/patients/:id", controller.getById.bind(controller))
patientRouter.put("/patients/:id", controller.update.bind(controller))
patientRouter.delete("/patients/:id", controller.delete.bind(controller))

export { patientRouter }
```

> `.bind(controller)` é necessário para preservar o contexto `this` quando o Express chama o método.

---

## Resumo das dependências

| Dependência | Tipo | Função |
|---|---|---|
| express | produção | framework HTTP |
| dotenv | produção | carrega o .env |
| zod | produção | validação de dados e env vars |
| @prisma/client | produção | client do banco de dados |
| @prisma/adapter-pg | produção | adapter PostgreSQL para Prisma v7 |
| pg | produção | driver PostgreSQL |
| typescript | dev | compilador TypeScript |
| @types/node | dev | tipos do Node.js |
| @types/express | dev | tipos do Express |
| @types/pg | dev | tipos do driver pg |
| tsx | dev | roda .ts direto sem compilar |
| prisma | dev | CLI do Prisma (migrations, generate) |
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

**Repositório in-memory** — implementação fake do repositório que guarda dados em array. Permite testar use cases sem banco de dados rodando. Vive em `tests/` — nunca vai para produção.

**Upsert** — operação do Prisma que cria ou atualiza um registro dependendo se a chave já existe. Usado no `save` do repositório para unificar create e update em um único método.

**Mapeamento de entidade** — o Prisma retorna objetos planos sem métodos. É necessário converter para instâncias da entidade usando `new Patient(...)` para preservar as validações e métodos do domínio.

**Factory** — função que instancia e conecta repositórios e use cases. É o único lugar onde a implementação concreta do repositório é definida — trocar de banco exige mudança em apenas um lugar.

**bind(controller)** — necessário ao passar métodos de classe como callbacks para o Express. Sem o `bind`, o `this` se perde e o controller não consegue acessar seus próprios atributos.

**Driver Adapter** — no Prisma v7 é obrigatório usar um adapter de driver (`PrismaPg` para PostgreSQL). O client não conecta ao banco diretamente sem o adapter.
