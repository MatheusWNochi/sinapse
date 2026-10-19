# Specs

Cada mudança relevante no Sinapse começa por uma spec. A spec é escrita para ser executada por um agente de IA: precisa ser específica, testável e dizer o que está fora do escopo.

## Convenções

- Arquivo: `specs/NNN-slug-curto.md` (ex.: `001-composicoes-analiticas.md`). Numeração sequencial, sem reaproveitar números.
- A **SPEC 000** (MVP) fica na raiz, em [`../SPEC.md`](../SPEC.md).
- Novas specs partem de [`_TEMPLATE.md`](_TEMPLATE.md) (use `/nova-spec <tema>`).
- Uma spec **complementa** as anteriores: não repita regras globais (stack, Definition of Done), apenas referencie e indique o que muda.
- Se uma spec alterar uma decisão anterior, registre isso na seção "Decisões" e atualize a spec antiga com um aviso apontando para a nova.

## Ciclo de vida

O status fica no cabeçalho de cada spec.

| Status | Significado |
| --- | --- |
| `rascunho` | Em escrita. Agentes **não** implementam. |
| `aprovada` | Revisada pelo usuário. Pode ser implementada. |
| `em andamento` | Ao menos uma fase iniciada. |
| `concluída` | Todas as fases com aceite verificado. |
| `substituída` | Trocada por outra spec (indicar qual). |

Só o usuário move uma spec de `rascunho` para `aprovada`.

## Execução por fases

1. `/fase <spec> <n>`: o `spec-implementer` implementa a fase, roda lint + typecheck + testes e faz os commits.
2. `/revisar-fase <spec> <n>`: o `spec-reviewer` confere os critérios de aceite e a Definition of Done e devolve um parecer.
3. Com parecer aprovado, os checkboxes da fase são marcados e a próxima fase pode começar.

## Índice

| Nº | Título | Status |
| --- | --- | --- |
| 000 | [MVP inicial](../SPEC.md) | aprovada |

## Backlog (SPEC 000, seção 11)

Itens fora do escopo do MVP que viram specs próprias. Ainda não escritas:

- Autenticação e multiusuário
- Composições próprias e abertura analítica de composições
- Cronograma físico-financeiro
- Atualização de orçamento para nova data-base
- Histórico de preços
- Exportação em PDF
- Download automático das planilhas no site da Caixa
- Deploy em nuvem
