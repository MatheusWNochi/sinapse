# Sinapse — guia para agentes

Aplicação web para orçamentos de obras a partir da base pública SINAPI. O desenvolvimento é **guiado por specs**.

## Fonte de verdade

- [SPEC.md](SPEC.md) é a **SPEC 000 (MVP)** e a fonte de verdade do projeto. Leia antes de qualquer tarefa.
- Specs seguintes ficam em [specs/](specs/) (`001-...`, `002-...`). Veja [specs/README.md](specs/README.md) para o ciclo de vida e o índice.
- Em caso de dúvida ou ambiguidade na spec, **pare e pergunte ao usuário**. Não assuma.
- Se uma tarefa pedida não estiver em nenhuma spec, ou estiver em "Fora do escopo", avise antes de implementar.

## Fluxo de trabalho

1. Trabalhe **uma fase por vez** (SPEC.md, seção 10). Não comece a próxima fase sem concluir a atual.
2. Ao terminar uma fase: `pnpm lint && pnpm typecheck && pnpm test`, marque os checkboxes da fase na spec e faça commit.
3. Commits em **Conventional Commits** (`feat:`, `fix:`, `test:`, `chore:`, `refactor:`, `docs:`), um commit por unidade lógica.
4. Bibliotecas fora da stack (SPEC.md, seção 3) só com justificativa no commit.

Comandos disponíveis:

| Comando | Uso |
| --- | --- |
| `/spec-status` | Progresso de todas as specs e próxima tarefa |
| `/fase <spec> <n>` | Implementa a fase `n` da spec (ex.: `/fase 000 2`) |
| `/revisar-fase <spec> <n>` | Revisa a fase contra critérios de aceite e Definition of Done |
| `/nova-spec <tema>` | Cria uma nova spec a partir do template |

Subagentes em [.claude/agents/](.claude/agents/): `spec-implementer`, `spec-reviewer`, `spec-writer`, `domain-guardian`.

## Regras que não se negociam

- **Nunca `number` para dinheiro ou coeficientes.** Use `Decimal` (Prisma `Decimal` + `decimal.js`). Arredondar só na apresentação (2 casas, `ROUND_HALF_UP`).
- Regras de negócio em `src/domain/` como **funções puras**, sem dependência de framework, banco ou UI, sempre com testes unitários.
- **Não inventar o layout das planilhas SINAPI.** Inspecione `data/sinapi/`; se estiver vazio, peça os arquivos.
- `ItemOrcamento` guarda snapshot de descrição, unidade e custo. Reimportar base não altera orçamentos.
- Sem `any` explícito, sem `@ts-ignore`. TypeScript `strict`.
- Toda entrada de API validada com Zod (`src/schemas/`). Erros no formato `{ error: { code, message, details? } }`.
- Entidades e domínio em português (`Insumo`, `Composicao`, `Orcamento`); infraestrutura pode ser em inglês.
- UI em pt-BR; moeda com `Intl.NumberFormat('pt-BR', { style: 'currency', currency: 'BRL' })`.

## Comandos do projeto

Gerenciador: **pnpm**.

```bash
docker compose up -d          # PostgreSQL local
pnpm prisma migrate dev       # migrations
pnpm dev                      # servidor de desenvolvimento
pnpm lint && pnpm typecheck && pnpm test
pnpm test:coverage            # domínio exige ≥ 90%
pnpm import:sinapi --uf=PR --ref=2026-09 --regime=nao-desonerado --file=data/sinapi/arquivo.xlsx
```

(Os scripts passam a existir ao final da Fase 1 da SPEC 000.)
