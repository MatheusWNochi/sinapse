---
description: Implementa uma fase de uma spec (ex.: /fase 000 2)
argument-hint: <spec> <fase>
---

Implemente a fase indicada em: $ARGUMENTS

- Spec `000` = `SPEC.md`; outras = `specs/<número>-*.md`.
- Se os argumentos estiverem ausentes ou ambíguos, rode o equivalente a `/spec-status` e pergunte qual fase implementar.

Passos:

1. Delegue a implementação ao subagente `spec-implementer`, passando a spec e o número da fase. Se a fase for essencialmente de regras de negócio em `src/domain/`, use o `domain-guardian` para essa parte.
2. Se o subagente devolver uma pergunta (ambiguidade, arquivos ausentes, escopo), repasse-a ao usuário e pare.
3. Ao terminar, delegue a revisão ao subagente `spec-reviewer`.
4. Se o parecer for REPROVADA, peça ao `spec-implementer` que corrija os problemas apontados e revise de novo (no máximo 2 ciclos; depois disso, reporte ao usuário).
5. Responda ao usuário com: o que foi feito, commits, parecer da revisão e o que ficou pendente (inclusive critérios de aceite que dependem de ação dele).
