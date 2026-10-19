# SPEC 000 — Sinapse: MVP inicial

> **Status:** aprovada
>
> Documento de especificação para implementação por agente de IA.
> Esta spec é a **fonte de verdade**. Em caso de dúvida ou ambiguidade, pare e pergunte antes de assumir.

---

## 1. Contexto

**Sinapse** é uma aplicação web fullstack para elaboração de orçamentos de obras da construção civil a partir da base pública **SINAPI** (Sistema Nacional de Pesquisa de Custos e Índices da Construção Civil, mantido pela Caixa e pelo IBGE).

O SINAPI publica mensalmente, por UF, preços de **insumos** e **composições** de serviços, em dois regimes de encargos sociais: **desonerado** e **não desonerado**.

### Objetivo do MVP

Permitir que um único usuário (sem autenticação):

1. Importe planilhas SINAPI de uma UF / mês de referência / regime.
2. Pesquise composições e insumos.
3. Monte um orçamento organizado em etapas e itens.
4. Configure o BDI e veja o preço de venda calculado.
5. Visualize a curva ABC dos serviços do orçamento.
6. Exporte o orçamento para Excel.

---

## 2. Instruções para o agente

- Trabalhe **por fases** (seção 10). Conclua uma fase, rode lint + testes, faça commit e só então avance.
- Use **Conventional Commits** (`feat:`, `fix:`, `test:`, `chore:`, `refactor:`, `docs:`).
- **Não invente o layout das planilhas SINAPI.** O formato oficial já mudou ao longo dos anos. Antes de implementar o parser, inspecione os arquivos de exemplo em `data/sinapi/`. Se não houver arquivos, pare e peça ao usuário.
- **Nunca use `number` (float) para valores monetários ou coeficientes.** Use `Decimal` (Prisma `Decimal` + `decimal.js`) em toda a camada de domínio.
- Regras de negócio ficam em `src/domain/`, como **funções puras sem dependência de framework, banco ou UI**. Elas devem ter testes unitários.
- Não adicione bibliotecas fora da stack (seção 3) sem justificar no commit.
- Não implemente nada listado em "Fora do escopo" (seção 11).
- Ao terminar cada fase, atualize os checkboxes da seção 10 neste arquivo.

---

## 3. Stack

Use as versões estáveis mais recentes no momento da instalação.

| Camada | Tecnologia |
| --- | --- |
| Linguagem | TypeScript (`strict: true`) |
| Framework | Next.js (App Router) |
| UI | MUI + MUI X DataGrid (versão gratuita) |
| Formulários | React Hook Form + Zod (`@hookform/resolvers`) |
| Data fetching | TanStack Query |
| ORM / Banco | Prisma + PostgreSQL |
| Decimais | `decimal.js` |
| Leitura/escrita de Excel | `exceljs` |
| Testes | Jest + Testing Library |
| Qualidade | ESLint + Prettier |
| Infra local | Docker Compose (apenas PostgreSQL) |
| CI | GitHub Actions (lint + typecheck + testes) |
| Gerenciador de pacotes | pnpm |

---

## 4. Estrutura de pastas

```
sinapse/
├── data/sinapi/                # planilhas de exemplo (não versionar arquivos grandes)
├── prisma/
│   ├── schema.prisma
│   └── seed.ts
├── specs/                      # specs futuras (001-..., 002-...)
├── scripts/
│   └── import-sinapi.ts        # CLI de importação
├── src/
│   ├── app/                    # rotas e páginas (App Router)
│   │   └── api/                # route handlers
│   ├── components/
│   ├── domain/                 # regras de negócio puras
│   │   ├── bdi/
│   │   ├── abc-curve/
│   │   └── budget/
│   ├── importers/sinapi/       # parser + normalização
│   ├── lib/                    # prisma client, decimal helpers, query client
│   ├── schemas/                # schemas Zod compartilhados front/back
│   └── server/                 # services e repositories (acesso a banco)
├── tests/fixtures/             # planilhas SINAPI reduzidas para testes
├── docker-compose.yml
├── .env.example
└── SPEC.md
```

Nomes de entidades e regras de domínio em **inglês, camelCase** (`Resource`, `Composition`, `Budget`). Tabelas e colunas do banco em **snake_case**, via `@map`/`@@map` do Prisma. UI continua em pt-BR.

---

## 5. Modelo de dados

Rascunho do `schema.prisma`. Ajuste se necessário, mas mantenha as constraints de unicidade.

```prisma
enum Regime {
  EXEMPT
  NON_EXEMPT
}

enum ItemType {
  RESOURCE
  COMPOSITION
}

model ReferenceBase {
  id               String            @id @default(cuid())
  state            String            @db.Char(2)
  referenceDate    DateTime          @db.Date @map("reference_date")    // sempre dia 1 do mês (ex.: 2026-09-01)
  regime           Regime
  importedAt       DateTime          @default(now()) @map("imported_at")
  resourcePrices   ResourcePrice[]
  compositionCosts CompositionCost[]
  budgets          Budget[]

  @@unique([state, referenceDate, regime])
  @@map("reference_bases")
}

model Resource {
  id          String          @id @default(cuid())
  code        String          @unique
  description String
  unit        String
  prices      ResourcePrice[]

  @@map("resources")
}

model ResourcePrice {
  id         String        @id @default(cuid())
  resourceId String        @map("resource_id")
  baseId     String        @map("base_id")
  price      Decimal       @db.Decimal(14, 4)
  resource   Resource      @relation(fields: [resourceId], references: [id])
  base       ReferenceBase @relation(fields: [baseId], references: [id], onDelete: Cascade)

  @@unique([resourceId, baseId])
  @@map("resource_prices")
}

model Composition {
  id          String            @id @default(cuid())
  code        String            @unique
  description String
  unit        String
  group       String?
  costs       CompositionCost[]

  @@map("compositions")
}

model CompositionCost {
  id            String        @id @default(cuid())
  compositionId String        @map("composition_id")
  baseId        String        @map("base_id")
  totalCost     Decimal       @db.Decimal(14, 4) @map("total_cost")
  composition   Composition   @relation(fields: [compositionId], references: [id])
  base          ReferenceBase @relation(fields: [baseId], references: [id], onDelete: Cascade)

  @@unique([compositionId, baseId])
  @@map("composition_costs")
}

model Budget {
  id        String        @id @default(cuid())
  name      String
  client    String?
  baseId    String        @map("base_id")
  bdi       Decimal       @db.Decimal(8, 6)    // fração: 0.2624 = 26,24%
  bdiParams Json?         @map("bdi_params")    // parâmetros usados no cálculo
  createdAt DateTime      @default(now()) @map("created_at")
  updatedAt DateTime      @updatedAt @map("updated_at")
  base      ReferenceBase @relation(fields: [baseId], references: [id])
  stages    Stage[]

  @@map("budgets")
}

model Stage {
  id       String       @id @default(cuid())
  budgetId String       @map("budget_id")
  order    Int
  name     String
  budget   Budget       @relation(fields: [budgetId], references: [id], onDelete: Cascade)
  items    BudgetItem[]

  @@map("stages")
}

model BudgetItem {
  id          String   @id @default(cuid())
  stageId     String   @map("stage_id")
  order       Int
  type        ItemType
  code        String                          // código SINAPI do insumo ou composição
  description String                          // snapshot no momento da inclusão
  unit        String
  quantity    Decimal  @db.Decimal(14, 4)
  unitCost    Decimal  @db.Decimal(14, 4) @map("unit_cost")    // snapshot do custo na base do orçamento
  stage       Stage    @relation(fields: [stageId], references: [id], onDelete: Cascade)

  @@map("budget_items")
}
```

**Decisão importante:** `BudgetItem` guarda um **snapshot** de descrição, unidade e custo unitário. Reimportar uma base não deve alterar orçamentos existentes.

No MVP, as composições usam o **custo total publicado pelo SINAPI**. A abertura analítica (coeficientes de cada insumo dentro da composição) fica para uma spec futura.

---

## 6. Regras de negócio

Todas em `src/domain/`, usando `Decimal`. Arredondamento apenas na **apresentação** (2 casas, `ROUND_HALF_UP`), nunca nos cálculos intermediários.

### 6.1 Item do orçamento

```
totalItemCost  = quantity × unitCost
unitPrice      = unitCost × (1 + bdi)
totalItemPrice = quantity × unitPrice
```

### 6.2 Totais

```
totalStageCost  = Σ totalItemCost da etapa
totalBudgetCost = Σ totalStageCost
totalBudgetPrice = totalBudgetCost × (1 + bdi)
stageWeight (%) = totalStageCost / totalBudgetCost
```

Se `totalBudgetCost` for zero, `stageWeight` = 0 (sem divisão por zero).

### 6.3 BDI (fórmula de referência do TCU)

```
BDI = [ (1 + AC + S + R + G) × (1 + DF) × (1 + L) / (1 − I) ] − 1
```

| Parâmetro | Significado |
| --- | --- |
| AC | Administração central |
| S | Seguros |
| R | Riscos |
| G | Garantias |
| DF | Despesas financeiras |
| L | Lucro |
| I | Tributos (PIS + COFINS + ISS + CPRB, quando aplicável) |

Todos como fração (0.04 = 4%). Validar com Zod: cada parâmetro entre 0 e 1, e `I < 1`.

O usuário pode **informar o BDI diretamente** ou **calcular pelos parâmetros**. Quando calculado, salvar os parâmetros em `bdiParams`.

**Caso de teste obrigatório:**

| AC | S | R | G | DF | L | I | BDI esperado |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 0.04 | 0.008 | 0.0127 | 0 | 0.0123 | 0.074 | 0.0865 | 0.262401597… (≈ 26,24%) |

### 6.4 Curva ABC de serviços

1. Agrupar itens do orçamento por `code` + `type`, somando `totalItemCost` (o mesmo serviço pode aparecer em várias etapas).
2. Ordenar por custo total, decrescente.
3. Calcular percentual individual e percentual acumulado sobre o custo total do orçamento.
4. Classificar pelo percentual acumulado **antes** de somar o item:
   - **A**: acumulado anterior < 80%
   - **B**: acumulado anterior < 95%
   - **C**: restante
5. Limites 80/95 devem ser parâmetros com esses valores padrão.

Usar o acumulado anterior garante que o item que "cruza" a fronteira fique na classe superior, e que exista sempre pelo menos um item A.

Casos de teste mínimos: orçamento vazio, um único item (deve ser A), itens repetidos em etapas diferentes, empate de valores (ordem estável por código).

---

## 7. Importador SINAPI

### Requisitos

- CLI: `pnpm import:sinapi --uf=PR --ref=2026-09 --regime=nao-desonerado --file=data/sinapi/arquivo.xlsx`
- Também disponível pela UI (upload de arquivo + seleção de UF/mês/regime), reutilizando o mesmo serviço.
- **Primeiro passo:** inspecionar as planilhas reais em `data/sinapi/` e documentar o layout encontrado (abas, linha de cabeçalho, colunas) em um comentário no topo do parser. Se não houver arquivos, pedir ao usuário.
- Detectar a linha de cabeçalho dinamicamente (as planilhas oficiais têm linhas de título antes dos dados), procurando pelos nomes de coluna em vez de um índice fixo.
- Normalizar: `trim` em textos, código como string (preservar zeros à esquerda), preço com vírgula ou ponto convertido para `Decimal`.
- **Idempotente:** reimportar a mesma UF/referência/regime faz upsert, não duplica.
- Toda a importação em **uma transação**. Falhou uma linha crítica, nada é gravado.
- Linhas inválidas não críticas (preço vazio, por exemplo) são ignoradas e reportadas.
- Inserção em lotes (`createMany` / upserts em chunks) para suportar dezenas de milhares de linhas.
- Ao final, retornar um resumo: insumos importados, composições importadas, linhas ignoradas (com motivo), tempo total.

### Testes

- Criar fixtures reduzidas em `tests/fixtures/` (poucas dezenas de linhas) que reproduzam o layout real.
- Testar: parsing correto, detecção de cabeçalho, códigos com zero à esquerda, idempotência, linha inválida reportada.

---

## 8. Telas

| Rota | Descrição |
| --- | --- |
| `/` | Dashboard simples: bases importadas e últimos orçamentos |
| `/bases` | Lista de bases importadas + formulário de importação |
| `/catalog` | Busca de insumos e composições por código/descrição, filtrando por base |
| `/budgets` | Lista de orçamentos (criar, duplicar, excluir) |
| `/budgets/new` | Formulário: nome, cliente, base de referência, BDI |
| `/budgets/[id]` | Editor do orçamento (ver abaixo) |
| `/budgets/[id]/abc` | Curva ABC em tabela + gráfico de Pareto simples |

### Editor do orçamento (`/budgets/[id]`)

- Etapas em blocos colapsáveis, com subtotal e peso percentual.
- Adicionar / renomear / reordenar / excluir etapas.
- Adicionar item via busca com autocomplete no catálogo da base do orçamento.
- Editar quantidade inline; totais recalculam imediatamente (atualização otimista com TanStack Query).
- Painel lateral ou cabeçalho fixo com: custo total, BDI, preço total.
- Modal de BDI com duas abas: "Informar valor" e "Calcular pelos parâmetros".
- Botão "Exportar Excel".

Interface em **pt-BR**, valores formatados com `Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' })`.

---

## 9. API

Route handlers em `src/app/api/`. Toda entrada validada com schemas Zod de `src/schemas/`. Erros no formato `{ error: { code, message, details? } }` com status HTTP adequado.

| Método | Rota | Descrição |
| --- | --- | --- |
| GET | `/api/bases` | Lista bases |
| POST | `/api/bases/import` | Upload e importação (multipart) |
| DELETE | `/api/bases/:id` | Remove base (bloquear se houver orçamentos vinculados) |
| GET | `/api/catalog?baseId=&q=&type=&page=` | Busca paginada |
| GET | `/api/budgets` | Lista |
| POST | `/api/budgets` | Cria |
| GET | `/api/budgets/:id` | Detalhe com etapas, itens e totais calculados |
| PATCH | `/api/budgets/:id` | Atualiza dados gerais e BDI |
| DELETE | `/api/budgets/:id` | Exclui |
| POST | `/api/budgets/:id/duplicate` | Duplica orçamento completo |
| POST | `/api/budgets/:id/stages` | Cria etapa |
| PATCH / DELETE | `/api/stages/:id` | Atualiza / exclui etapa |
| POST | `/api/stages/:id/items` | Adiciona item (busca custo na base e grava snapshot) |
| PATCH / DELETE | `/api/items/:id` | Atualiza quantidade / exclui item |
| GET | `/api/budgets/:id/abc` | Curva ABC |
| GET | `/api/budgets/:id/export.xlsx` | Exportação Excel |

A busca no catálogo deve usar índice adequado (ex.: `pg_trgm` em `description`) para responder rápido com dezenas de milhares de registros.

### Exportação Excel

Uma planilha com: cabeçalho (nome, cliente, base, BDI), tabela por etapa (item, código, descrição, unidade, quantidade, custo unitário, custo total, preço unitário, preço total), subtotais por etapa e totais gerais. Segunda aba com a curva ABC. Valores numéricos como número (não texto), com formatação de moeda.

---

## 10. Fases de implementação

### Fase 1 — Setup
- [ ] Projeto Next.js + TypeScript strict + ESLint + Prettier
- [ ] Docker Compose com PostgreSQL e `.env.example`
- [ ] Prisma configurado com o schema da seção 5 e primeira migration
- [ ] Jest configurado e um teste de exemplo passando
- [ ] Workflow do GitHub Actions (lint, typecheck, test)
- [ ] Scripts no `package.json`: `dev`, `build`, `lint`, `typecheck`, `test`, `test:coverage`, `import:sinapi`

**Aceite:** `pnpm lint && pnpm typecheck && pnpm test` passam; `docker compose up -d && pnpm prisma migrate dev` funciona do zero.

### Fase 2 — Domínio
- [ ] Helpers de `Decimal` e formatação em `src/lib/`
- [ ] `src/domain/orcamento`: cálculos de item, etapa e orçamento
- [ ] `src/domain/bdi`: fórmula TCU + validação
- [ ] `src/domain/curva-abc`: classificação
- [ ] Testes unitários com cobertura ≥ 90% em `src/domain/`

**Aceite:** todos os casos de teste da seção 6 passam.

### Fase 3 — Importador
- [ ] Inspeção das planilhas reais e documentação do layout
- [ ] Parser com detecção de cabeçalho
- [ ] Serviço de importação transacional e idempotente
- [ ] CLI `import:sinapi`
- [ ] Fixtures e testes

**Aceite:** importar uma base real completa sem erro; reimportar não duplica registros.

### Fase 4 — API
- [ ] Schemas Zod
- [ ] Services/repositories em `src/server/`
- [ ] Route handlers da seção 9 (exceto exportação)
- [ ] Testes de integração dos services principais

**Aceite:** é possível criar um orçamento completo apenas via API.

### Fase 5 — Interface
- [ ] Layout base com MUI (navegação lateral, tema claro/escuro)
- [ ] Telas `/bases` e `/catalogo`
- [ ] Telas `/orcamentos` e `/orcamentos/novo`
- [ ] Editor do orçamento com atualização otimista
- [ ] Modal de BDI
- [ ] Tela da curva ABC

**Aceite:** fluxo completo importar → criar orçamento → adicionar itens → ajustar BDI → ver ABC funciona pela UI.

### Fase 6 — Exportação e acabamento
- [ ] Exportação Excel
- [ ] Estados de carregamento, vazio e erro em todas as telas
- [ ] Seed com uma base de exemplo e um orçamento demonstrativo
- [ ] README do repositório

**Aceite:** arquivo Excel abre corretamente e os totais batem com a tela.

---

## 11. Fora do escopo (MVP)

- Autenticação e multiusuário
- Composições próprias e abertura analítica de composições
- Cronograma físico-financeiro
- Atualização de orçamento para nova data-base
- Histórico de preços
- Exportação em PDF
- Download automático das planilhas no site da Caixa
- Deploy em nuvem

Cada item acima será uma spec própria em `specs/`.

---

## 12. Definition of Done (vale para toda fase)

- Lint, typecheck e testes passando
- Sem `any` explícito e sem `@ts-ignore`
- Nenhum valor monetário manipulado como `number`
- Entradas de API validadas com Zod
- Commits seguindo Conventional Commits
- Checkboxes desta spec atualizados
