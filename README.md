# 🛡️ AssistBR: Plataforma de Dados para Assistência 24h e Seguro Residencial

![Status](https://img.shields.io/badge/Status-Em_Desenvolvimento-yellow?style=flat-square)
![Architecture](https://img.shields.io/badge/Arquitetura-Medallion-blue?style=flat-square)
![Modeling](https://img.shields.io/badge/Modelagem-Galaxy_Schema-brightgreen?style=flat-square)
![Stack](https://img.shields.io/badge/Stack-SQL_Server_|_Databricks_|_Power_BI-blueviolet?style=flat-square)

> **Resumo:** Plataforma analítica de ponta a ponta projetada para integrar e processar dados operacionais, financeiros e de prestadores de serviço, habilitando inteligência de negócio para uma operação de assistência 24h e seguro automotivo/residencial.

---

## 📌 Visão Geral do Projeto

O **AssistBR** resolve o problema clássico de silos de dados em ambientes corporativos: dados transacionais distribuídos entre diferentes bancos SQL Server (Operacional e Financeiro) e controles externos em planilhas Excel (Prestadores). O projeto visa consolidar essas informações, reconstruir históricos de contratos e atendimentos, e aplicar regras complexas de negócio para disponibilizar dados confiáveis para consumo em BI.

### 🎯 O Desafio de Negócio
A operação demanda análises consolidadas e relatórios automatizados para responder a perguntas críticas, tais como:
* 📈 Qual o volume e a evolução dos atendimentos ao longo do tempo?
* ⏱️ Quais prestadores apresentam maior tempo de atendimento (TMA/TMC) ou quebras de SLA?
* 🗺️ Quais regiões concentram o maior volume de acionamentos e custos associados?
* 💰 Qual é o custo médio por acionamento e a margem de lucro por plano, região e serviço?
* 👥 Qual a produtividade por analista e setor?
* 🔍 Onde existem divergências entre o valor financeiro esperado e o valor cobrado pelo prestador?

**Métricas-Alvo do Negócio:** SLA > 92%, NPS > 75%, TMA (Tempo Médio de Atendimento), TMC (Tempo Médio de Chegada).

---

## 🏗️ Arquitetura de Dados (Medallion)

A arquitetura foi desenhada baseada no padrão **Medallion**, garantindo a separação clara entre preservação, transformação e consumo dos dados.

| Camada | Responsabilidade |
| :--- | :--- |
| **Origens (OLTP)** | Sistemas legados e escrita operacional (SQL Server Operacional, SQL Server Financeiro, Excel). |
| 🥉 **Bronze** | Ingestão e preservação do dado bruto em seu formato original, criando um histórico imutável. |
| 🥈 **Silver** | Limpeza, padronização, integração (cruzamento de fontes) e cálculos imutáveis baseados em eventos (ex: cálculo de TMA e TMC). |
| 🥇 **Gold** | Modelagem dimensional otimizada para leitura, aplicação de regras mutáveis de negócio (SLA, cobertura) e estruturação para o BI. |
| 📊 **BI / Serving** | Camada semântica e dashboards para consumo executivo via Power BI e Databricks SQL. |

> **💡 Princípio Central:** Cálculos derivados de eventos (como tempos de atendimento) são resolvidos na camada **Silver**. Regras sujeitas a revisões comerciais (como metas de SLA e regras de cobertura) são aplicadas na camada **Gold**.

---

## 📐 Modelagem Dimensional

O Data Warehouse utiliza o modelo **Constellation Schema (Galaxy)**, justificado pela presença de processos com granularidades distintas que não se encaixam de forma eficiente em uma única tabela Fato.

### 📦 Tabelas Fato
| Objeto | Granularidade | Status |
| :--- | :--- | :--- |
| `fact_attendance` | 1 linha por serviço prestado dentro de um chamado | 🟢 Fechada |
| `fact_interaction` | 1 linha por interação do analista/setor no chamado | 🟡 Grão fechado; colunas pend. |
| `fact_financial` | 1 linha por lançamento financeiro do atendimento | 🟡 Regras fechadas; colunas pend. |

### 🗂️ Principais Dimensões Compartilhadas
* **`dim_address`**: Role-playing dimension (origem/destino em nível bairro/cidade/estado).
* **`dim_client`**: Clientes PF/PJ com histórico de mudanças.
* **`dim_contract`**: Unidade de vigência e regras de cobertura (SCD aplicada para versionamento de status).
* **`dim_plan` & `dim_plan_coverage`**: Regras de preços, SLA e limites (via bridge table).
* **`dim_asset_vehicle` / `property`**: Ativos polimórficos vinculados aos contratos e serviços.
* *(Em andamento: `dim_analyst`, `dim_service`, `dim_provider`, `dim_sector`, etc.)*

---

## ⚙️ Decisões de Engenharia

Para garantir escalabilidade e fidelidade analítica, as seguintes decisões técnicas foram adotadas:

1. **Granularidade Flexível:** A `fact_attendance` desce ao nível de "serviço prestado", permitindo múltiplos serviços em um único chamado sem perder a rastreabilidade.
2. **SCD Type 2 (Slowly Changing Dimensions):** Implementada rigorosamente em dimensões críticas (endereço, contratos, coberturas) onde o histórico possui peso financeiro ou jurídico.
3. **Role-playing Dimensions:** Uma única tabela física `dim_address` tratada via views na camada Gold para isolar semântica no Power BI.
4. **Degenerate Dimensions:** Atributos super específicos de eventos (como rua e número exato do incidente) permanecem na tabela Fato para evitar hiper-crescimento das dimensões.
5. **Chaves (SK vs. BK):** Uso estrito de Surrogate Keys (SK) no modelo analítico, mantendo Business Keys (BK) apenas para lineage e deduplicação.
6. **Linhas Sentinelas:** Substituição de `NULL` em Foreign Keys por chaves `-1` ("Não Aplicável"), otimizando filtros no front-end.
7. **Integração de Contratos:** Inferência de linhagem (renovações) calculada via window functions na Silver, já que o sistema legado não possui vínculo explícito.

---

## 🛠️ Stack Tecnológica

| Tecnologia | Papel no Ecossistema |
| :--- | :--- |
| **SQL Server** | Simulação do ambiente transacional (Operacional e Financeiro). |
| **Databricks** | Motor de processamento analítico e orquestração das camadas Bronze, Silver e Gold. |
| **SQL / Python** | Linguagens core para extração, transformação e automação de pipelines. |
| **Power BI** | Camada de visualização, relatórios gerenciais e acompanhamento de métricas. |
| **Git / GitHub** | Versionamento de código, controle de artefatos e documentação. |

---

## 🗺️ Roadmap de Implementação

- [x] **Fase 1: Descoberta & Requisitos** (Levantamento de processos, SLA, regras contratuais, mapeamento de fontes).
- [x] **Fase 2: Arquitetura & Modelagem** (Definição Medallion, arquitetura Galaxy Schema, mapeamento de granularidades).
- [x] **Fase 3: Regras de Negócio** (Definição de cálculos TMA/TMC, SCD, chaves sentinelas, resolução de ativos polimórficos).
- [ ] **Fase 4: Infraestrutura & Origens** (Deploy do banco SQL Server simulado e geração dos dados transacionais/planilhas).
- [ ] **Fase 5: Pipelines de Dados (ETL/ELT)** (Implementação das ingestões Bronze, transformações Silver e materialização Gold no Databricks).
- [ ] **Fase 6: Qualidade de Dados** (Testes de unicidade, integridade referencial e reconciliação financeiro x operacional).
- [ ] **Fase 7: Analytics & BI** (Criação do modelo tabular e dashboards no Power BI).

---

## 📊 Métricas e Entregáveis Analíticos Esperados

| Domínio | Indicador | Descrição de Negócio |
| :--- | :--- | :--- |
| **Operações** | `SLA` | Percentual de atendimentos que cumpriram o prazo estipulado em contrato. |
| **Operações** | `TMA / TMC` | Tempo gasto em cada etapa do atendimento logístico. |
| **Finanças** | `Custo por Acionamento` | Rateio e identificação do custo unitário por serviço executado. |
| **Finanças** | `Expected vs Realized` | Tracking de divergências entre a tabela de preços do prestador e a fatura final. |
| **Finanças** | `Margem` | Rentabilidade segmentada por plano, região e tipo de serviço. |
| **Qualidade** | `NPS / Reacionamento` | Satisfação do cliente e volume de chamados reabertos pelo mesmo problema. |
| **Performance** | `Produtividade` | Eficiência das equipes de analistas, backoffice e operações de campo. |
