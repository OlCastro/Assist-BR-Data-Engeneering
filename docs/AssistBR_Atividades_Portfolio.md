# AssistBR — Atividades do Projeto (até o momento)

**Projeto:** Engenharia de Dados — BI para assistência 24h  
**Empresa fictícia:** AssistBR  
**Escopo deste documento:** atividades realizadas e entregas produzidas até a fase atual do projeto

---

## Objetivo do projeto

Construir um projeto completo de engenharia de dados para portfólio, cobrindo desde a compreensão do negócio até a modelagem dimensional, com arquitetura em camadas e stack alinhada ao mercado brasileiro (SQL Server + Databricks).

---

## Fase 1 — Fundamentos de modelagem

**Contexto:** exercícios introdutórios antes do projeto AssistBR.

| Atividade | Descrição | Entrega |
|-----------|-----------|---------|
| Conceitos fato × dimensão | Diferenciação entre eventos mensuráveis e contexto descritivo | Raciocínio documentado na mentoria |
| Chaves compostas × surrogate key | Unicidade de linha em cenários com múltiplos itens por evento | Exercício com vendas multi-produto |
| Modelo de auditoria de acessos | Primeiro star schema completo (fato + 4 dimensões) | Modelo com fato de acessos, dimensões de usuário, sistema, recurso e período |
| SQL analítico | Queries de agregação, JOINs, CTEs e window functions | Exercícios sobre o modelo de auditoria |

---

## Fase 2 — Levantamento de negócio

**Contexto:** simulação de entrevista com o Diretor de Operações da AssistBR.

| Atividade                           | Descrição                                                                       | Entrega                   |
| ----------------------------------- | ------------------------------------------------------------------------------- | ------------------------- |
| Entrevista de requisitos — rodada 1 | Negócio, fluxo operacional, SLA, métricas, metas, fontes de dados               | Informações coletadas     |
| Entrevista de requisitos — rodada 2 | Clientes PF/PJ, ativos cobertos, responsáveis autorizados, limites de cobertura | Informações coletadas     |
| Entrevista de requisitos — rodada 3 | Contratos, operação da central, prestadores, financeiro                         | Informações coletadas     |
| Entrevista de requisitos — rodada 4 (2026-07-19) | Renovação de contrato (sem vínculo na fonte), janela do limite QTD_ANUAL (aniversário do contrato), política de mensalidade PF/PJ e reajuste anual | Informações coletadas, 3 pendências de negócio resolvidas |
| Consolidação por partes             | Revisão estruturada: negócio, planos, prestadores, financeiro, operação         | Validação dos requisitos  |
| Documento de levantamento           | Registro objetivo das informações da entrevista                                 | [[Levantamento_AssistBR]] |

---

## Fase 3 — Análise transacional × analítica

| Atividade | Descrição | Entrega |
|-----------|-----------|---------|
| Diferenciação OLTP × OLAP | Objetivo, estrutura, risco e uso de cada camada | Compreensão aplicada ao contexto AssistBR |
| Normalização × dimensional | Por que o transacional não serve como base direta de BI | Quadro comparativo consolidado |
| Identificação de fontes | Mapeamento das 3 fontes e seus problemas de integração | Inventário de fontes documentado |

---

## Fase 4 — Definição da arquitetura de dados

| Atividade | Descrição | Entrega |
|-----------|-----------|---------|
| Arquitetura Medallion | Definição das camadas Bronze, Silver e Gold | Fluxo de dados definido |
| Stack tecnológica | SQL Server (SSMS) como transacional; Databricks para camadas analíticas | Arquitetura de referência |
| Responsabilidade por camada | Bronze = bruto; Silver = limpeza e integração; Gold = modelo dimensional | Papéis das camadas definidos |
| Regras de transformação | TMA/TMC na Silver; regras mutáveis (ex.: SLA) na Gold | Política de cálculos definida |
| Padrão de nomenclatura | Inglês abreviado com prefixos (`sk_`, `bk_`, `id_`, `dt_`, `hr_`, `fl_`, `nr_`, `vl_`) | Convenção documentada |
| Dicionário de dados | Complemento ao padrão abreviado para legibilidade | Previsto como entrega do portfólio |

---

## Fase 5 — Modelagem dimensional (em andamento)

| Atividade | Descrição | Status |
|-----------|-----------|--------|
| Separação por processo de negócio | Operacional, financeiro e interações como processos distintos | Concluído |
| Definição das fatos | Fato Atendimento, Fato Interação, Fato Financeiro | Concluído |
| Granularidade — Fato Atendimento | Uma linha por serviço prestado dentro de um chamado | Concluído |
| Granularidade — Fato Interação | Uma linha por interação de analista/setor no chamado | Concluído |
| Granularidade — Fato Financeiro | Uma linha por lançamento financeiro vinculado ao atendimento | Concluído |
| Constellation Schema | Múltiplas fatos compartilhando dimensões comuns | Concluído |
| Fato Atendimento — colunas | Estrutura final com chaves, FKs, flags, eventos e métricas | Concluído |
| Decisões de SCD | Atributos Type 1 e Type 2 por entidade (contrato, prestador, cliente) | Concluído |
| dim_address | Role-playing dimension, SCD Type 1, BK composta, linha sentinela | Concluído |
| dim_service_type | Categoria × urgência; SCD ainda não definida formalmente | Estrutura concluída, SCD pendente |
| dim_client | PF/PJ nativo, endereço versionado (Type 2), múltiplos contratos resolvidos via dim_contract | Concluído |
| dim_contract | Grão 1 linha/contrato, status versionado (Type 2), person_type_cd replicado. Inclui `fk_previous_contract` (inferido, heurística 30 dias) + `fl_previous_contract_inferred` e `vl_monthly_fee_negotiated` (override PJ) | Concluído (estrutura) |
| dim_plan_coverage | Bridge SCD Type 2, colunas finais fechadas. Dois padrões de cálculo decididos: KM/VALOR (comparação direta na Gold) e QTD_ANUAL (contagem acumulada na Silver via `nr_contract_attendance_seq`, comparada na Gold). Janela do QTD_ANUAL confirmada com o diretor: aniversário do contrato | Concluído (estrutura) |
| dim_plan | `bk_plan` BK real, `plan_desc` Type 1, SLA e mensalidade (`vl_monthly_fee`) como colunas Type 2 na própria dimensão — 1:1 com o plano, sem N:N que justifique tabela separada. Confirmado com o diretor: PF fixo, PJ negociado via `dim_contract` | Concluído (estrutura) |
| dim_authorized_contact, dim_status_reason | Bridge `bridge_contract_authorized_contact` versionada (SCD Type 2 — auditoria de quem estava autorizado em cada data); `dim_status_reason` com `reason_cd`/`ds_status_reason_detail` formalizados | Concluído (estrutura) |
| dim_asset | Duas tabelas físicas separadas (`dim_asset_vehicle`, `dim_asset_property`), sem supertype/subtype comum. BK do veículo = chassi; BK do imóvel pendente de validação com o diretor. Cardinalidade contrato×ativo: PF 1:1, PJ 1:N (frota), resolvida via `bridge_contract_asset_vehicle`/`bridge_contract_asset_property` (versionadas, SCD Type 2). `fact_attendance` ganha `fk_asset_vehicle`/`fk_asset_property` (nullable por design, discriminadas por `category_desc`) | Concluído (estrutura) |
| Fato Interação — colunas | Estrutura detalhada da tabela | Pendente |
| Fato Financeiro — colunas | Estrutura detalhada da tabela | Pendente |
| Elo fato_atendimento → dim_contract | `fk_contract` adicionado à fato, apontando para `sk_contract` vigente no momento do atendimento | Concluído |
| Elo fato_atendimento → dim_asset | `fk_asset_vehicle`/`fk_asset_property` adicionados à fato, nullable por design de domínio, discriminados por `category_desc` | Concluído |
| dim_provider | Cadastro de prestador (PJ) + `bridge_provider_service_type` (preço-padrão) + `bridge_provider_service_region_price` (exceção negociada por região, versionada) + `dim_provider_status_reason` (mini-dimensão própria). 2 pendências de negócio: premissa PJ, área de atuação | Estrutura em fechamento (2026-08-12) |
| Demais dimensões | Analyst, Service, Modality, Period, Channel, Sector | Pendente |
| Diagrama visual do modelo | Representação ER/star schema | Primeira versão produzida (DBML) |

---

## Fase 6 — Implementação (prevista)

| Atividade | Descrição | Status |
|-----------|-----------|--------|
| Banco transacional simulado | SQL Server com dados operacionais e financeiros | Pendente |
| Camada Bronze | Ingestão bruta das 3 fontes no Databricks | Pendente |
| Camada Silver | Padronização, integração e cálculo de TMA/TMC | Pendente |
| Camada Gold | Modelo dimensional materializado | Pendente |
| Pipeline ETL/ELT | Orquestração entre camadas | Pendente |
| Queries analíticas | SQL sobre o Gold | Pendente |
| Visualização / BI | Power BI ou Databricks SQL | Pendente |

---

## Entregáveis produzidos até aqui

| Entregável                               | Arquivo / formato                     |
| ---------------------------------------- | ------------------------------------- |
| Levantamento de requisitos de negócio    | [[Levantamento_AssistBR]]             |
| Resumo de atividades do projeto          | [[AssistBR_Atividades_Portfolio]]     |
| Resumo técnico de modelagem              | [[AssistBR_Modelagem_Tecnica]]        |
| Fato Atendimento (versão fechada)        | Diagrama visual (imagens na mentoria) |
| Modelo de auditoria (exercício anterior) | Documentado na conversa de mentoria   |

---

## Métricas de negócio mapeadas

As análises abaixo orientaram a modelagem e devem ser suportadas pelo projeto:

- Taxa de cumprimento de SLA (por plano, região, prestador)
- Tempo médio de atendimento (TMA) e tempo médio de chegada (TMC)
- Custo médio por acionamento
- Taxa de reacionamento
- NPS
- Produtividade por analista e por setor
- Margem por plano, região e tipo de serviço
- Divergência entre valor esperado e valor realizado (prestadores)

---

## Situação atual do projeto

**Concluído:** levantamento de negócio (incluindo 4ª rodada de entrevista, 2026-07-19), arquitetura de referência, convenções técnicas, estrutura das três fatos e colunas da Fato Atendimento, `dim_address`, `dim_service_type` (estrutura), `dim_client`, **toda a árvore de `dim_contract`** (`dim_plan`, `dim_plan_coverage`, `dim_authorized_contact`, `dim_status_reason`, fechadas em 2026-07-19) e **`dim_asset`** (`dim_asset_vehicle`, `dim_asset_property` + bridges versionadas, fechada em estrutura em 2026-08-09). **As 3 pendências de negócio da árvore de contrato também foram resolvidas** via entrevista com o diretor: linhagem de renovação (sem vínculo na fonte → `fk_previous_contract` inferido), janela do `QTD_ANUAL` (aniversário do contrato) e política de mensalidade PF/PJ (`dim_plan.vl_monthly_fee` + `dim_contract.vl_monthly_fee_negotiated`).

**Em andamento:** `dim_provider` (estrutura em fechamento, iniciada em 2026-08-12) e demais dimensões (`dim_analyst`, `dim_service`, `dim_modality`, `dim_channel`, `dim_sector`) e das fatos Interação e Financeiro.

**Pendências de negócio em aberto (2026-08-12):** (1) identificador único do imóvel na fonte operacional (BK de `dim_asset_property`); (2) se metragem do imóvel influencia precificação/cobertura; (3) hipótese não confirmada de planos/coberturas diferenciados por ativo dentro de um mesmo contrato de frota PJ; (4) premissa "prestador sempre PJ"; (5) área de atuação/cobertura regional do prestador (métrica real ou regra de despacho?); (6) existência do cenário de preço negociado por região para prestadores.

**Próxima entrega prevista:** fechamento de `dim_provider` (pendências de negócio via entrevista com o diretor) e abertura das dimensões restantes (`dim_analyst`, `dim_service`, `dim_modality`, `dim_channel`, `dim_sector`). Diagrama visual será atualizado em conjunto conforme novas dimensões forem fechadas.
