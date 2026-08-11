# AssistBR — Data Warehouse de Assistência 24h & Seguro Residencial

> Projeto de portfólio em Engenharia de Dados: modelagem dimensional completa (Kimball / Constellation Schema) de uma empresa fictícia de assistência 24h automotiva e residencial, da ingestão de múltiplas fontes até o consumo em Power BI.

---

## 🎯 O que este projeto resolve

A AssistBR é uma empresa fictícia que presta dois tipos de serviço, que não podem ser confundidos entre si:

- **Assistência 24h** — emergências automotivas e residenciais (ex.: guincho, cano estourado).
- **Seguro residencial agendado** — serviços programados (ex.: limpeza de caixa d'água).

Os dados nascem espalhados em três fontes sem integração nativa entre si:

| Fonte | Conteúdo | Característica |
|---|---|---|
| SQL Server (Operacional) | Chamados e atendimentos | Sistema transacional principal |
| SQL Server (Financeiro) | Pagamentos e lançamentos | Servidor separado, sem integração |
| Excel (Prestadores/RH) | Cadastro de prestadores | Sem governança, sem padrão, sem lineage |

O projeto constrói o pipeline completo — **Bronze → Silver → Gold (Databricks) → Power BI** — e, mais importante, documenta **o processo de decisão de modelagem dimensional do zero**: o que vira fato, o que vira dimensão, como tratar histórico, qual granularidade cada relação exige.

---

## 🏗️ Arquitetura

```
SQL Server (Operacional) ─┐
SQL Server (Financeiro)  ─┼→ Bronze → Silver → Gold → Power BI
Excel (Prestadores)      ─┘
```

| Camada | Responsabilidade |
|---|---|
| Transacional (SSMS) | Escrita transacional, sistemas de origem |
| Bronze | Dado bruto, preservado como na fonte |
| Silver | Limpeza, padronização, integração entre fontes, **cálculos imutáveis** |
| Gold | Modelo dimensional (fatos/dimensões), **regras mutáveis** |
| Consumo | Power BI (motor Tabular / VertiPaq) |

**Schema dimensional:** Constellation Schema (Galaxy) — três fatos compartilhando dimensões, unidas por `bk_attendance`.

```
                    ┌─────────────────┐
                    │   dim_period    │
                    └────────┬────────┘
    ┌────────────────────────┼────────────────────────┐
┌───┴──────────┐    ┌────────┴────────┐    ┌──────────┴───────┐
│ fact_        │    │ fact_           │    │ fact_            │
│ attendance   │    │ interaction     │    │ financial        │
└───┬──────────┘    └────────┬────────┘    └──────────┬───────┘
    └────────────────────────┼────────────────────────┘
              Dimensões compartilhadas:
       client, address, contract, plan, asset, service...
```

**Princípio central de separação de camadas:** métricas de tempo de atendimento (TMA/TMC) e contagens de eventos passados são **fatos imutáveis**, calculados uma vez na Silver. Regras de negócio que podem mudar (SLA, limites de cobertura, mensalidade) ficam na Gold — corrigir uma regra errada vira **um UPDATE numa dimensão**, não o reprocessamento de uma partição inteira de fatos históricos.

---

## 📐 Decisões de arquitetura que valem destaque

Este projeto não é só "aqui está o schema final" — é um registro de **por que** cada decisão foi tomada, quais alternativas foram descartadas e qual o trade-off aceito. Alguns exemplos:

### 1. SCD (Slowly Changing Dimension) decidida atributo por atributo, não por tabela inteira
Em vez de marcar uma dimensão inteira como "Type 2", cada coluna é avaliada individualmente com uma pergunta prática: *"se esse atributo mudar, alguém vai perguntar 'qual era o valor dele no momento do evento X'?"*
Exemplo: numa dimensão de veículo, a **placa** é Type 2 (peso jurídico/legal, auditoria de sinistro exige o valor histórico), mas **cor** é Type 1 — muda, mas tem baixo valor de reconstituição histórica (é identificação visual operacional, não um dado com peso de disputa).

### 2. Bridge versionada em vez de FK fixa, quando a *relação* muda mais do que a *entidade*
Quando um funcionário autorizado pode ser desvinculado de um contrato sem o contrato encerrar, ou quando um veículo pode entrar/sair de uma frota PJ sem o contrato terminar, a vigência não pertence à dimensão da entidade — pertence à **relação**. Solução: tabela bridge com `dt_start`/`dt_end`/`fl_current`, versionada com SCD Type 2 aplicado à própria bridge. Isso preserva a capacidade de responder "quem/o que estava vinculado ao contrato numa data passada", mesmo depois de mudanças.

### 3. FK nullable por design de domínio ≠ FK nullable por dado ausente
Uma fato de atendimento pode se referir a um ativo automotivo **ou** residencial, nunca aos dois. A solução adotada foi duas colunas de FK (`fk_asset_vehicle`, `fk_asset_property`), sempre uma delas `NULL` — discriminadas por uma coluna de categoria já existente na fato. Isso quebra, à primeira vista, o princípio geral do projeto de "nunca deixar FK opcional em `NULL`" — mas esse princípio existe para **ausência de dado**; aqui o `NULL` é uma **consequência estrutural do tipo do evento**, não informação faltando. A alternativa (FK polimórfica única, apontando dinamicamente para uma de duas tabelas) foi descartada porque o motor Tabular do Power BI exige relacionamentos fixos entre tabelas — decisão de modelagem sendo puxada pela camada de consumo, não só pela teoria relacional.

### 4. Dado inferido nunca se disfarça de dado confirmado
Quando a fonte operacional não grava vínculo entre um contrato antigo e sua renovação, o vínculo é reconstruído por heurística (mesmo cliente + janela de datas). Em vez de simplesmente preencher a FK como se fosse fato, o modelo carrega uma flag explícita (`fl_previous_contract_inferred`) — qualquer consumidor do dado sabe, sem ambiguidade, que aquele vínculo é inferência, não confirmação da fonte.

### 5. Reuso e grão decidem se algo vira dimensão própria, bridge, ou coluna inline
O mesmo teste é aplicado de forma consistente ao longo do projeto: um atributo só vira dimensão/bridge separada se tiver reuso real (várias entidades compartilhando o mesmo valor) ou grão próprio (uma relação N:N genuína). SLA, por exemplo, é 1:1 com o plano — vira coluna Type 2 dentro da própria dimensão de plano, não uma bridge. Cobertura por tipo de serviço, por outro lado, é N:N real entre plano e tipo de serviço — essa sim vira bridge.

---

## 🧭 Metodologia — mentoria técnica orientada por IA

Este projeto foi desenvolvido através de um formato deliberado de **mentoria técnica socrática**, usando o Claude (Anthropic) como Tech Lead simulado.

**O que isso significa na prática:** a IA não desenhou o schema. Em cada decisão de modelagem, o papel do Claude foi **questionar premissas, apresentar alternativas e trade-offs, e forçar justificativa técnica** antes de qualquer estrutura ser fechada — nunca entregar a resposta pronta de imediato. As decisões, os erros cometidos no caminho, e as correções de raciocínio são meus.

Por que isso importa pra quem está avaliando este repositório: a maior parte de um portfólio técnico mostra só o resultado final. Este projeto documenta também o **processo de decisão** — inclusive os momentos em que uma primeira resposta estava errada (por exemplo, classificar um atributo como Type 1 pelo motivo errado, e ser confrontado com um contraexemplo até chegar na justificativa correta). Isso fica registrado, sem edição retroativa, no changelog de `PROJECT_CONTEXT.md`.

**Regra de trabalho seguida à risca ao longo do projeto:** toda decisão fechada em conversa é propagada, no mesmo dia, para os quatro documentos técnicos do projeto — nunca existe uma decisão "só na cabeça", sem registro escrito com data, contexto e alternativas descartadas.

---

## 📂 Estrutura do repositório

```
assistbr-data-warehouse/
├── README.md                          ← este arquivo
└── docs/
    ├── PROJECT_CONTEXT.md             ← documento canônico: arquitetura, decisões, changelog completo
    ├── AssistBR_Modelagem_Tecnica.md  ← especificação técnica detalhada de cada fato/dimensão
    ├── AssistBR_Atividades_Portfolio.md ← acompanhamento de progresso por fase
    └── Diagrama_ER.md                 ← modelo relacional em DBML
```

| Documento | Serve para |
|---|---|
| [`PROJECT_CONTEXT.md`](./docs/PROJECT_CONTEXT.md) | Fonte única de verdade. Toda decisão arquitetural, com data, motivação e alternativas descartadas. Changelog cronológico completo. |
| [`AssistBR_Modelagem_Tecnica.md`](./docs/AssistBR_Modelagem_Tecnica.md) | Estrutura de colunas de cada fato e dimensão, classificação SCD atributo por atributo, regras de cálculo por camada. |
| [`AssistBR_Atividades_Portfolio.md`](./docs/AssistBR_Atividades_Portfolio.md) | Progresso do projeto por fase — o que está fechado, em andamento, ou pendente. |
| [`Diagrama_ER.md`](./docs/Diagrama_ER.md) | Modelo relacional completo em DBML (renderizável em [dbdiagram.io](https://dbdiagram.io)). |

---

## 📊 Estado atual do modelo

**Fechado (estrutura + regras de negócio validadas):**
- Fato Atendimento (`fact_attendance`) — grão, todas as colunas, métricas TMA/TMC
- `dim_client`, `dim_address` (role-playing dimension)
- Árvore completa de `dim_contract` — `dim_plan`, `dim_plan_coverage` (bridge), `dim_authorized_contact` (bridge versionada), `dim_status_reason`
- `dim_asset` — `dim_asset_vehicle`, `dim_asset_property` (tabelas físicas separadas) + bridges versionadas de vínculo com contrato

**Em andamento:**
- Fato Interação e Fato Financeiro — grão definido, colunas pendentes de detalhamento
- Dimensões restantes: Analyst, Service, Modality, Provider, Channel, Sector

**Pendências de negócio em aberto** (perguntas que exigem validação com stakeholder antes de virar estrutura definitiva, não decisões técnicas):
- Identificador único do imóvel na fonte operacional (business key de `dim_asset_property`)
- Se a metragem do imóvel influencia precificação ou limite de cobertura
- Se contratos de frota PJ podem ter cobertura diferenciada por ativo dentro do mesmo contrato

---

## 🛠️ Stack

- **Databricks** — arquitetura Medallion (Bronze / Silver / Gold)
- **Power BI** — camada de consumo (motor Tabular / VertiPaq), star/constellation schema
- **SQL Server** — sistemas transacionais de origem (simulados)
- **Modelagem dimensional** — padrão Kimball, Constellation Schema

---

## 👤 Autor

Guilherme — Applications Development Engineer | Data Engineering

O AssistBR nasceu do meu objetivo de dominar o ciclo de vida dos dados de ponta a ponta. Construí este projeto para assumir o protagonismo arquitetural: desenhar a modelagem, estruturar os pipelines e definir as regras de negócio do absoluto zero. É um laboratório prático que consolida a visão estratégica e técnica, demonstrando a minha capacidade não apenas de operar ecossistemas existentes, mas de projetar soluções de dados robustas, escaláveis e alinhadas ao negócio.
