# Calculadora de Precatórios e Ativos Judiciais — MVP

Ferramenta financeira (HTML + CSS + JavaScript puro, **sem bibliotecas externas**) para
calcular, projetar e comparar valores de **precatórios federais**, **precatórios
estaduais/municipais** (com parâmetros editáveis da **EC 136/2025**) e **ativos judiciais**.

> ⚠️ **Aviso:** ferramenta **financeira**, não substitui análise jurídica. Todos os
> índices e projeções embarcados são **mock/demo fictícios**, claramente marcados, e
> devem ser substituídos por fontes oficiais (BCB/SGS, IBGE/SIDRA, CNJ, Planalto) antes
> de qualquer uso real. **Nenhum dado fictício é apresentado como real.**

## Como usar

Abra `index.html` em qualquer navegador moderno. Funciona **offline**.

Fluxo: preencha **Módulo A (Dados do ativo)** → ajuste índices/projeções (mock) nas abas
**B/C** → preencha o módulo do tipo de ativo (**D**, **E** ou **F**) → confira a aba
**EC 136/25** se for precatório municipal/estadual → clique **Calcular** → veja
**Resultados** (resumo, premissas, cenários, tabelas, gráficos, memória de cálculo) →
exporte **CSV/JSON** ou copie a memória.

## O que já está implementado (MVP)

- Abas/cards, inputs rotulados, design responsivo.
- Motor de cálculo modular: fator acumulado, correção monetária, juros simples/compostos,
  Selic acumulada, valor futuro, valor presente, deságio, fluxo de fila.
- Validações: data inválida, data de corte < data-base, taxa negativa, valor nulo, RCL
  vazia, percentual > 100%, índice ausente, ano de orçamento ausente. **Erros são exibidos
  e não travam silenciosamente.**
- Cenários (conservador/base/otimista/customizado), tabelas de atualização e fluxo,
  gráficos em `<canvas>` puro.
- Exportação **CSV** (memória) e **JSON** (inputs + premissas + outputs), copiar memória.
- Parâmetros da **EC 136/25 totalmente editáveis** (percentuais são *placeholders*, **não**
  extraídos automaticamente do texto oficial) + área de observações e links de fontes.
- Cuidado anti-dupla-contagem: quando o índice é **Selic**, juros de mora adicionais são
  zerados (Selic já engloba correção + juros).

### Limites honestos do MVP
- A correção monetária usa a **projeção anual mock** como *proxy* (sinalizado em todas as
  premissas/alertas). Não há série histórica real nem cálculo *pro rata die* oficial.
- Sem integração de API (CORS em HTML puro). `Dados.loadFromAPI()` é um *placeholder* que
  avisa em vez de inventar números.

## Substituindo os mocks por dados reais

Pontos de troca isolados de propósito:
- `Dados.mockMensal()` / `Dados.mockProjecao()` → fonte das séries/projeções.
- `Dados.loadFromAPI()` → integração HTTP real (via backend/proxy).
- `Motor.fatorCorrecao()` → trocar o *proxy* por fator acumulado real da série.

## Arquitetura de produção sugerida (próximo passo)

```
┌─────────────┐     HTTPS      ┌──────────────────┐
│  Frontend   │  ───────────▶  │   API (backend)  │
│ (SPA/HTML)  │  ◀───────────  │  REST/GraphQL    │
└─────────────┘                └────────┬─────────┘
                                        │
                       ┌────────────────┼─────────────────┐
                       ▼                ▼                  ▼
                ┌────────────┐   ┌────────────┐    ┌──────────────┐
                │ PostgreSQL │   │   Cache     │    │  Job diário  │
                │ (séries,   │   │  (Redis)    │    │ (cron/Airflow│
                │  cálculos) │   │             │    │  /Lambda)    │
                └────────────┘   └────────────┘    └──────┬───────┘
                                                          │ coleta
                              ┌───────────────────────────┼───────────────┐
                              ▼                           ▼                ▼
                        BCB/SGS API              IBGE/SIDRA API      CNJ / tribunais
                     (Selic, IPCA, INPC)      (IPCA, IPCA-E, INPC)  (regras/precatórios)
```

- **Backend**: Node/TypeScript (NestJS) ou Python (FastAPI). Resolve CORS, centraliza
  regras de negócio e versiona parâmetros jurídicos (EC 136/25) com histórico.
- **Banco**: PostgreSQL — tabelas `indice_serie` (índice, competência, valor, fonte,
  coletado_em), `projecao`, `parametro_legal` (versionado), `calculo` (auditoria de
  inputs/premissas/outputs).
- **Cache**: Redis para séries consolidadas e fatores acumulados.
- **Rotina diária de atualização dos índices**:
  1. Job agendado (cron / Airflow / AWS Lambda+EventBridge) roda de manhã.
  2. Coleta BCB/SGS (séries 11 Selic, 433 IPCA, 188 INPC etc.) e IBGE/SIDRA.
  3. Valida/normaliza, grava com `fonte` e `coletado_em`, invalida cache.
  4. Alerta se uma fonte falhar — **nunca** preenche com dado fabricado.
- **Parâmetros jurídicos**: tabela versionada + *feature flag* de override manual, para
  refletir regulamentação/CNJ/tribunal sem redeploy.
- **Observabilidade/auditoria**: cada cálculo persiste inputs, premissas e fonte de cada
  número, garantindo rastreabilidade (essencial para uso profissional).

## Importante sobre a EC 136/2025

Os percentuais de RCL e regras de split (acordo × ordem cronológica) são **placeholders
editáveis** e **não** foram extraídos automaticamente do texto oficial. Valide sempre em:
[Planalto — EC 136/2025](https://www.planalto.gov.br/ccivil_03/constituicao/emendas/emc/emc136.htm),
CNJ e tribunal competente.
