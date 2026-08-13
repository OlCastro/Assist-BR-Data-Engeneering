# PROJECT_CONTEXT.md — AssistBR

> **Este é o documento canônico do projeto.** Toda decisão arquitetural relevante deve ser refletida aqui — este arquivo substitui a necessidade de reprocessar o histórico de conversas. Ao tomar uma nova decisão de modelagem, arquitetura ou escopo, **atualize a seção correspondente** (não crie um novo arquivo de changelog separado; mantenha este como fonte única de verdade).

---

## 1. Visão geral do projeto

**Nome:** AssistBR **Natureza:** Projeto de portfólio em Engenharia de Dados **Domínio simulado:** Empresa fictícia de assistência 24h + seguro residencial **Escopo do portfólio:** Banco transacional (SSMS) → Bronze → Silver → Gold (Databricks) → Power BI

**Autor/aprendiz:** Guilherme, Applications Development Engineer I. Nível atual: SQL 7, Python 3, Modelagem 6, ETL 5, Databricks 5. Objetivo: nível pleno, competitivo BR e internacional.

**Gap de habilidade sendo trabalhado:** no ambiente de trabalho de Guilherme, o pipeline e o modelo de dados já chegam prontos. Este projeto existe para forçar a tomada de decisão de modelagem do zero — o que vira fato, o que vira dimensão, tratamento de histórico, granularidade — que é a lacuna real a ser preenchida.

**Formato de trabalho:** mentoria técnica socrática (ver seção 8).

### 1.1 Conceitos já dominados (calibração de nível)

> Rastreio de conceitos já trabalhados com Guilherme, usado pra calibrar o nível de discussão (seguindo o princípio da seção 8: "não repetir conceito básico já dominado"). Migrado de `AssistBR_Contexto_Mentoria.md` (descontinuado em 2026-07-19) — única informação daquele arquivo sem equivalente em outro documento do projeto.

| Conceito | Status |
|---|---|
| Fato × Dimensão | ✓ Dominado |
| SK × BK × chave composta | ✓ Dominado |
| Dimensão de período | ✓ Dominado |
| OLTP × OLAP, normalizado × dimensional | ✓ Dominado |
| Medallion (Bronze/Silver/Gold) | ✓ Dominado |
| SCD Type 1, 2, 3 | ✓ Dominado |
| Constellation Schema | ✓ Dominado |
| Fato esparsa | ✓ Dominado (motivou a decisão de grão da Fato Interação) |
| Pre-aggregation / regras imutáveis vs. mutáveis | ✓ Dominado |
| Valor esperado vs. realizado | ✓ Dominado |
| Role-playing dimension | ✓ Dominado (aplicado em dim_address) |
| Degenerate dimension | ✓ Dominado (aplicado em ds_address_origin/destination) |
| Bridge table / regra de negócio versionada | ✓ Dominado (aplicado em dim_plan_coverage) |
| Damage-first reasoning para decisão de SCD | ✓ Dominado (argumento de exposição jurídica em dim_plan_coverage) |

**Exercício anterior ao AssistBR:** modelo de auditoria de acessos (fato de acessos + dimensões de usuário, sistema, recurso e período) + SQL analítico (agregações, JOINs, CTEs, window functions). Registrado também em `AssistBR_Atividades_Portfolio.md`, Fase 1.

---

## 2. Escopo de negócio — premissas fixas (não alterar sem validação explícita)

- AssistBR cobre **dois grandes tipos de atendimento**, que não devem ser confundidos:
    1. **Assistência 24h** — emergências, tanto **automotivas** quanto **residenciais** (ex.: cano estourado).
    2. **Seguro residencial agendado** — serviços programados (ex.: limpeza de caixa d'água).
- O negócio **não** é exclusivamente residencial — erro já cometido uma vez nesta mentoria (assumir que "assistência 24h" era só residencial) e corrigido.
- Fontes de dados reais do domínio simulado:
    1. SQL Server operacional (chamados)
    2. SQL Server financeiro (pagamentos) — servidor separado, sem integração nativa
    3. Excel de prestadores (RH) — sem governança, sem padrão, sem lineage
- Métricas de negócio centrais (definidas pelo "diretor" fictício): SLA (>92%), NPS (>75), TMA, TMC, custo por acionamento, reacionamento, margem por plano/região/serviço, produtividade de analista.

---

## 3. Arquitetura técnica

```
SQL Server (Operacional) ─┐
SQL Server (Financeiro)  ─┼→ Bronze → Silver → Gold → BI
Excel (Prestadores)      ─┘
```

|Camada|Responsabilidade|
|---|---|
|Transacional (SSMS)|Escrita transacional, sistemas legados|
|Bronze|Dado bruto, preservado como na fonte|
|Silver|Limpeza, padronização, integração entre fontes, cálculos **imutáveis**|
|Gold|Modelo dimensional (fatos/dimensões), regras **mutáveis**|
|Consumo|Power BI / Databricks SQL|

**Schema dimensional:** Constellation Schema (Galaxy) — 3 fatos compartilhando dimensões, unidas por `bk_attendance`.

```
                    ┌─────────────────┐
                    │  dim_period     │
                    └────────┬────────┘
    ┌────────────────────────┼────────────────────────┐
┌───┴──────────┐    ┌────────┴────────┐    ┌──────────┴───────┐
│ fact_        │    │ fact_           │    │ fact_            │
│ attendance   │    │ interaction     │    │ financial        │
└───┬──────────┘    └────────┬────────┘    └──────────┬───────┘
    └────────────────────────┼────────────────────────┘
              Dimensões compartilhadas:
              client, address, period, service, provider...
```

---

## 4. Convenções e padrões (fixos)

Inglês abreviado + dicionário de dados formal (pendente, seção 9).

|Prefixo|Uso|Exemplo|
|---|---|---|
|`sk_`|Surrogate key|`sk_attendance`|
|`bk_`|Business key|`bk_attendance`|
|`*_id`|FK (sufixo)|`client_id`|
|`id_dt_*`|FK para dim_period|`id_dt_open`|
|`hr_`|Hora (time)|`hr_open`|
|`fl_`|Flag booleana|`fl_accepted`|
|`nr_`|Número/quantidade|`nr_nps`|
|`vl_`|Valor monetário|`vl_expected`|
|`ds_`|Descrição|`ds_address_origin`|
|`nm_`|Nome|—|

**Princípios de design gerais, aplicáveis a qualquer dimensão/fato futura:**

- **Agrupamento/filtro em BI sempre por código (`_cd`/`sk`), nunca por descrição (`_desc`)** — descrições podem se repetir entre entidades diferentes (ex.: nomes de cidade duplicados entre estados).
- **FK opcional nunca fica `NULL` nem recebe string `"N/A"`** — usar linha sentinela (`sk = -1`, descrição "Não aplicável") na dimensão referenciada.
- **Role-playing dimension:** quando o mesmo conceito é usado com papéis diferentes (ex.: endereço de origem/destino), usar **uma única tabela física** com múltiplas FKs, nunca duplicar estrutura.
- **Degenerate dimension:** atributos únicos por linha, sem valor de agrupamento e sem hierarquia própria (ex.: endereço completo rua+número), ficam como texto solto na fato — não geram dimensão nem FK.
- **Evitar redundância entre fato e dimensão** para o mesmo dado de negócio — uma informação deve ter uma única fonte de verdade.
- **Não modelar para cenários não confirmados no levantamento de negócio** (over-engineering) — qualquer extensão de escopo deve ser validada contra a seção 2 antes de virar estrutura.
- **TMA/TMC são calculados na Silver (imutáveis); SLA e regras de negócio mutáveis ficam na Gold.**

---

## 5. Fatos

### 5.1 Fato Atendimento (`fact_attendance`) — FECHADA

Grão: uma linha por serviço prestado dentro de um chamado.

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_attendance`|int|Surrogate key|
|`bk_attendance`|varchar|Business key (`assistência/chamado`, ex. `123456/1`)|
|`analyst_id`|int|FK → dim_analyst|
|`client_id`|int|FK → dim_client|
|`fk_contract`|int|FK → dim_contract (aponta para `sk_contract` — a versão vigente no momento do atendimento, não o contrato em geral)|
|`service_id`|int|FK → dim_service|
|`service_type_id`|int|FK → dim_service_type|
|`modality_id`|int|FK → dim_modality|
|`provider_id`|int|FK → dim_provider|
|`fk_address_origin`|int|FK → dim_address (role-playing)|
|`fk_address_destination`|int|FK → dim_address (role-playing)|
|`ds_address_origin`|varchar|Degenerate dimension: rua+número (origem)|
|`ds_address_destination`|varchar|Degenerate dimension: rua+número (destino)|
|`fl_accepted`|boolean|Prestador aceitou o chamado|
|`id_chnl_accept`|int|FK → dim_channel|
|`id_dt_open` / `hr_open`|int / time|Abertura|
|`id_dt_accept` / `hr_accept`|int / time|Aceite|
|`arrival_estimated`|int|Previsão de chegada (minutos)|
|`id_dt_arrival` / `hr_arrival`|int / time|Chegada|
|`id_dt_conclusion` / `hr_conclusion`|int / time|Conclusão|
|`nr_nps`|int|Nota do cliente (1–5)|
|`nr_contract_attendance_seq`|int|Sequência do atendimento dentro da janela de contagem do contrato (ex.: "4º acionamento do ano"). Calculada na Silver via window function particionada por `fk_contract` — fato imutável, mesmo padrão de TMA/TMC. Alimenta a comparação contra `dim_plan_coverage.limit_value` (tipo `QTD_ANUAL`) na Gold. **Janela confirmada com o diretor (2026-07-19): 12 meses a partir de `dt_start` do contrato (aniversário), não ano-calendário** — window function particiona por `fk_contract` sem necessidade de lógica de data adicional, já que cada renovação gera `fk_contract` novo e reseta a contagem naturalmente|
|`fk_asset_vehicle`|int|FK → `dim_asset_vehicle` (nullable **por design de domínio**, não por dado ausente — só preenchida quando `dim_service_type.category_desc = 'Automotivo'`)|
|`fk_asset_property`|int|FK → `dim_asset_property` (nullable **por design de domínio** — só preenchida quando `dim_service_type.category_desc = 'Residencial'`)|

**Ativo atendido (2026-08-09):** duas FKs nullable por subtipo, discriminadas por `dim_service_type.category_desc` já existente na fato — nunca as duas preenchidas na mesma linha. Alternativa de FK genérica única (`fk_asset` apontando pra uma de duas tabelas) descartada: o motor Tabular do Power BI (VertiPaq/Import Mode) exige relacionamento fixo entre tabelas, não suporta FK polimórfica nativamente. Motivação de negócio: sem granularidade de ativo, é impossível responder "qual veículo específico da frota foi atendido" em contratos PJ com múltiplos ativos (ver 6.3f).

**Métricas (Silver):** TMA = `hr_arrival - hr_accept`; TMC = `hr_conclusion - hr_arrival`; `nr_contract_attendance_seq` = window function particionada por `fk_contract` sobre a janela de contagem vigente.

**Removido do modelo original:** `fl_scheduled` — redundante com `dim_service_type.urgency_desc`; retirado para manter fonte única de verdade.

### 5.2 Fato Interação (`fact_interaction`) — grão fechado, colunas pendentes

- Grão: uma linha por interação de analista/setor no chamado.
- Motivação: um chamado passa por múltiplos setores/analistas (Central → Backoffice → Campo); `analyst_id` sozinho na Fato Atendimento não captura o fluxo.
- Relacionamento: `bk_attendance` → Fato Atendimento.
- Dimensões previstas: `dim_analyst`, `dim_sector` + compartilhadas.
- **Pendente:** detalhar FKs, timestamps, flags por setor.

### 5.3 Fato Financeiro (`fact_financial`) — regras fechadas, colunas pendentes

- Grão: uma linha por lançamento financeiro vinculado a um atendimento.
- `tipo_lancamento`: `CUSTO_PRESTADOR` | `EXCEDENTE_CLIENTE`.
- Conferência automática: `vl_expected`, `vl_realized`, `vl_difference`.
- Margem = receita de mensalidade − `CUSTO_PRESTADOR` + `EXCEDENTE_CLIENTE` (via dimensões compartilhadas).
- Relacionamento: `bk_attendance` → Fato Atendimento.
- **Pendente:** detalhar componentes (mão de obra, km, adicionais).

---

## 6. Dimensões

### 6.1 dim_address — FECHADA

Renomeada de `dim_location` (correção ortográfica).

- **Conceito:** role-playing dimension — referenciada por `fk_address_origin`/`fk_address_destination` na fato, e por FKs de endereço em `dim_client` e `dim_asset`.
- **Granularidade:** macro (bairro/cidade/estado) — não inclui rua/número (isso é degenerate dimension na fato).

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_address`|int|Surrogate key|
|`bk_address`|varchar|Composta: `uf_cd + city_cd + nbhd_cd` (não existe código nacional único de bairro)|
|`nbhd_cd` / `nbhd_desc`|int / varchar|Bairro|
|`city_cd` / `city_desc`|int / varchar|Cidade|
|`uf_cd` / `uf_desc`|int / varchar|Estado|
|`region_cd` / `region_desc`|int / varchar|Reservado para expansão futura (hoje constante)|
|`country_cd` / `country_desc`|int / varchar|Reservado para expansão futura (hoje sempre Brasil)|

- **SCD:** Type 1.
- **Linha sentinela:** `sk_address = -1` ("Não aplicável").
- **Implementação Gold/Power BI:** views `dim_address_origin` e `dim_address_destination` sobre a tabela física única (evita `USERELATIONSHIP` manual em DAX).

### 6.2 dim_service_type — estrutura fechada, SCD pendente

Criada durante o fechamento da Fato Atendimento (não prevista no levantamento original).

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_service_type`|int|Surrogate key|
|`bk_service_type`|varchar|Pendente de definição|
|`category_desc`|varchar|Residencial / Automotivo|
|`urgency_desc`|varchar|Emergencial / Agendado|

Valores conhecidos: Residencial+Emergencial, Residencial+Agendado, Automotivo+Emergencial, Automotivo+Agendado.

### 6.3 dim_client — FECHADA

|Coluna|Tipo|SCD|Descrição|
|---|---|---|---|
|`sk_client`|int|—|Surrogate key|
|`bk_client`|varchar|—|CPF (PF) ou CNPJ (PJ)|
|`person_type_cd`|char|Type 1|PF / PJ — nativo do cadastro na fonte operacional (documentos de cadastro distintos: RG/CPF vs. CNPJ/cartão CNPJ); nunca muda ao longo da vida do cliente|
|`nm_client`|varchar|Type 1|Nome ou razão social|
|`fk_address`|int|Type 2|FK → dim_address, versionado por vigência|
|`dt_address_start` / `dt_address_end`|date|—|Vigência da versão de endereço|
|`phone`, `email`|varchar|Type 1|Sem valor analítico histórico|
|`fl_current`|boolean|—|Flag de versão vigente|

**Múltiplos contratos:** resolvido sem estrutura extra em `dim_client` — é `dim_contract.client_id` permitindo N:1 (um cliente pode ter contrato veicular e residencial simultâneos, cada um em sua própria linha de `dim_contract`).

### 6.3a dim_contract — FECHADA (estrutura)

|Coluna|Tipo|SCD|Descrição|
|---|---|---|---|
|`sk_contract`|int|—|Surrogate key|
|`bk_contract`|varchar|—|Número único do contrato (fonte operacional)|
|`client_id`|int|—|FK → dim_client|
|`fk_plan`|int|—|FK → dim_plan|
|`dt_start` / `dt_end`|date|—|Vigência do contrato|
|`status_cd`|int|Type 2|Ativo / Cancelado / Suspenso|
|`fk_status_reason`|int|Type 1|FK → dim_status_reason|
|`dt_status_start` / `dt_status_end`|date|—|Vigência da versão de status|
|`fl_current`|boolean|—|Flag de versão vigente|
|`person_type_cd`|char|Type 1|Replicado de `dim_client` — denormalização proposital (baixa cardinalidade, alta frequência de filtro em BI, evita join obrigatório pra saber PF/PJ)|
|`fk_previous_contract`|int|Não versionado|FK autorreferenciada → `dim_contract.sk_contract` do contrato anterior, quando há renovação. **Inferida na Silver via heurística** (mesmo `bk_client` + `dt_end` do anterior próximo do `dt_start` do novo) — fonte operacional confirmou que **não existe** esse vínculo gravado (ver pendência resolvida abaixo). Nullable — vazio quando não há candidato a contrato anterior|
|`fl_previous_contract_inferred`|boolean|—|Sempre `true` quando `fk_previous_contract` está preenchido — marca explicitamente que o vínculo é heurística, não fato confirmado pela fonte. Mesmo princípio já usado em `vl_expected` vs. `vl_realized`: dado inferido nunca se disfarça de dado confirmado|
|`vl_monthly_fee_negotiated`|decimal|Não versionado|Valor de mensalidade negociado, aplicável apenas a contratos PJ com desconto por volume. Nullable — quando vazio, o valor efetivo vem de `dim_plan.vl_monthly_fee` (preço de tabela). Não versionado porque renovação já vira contrato novo (confirmado com o diretor) — renegociação dentro do mesmo contrato não é cenário confirmado|

**Renovação de contrato:** gera **nova linha** (novo `bk_contract`), nunca nova versão Type 2 da linha anterior — troca de plano na renovação não pode reescrever histórico. **Confirmado com o diretor (2026-07-19):** renovação sempre gera número de contrato novo; **não existe vínculo gravado no sistema operacional** entre contrato antigo e novo — hoje a empresa nem sabe de forma confiável há quanto tempo um cliente está na base. `fk_previous_contract` fica como campo inferido (ver coluna acima), nunca como fato confirmado.

**Premissa assumida (2026-07-19, validar se surgir exceção):** todo contrato tem duração de 1 ano — nenhum plano ou vigência mencionado no levantamento ultrapassa esse período. Isso sustenta `nr_contract_attendance_seq` particionar só por `fk_contract` sem precisar de janela móvel dentro do mesmo contrato. Se aparecer um contrato de duração maior, essa premissa precisa ser revisada.

**Elo com `fact_attendance`:** `fk_contract` na fato (ver seção 5.1) — o contrato usado já é conhecido no momento da abertura do atendimento (não precisa ser inferido depois), e aponta para o `sk_contract` vigente naquele momento, preservando o estado histórico do contrato mesmo que ele mude de status depois.

### 6.3b dim_plan — FECHADA (estrutura)

Separada de `dim_contract` por ter alto reuso (milhares de contratos compartilham o mesmo plano) — mudar uma regra do plano não deve exigir `UPDATE` em massa em `dim_contract`.

|Coluna|Tipo|SCD|Descrição|
|---|---|---|---|
|`sk_plan`|int|—|Surrogate key|
|`bk_plan`|varchar|—|Código do plano no sistema operacional (BK real — "plano contratado" já é referenciado lá, conforme levantamento seção 7)|
|`plan_desc`|varchar|Type 1|Basic / Premium|
|`nr_min_confirmation_sla`|int|Type 2|Prazo máximo (minutos) pra confirmação do prestador — Premium: 15, Basic: 30 (levantamento seção 4.1)|
|`nr_min_arrival_sla`|int|Type 2|Prazo máximo (minutos) pra início do atendimento — Premium: 60, Basic: 120|
|`vl_monthly_fee`|decimal|Type 2|Preço de tabela do plano. Aplicado diretamente pra contratos PF (sem negociação); pra PJ, funciona como referência quando não há valor negociado em `dim_contract.vl_monthly_fee_negotiated`. Reajustado anualmente (confirmado com o diretor) — Type 2 preserva o valor vigente em cada período|
|`dt_start` / `dt_end`|date|—|Vigência da versão|
|`fl_current`|boolean|—|Flag de versão vigente|

**Decisão de design (2026-07-19):** SLA modelado como **colunas Type 2 dentro da própria `dim_plan`**, não como dimensão/bridge separada. Teste aplicado: SLA não tem reuso cruzado nem grão próprio (não varia por tipo de serviço, por região, etc.) — é 1:1 com o plano, exatamente como `status_cd` é 1:1 com `dim_contract`. Não há relação N:N que justificasse bridge (diferente de `dim_plan_coverage`, onde a N:N plano×tipo de serviço é real). SCD Type 2 pelo mesmo damage-first reasoning já aplicado em outras regras mutáveis: se o board apertar o SLA do Premium, atendimentos antigos precisam continuar avaliados contra o valor vigente na época.

**Mensalidade — resolvida (2026-07-19), confirmado com o diretor:** PF paga preço de tabela fixo por plano, sem negociação; PJ negocia desconto por volume, registrado no próprio contrato; reajuste anual (geralmente janeiro) vale pra ambos, mas em PJ pode ser renegociado, não é automático. Modelagem: `vl_monthly_fee` (preço de tabela, Type 2) fica em `dim_plan`; valor negociado fica em `dim_contract.vl_monthly_fee_negotiated` (não versionado, nullable). **Valor efetivo calculado na Gold:** `COALESCE(dim_contract.vl_monthly_fee_negotiated, dim_plan.vl_monthly_fee)` — regra mutável que combina dois valores, mesmo padrão de regra-na-Gold já usado em SLA e cobertura. Reajuste percentual (se necessário) é **derivável** comparando duas versões consecutivas de `vl_monthly_fee` — não precisa de dimensão própria pra guardar percentual.

### 6.3c dim_plan_coverage — bridge versionada, colunas finais pendentes

Bridge entre `dim_plan` e `dim_service_type`, carregando o limite de cobertura por combinação. **SCD Type 2** — regra mutável que precisa de vigência histórica, para não gerar cobrança retroativa indevida (risco jurídico/disputa, não só técnico) quando o board alterar um limite.

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_plan_coverage`|int|Surrogate key|
|`fk_plan`|int|FK → dim_plan|
|`fk_service_type`|int|FK → dim_service_type|
|`limit_type_cd`|varchar|KM / VALOR / QTD_ANUAL — genérico, estrutura final (ver decisão abaixo: diferença fica no cálculo, não no schema)|
|`limit_value`|decimal|Valor do limite|
|`dt_start` / `dt_end`|date|Vigência da versão|
|`fl_current`|boolean|Flag de versão vigente|

**Cálculo de excedente na Gold:** join por `plan_id` + `service_type_id` (ambos já alcançáveis via `dim_contract` e direto na fato), filtrando pela versão vigente na data do atendimento — join por range de data, não chave simples.

**Dois padrões de cálculo distintos, mesma estrutura de regra (decisão de 2026-07-19):**
- **`KM` / `VALOR`:** comparação linha a linha — o valor do próprio atendimento (km rodado, valor gasto, futuramente em `fact_financial`) contra `limit_value`. Cálculo acontece **na Gold** (regra mutável, mesmo princípio já fixado pra SLA).
- **`QTD_ANUAL`:** exige contagem acumulada de atendimentos do contrato numa janela de tempo — não é comparação local. A contagem em si ("quantos acionamentos esse contrato já teve") é fato imutável (mesma natureza de TMA/TMC) e é calculada na Silver, armazenada como `fact_attendance.nr_contract_attendance_seq` (ver seção 5.1); a **comparação** contra o limite (que é mutável) continua na Gold.
- **Não mover o cálculo de excedente inteiro pra Silver:** avaliado e descartado — se o board corrigir um limite errado em `dim_plan_coverage`, um cálculo feito na Silver exigiria reprocessar a partição inteira; feito na Gold, corrige-se uma linha na dimensão e toda leitura futura já reflete certo. Seguindo a mesma lógica já fixada pra TMA/TMC × SLA (seção 7).

**Janela de contagem confirmada com o diretor (2026-07-19):** 12 meses a partir de `dt_start` do contrato (aniversário), não ano-calendário — e reseta a cada renovação, já que renovação é sempre contrato novo (`fk_contract` diferente). Como a contagem já particiona por `fk_contract`, o reset acontece naturalmente sem lógica extra.

### 6.3d dim_authorized_contact — FECHADA

Pessoa física autorizada a solicitar atendimento em nome de um contrato PJ ("responsável PJ"). Relação **N:N com `dim_contract`** via tabela ponte **versionada** — autorização é por **contrato** (não por ativo específico da frota; granularidade confirmada contra o levantamento de negócio, evitando over-engineering).

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_authorized_contact`|int|Surrogate key|
|`nm_contact`|varchar|Nome|
|`doc_contact`|varchar|CPF do responsável|

**Bridge `bridge_contract_authorized_contact` (versionada — SCD Type 2 aplicada à própria bridge):**

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_contract_authorized_contact`|int|Surrogate key da versão de autorização|
|`fk_contract`|int|FK → dim_contract|
|`fk_authorized_contact`|int|FK → dim_authorized_contact|
|`dt_start` / `dt_end`|date|Vigência da autorização|
|`fl_current`|boolean|Flag de autorização vigente|

**Decisão de design (2026-07-19):** funcionário autorizado pode ser desautorizado sem o contrato encerrar (levantamento seção 8, mesmo padrão já confirmado pra ativos entrando/saindo do contrato). Sem vigência, uma auditoria de atendimento antigo não teria como responder "quem estava autorizado a acionar naquela data" — mesmo damage-first reasoning já aplicado em `dim_plan_coverage` e no SLA de `dim_plan`. A vigência fica **na bridge**, não em `dim_authorized_contact`: o dado que muda no tempo é a **relação** (quem está autorizado em qual contrato), não os atributos da pessoa em si (nome/CPF não mudam por autorização) — por isso `dim_authorized_contact` continua Type 1.

### 6.3e dim_status_reason — FECHADA

Motivo de mudança de status do contrato (Inadimplência, Solicitação do cliente, Suspeita de fraude, Renegociação, Outro). **SCD Type 1** — o rótulo em si não tem histórico próprio; quem versiona é o vínculo em `dim_contract`.

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_status_reason`|int|Surrogate key|
|`reason_cd`|int|Código do motivo — uso em filtro/agrupamento (princípio: sempre por código, nunca por descrição); sentinela 99 = "Outro"|
|`reason_desc`|varchar|Motivo padronizado (texto)|
|`ds_status_reason_detail`|varchar|Campo livre complementar, preenchido **somente** quando `reason_cd = 99` — evita travar processo operacional em exceções não previstas no catálogo|

**Correção de consistência (2026-07-19):** `ds_status_reason_detail` estava descrito na prosa da versão anterior desta seção, mas ausente da tabela de colunas — corrigido. `reason_cd` adicionado por consistência com o princípio já fixado na seção 4 (filtro sempre por código).

**Governança do catálogo:** separar quem *usa* (Backoffice, no dia a dia) de quem *governa* (decide categoria nova) — não sobrecarregar o Backoffice com manutenção de taxonomia. Monitorar % de uso do código "Outro" como sinal de catálogo desatualizado.

### 6.3f dim_asset — FECHADA (estrutura)

Veículos e imóveis cobertos por contrato. **Duas tabelas físicas separadas** (`dim_asset_vehicle`, `dim_asset_property`), sem supertype/subtype comum — decisão baseada no mesmo teste de reuso/grão já aplicado em `dim_plan` vs. `dim_plan_coverage`, só que na direção oposta: aqui, a **ausência** de necessidade de consulta analítica unificada entre os dois segmentos (dinâmica de custo estruturalmente incomparável entre auto e residencial — ex.: precificação majoritariamente tabelada num segmento, dinâmica de sinistro diferente no outro) justifica não unificar. Modelo supertype/subtype (tabela núcleo + satélites) avaliado e descartado pelo mesmo motivo.

**Cardinalidade com `dim_contract`:** PF = 1:1 (um contrato cobre um ativo); PJ = 1:N (um contrato pode cobrir uma frota de veículos). Relação resolvida via **bridge versionada**, não FK direta na dimensão do ativo — ativos podem entrar/sair da frota sem o contrato encerrar (mesmo padrão de negócio já citado na decisão de `dim_authorized_contact`: "mesmo padrão já confirmado pra ativos entrando/saindo do contrato"). FK direta e fixa (`dim_asset_vehicle.fk_contract`) quebraria a capacidade de responder "quais veículos estavam cobertos pelo contrato em uma data passada" após qualquer mudança na frota.

**`dim_asset_vehicle`:**

|Coluna|Tipo|SCD|Descrição|
|---|---|---|---|
|`sk_asset_vehicle`|int|—|Surrogate key (versionada — múltiplas linhas por veículo ao longo do tempo)|
|`bk_asset_vehicle`|varchar|—|Chassi — identificador imutável do veículo, mesmo critério de imutabilidade já usado em `dim_client.bk_client` (CPF/CNPJ). Preferido sobre placa, que pode mudar (transferência de propriedade, remarcação Mercosul)|
|`plate`|varchar|**Type 2**|Placa do veículo. Versionada por vigência — placa tem peso jurídico/legal (fiscalização, disputa de sinistro, identificação formal), exige reconstituição histórica: "qual era a placa registrada na data do atendimento X"|
|`model_desc`|varchar|Type 1|Modelo — atributo de fábrica, não muda ao longo da vida do veículo|
|`nr_year`|int|Type 1|Ano de fabricação — atributo de fábrica, não muda|
|`color_desc`|varchar|Type 1|Cor. Pode mudar (repintura), mas classificada Type 1: baixo valor de reconstituição histórica — é identificação visual operacional no momento do atendimento, não atributo com peso de auditoria/jurídico como a placa|
|`fk_address`|int|Type 2|FK → dim_address (endereço de guarda/domicílio do veículo, quando aplicável)|
|`dt_start` / `dt_end`|date|—|Vigência da versão|
|`fl_current`|boolean|—|Flag de versão vigente|

**`dim_asset_property`:**

|Coluna|Tipo|SCD|Descrição|
|---|---|---|---|
|`sk_asset_property`|int|—|Surrogate key (versionada)|
|`bk_asset_property`|varchar|—|**Pendente de definição — pendência de negócio.** Endereço sozinho não é suficiente/exclusivo como BK (dois imóveis podem compartilhar endereço — ex.: apartamentos no mesmo prédio, casa principal + casa de veraneio no mesmo lote). A validar com o diretor: existe matrícula do imóvel, número de IPTU/cadastro municipal, ou outro identificador único capturado pela fonte operacional?|
|`property_type_desc`|varchar|**Type 2**|Tipo de imóvel (casa/apartamento/etc.). Pode mudar por reclassificação cadastral; tratado com o mesmo damage-first reasoning de outras regras mutáveis do projeto|
|`nr_area_m2`|decimal|**Type 1 (provisório)**|Metragem do imóvel. Classificação condicionada a pendência de negócio: se metragem influenciar precificação ou limite de cobertura em `dim_plan_coverage` no futuro, reclassificar para Type 2. Hoje `dim_plan_coverage` não tem nenhuma regra ligada a metragem — Type 1 é a classificação coerente com o que está confirmado|
|`fk_address`|int|**Type 2**|FK → dim_address, versionado por vigência — mesmo padrão já usado em `dim_client.fk_address` ("análises por região dependem do endereço no momento do evento")|
|`dt_start` / `dt_end`|date|—|Vigência da versão|
|`fl_current`|boolean|—|Flag de versão vigente|

**Bridges (versionadas — SCD Type 2 aplicada à própria bridge, mesmo padrão de `bridge_contract_authorized_contact`):**

Duas bridges separadas, uma por subtipo — consistente com a decisão de tabelas físicas separadas, e pelo mesmo motivo já usado pra decidir `fk_asset_vehicle`/`fk_asset_property` como FKs separadas na fato: a bridge também é consumida pelo Power BI e está sujeita à mesma restrição de relacionamento fixo do motor Tabular. Não existe motivo técnico pra bridge se comportar diferente da fato nessa decisão.

`bridge_contract_asset_vehicle`:

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_contract_asset_vehicle`|int|Surrogate key da versão do vínculo|
|`fk_contract`|int|FK → dim_contract|
|`fk_asset_vehicle`|int|FK → dim_asset_vehicle|
|`dt_start` / `dt_end`|date|Vigência do vínculo (ativo entrando/saindo da frota do contrato)|
|`fl_current`|boolean|Flag de vínculo vigente|

`bridge_contract_asset_property`: mesma estrutura, trocando `fk_asset_vehicle` por `fk_asset_property`.

**Pendências de negócio registradas (2026-08-09), a validar com o diretor:**
1. BK de `dim_asset_property` — identificador único do imóvel na fonte operacional (matrícula, IPTU, ou equivalente).
2. Se metragem do imóvel influencia precificação ou limite de cobertura — impacta classificação SCD de `nr_area_m2` e, potencialmente, `dim_plan_coverage`.
3. (Registrada anteriormente, ainda sem definição) Se planos/coberturas diferenciados por ativo dentro de um mesmo contrato de frota PJ são um cenário real (ex.: carro popular vs. executivo vs. carga com coberturas distintas) — hipótese levantada durante a modelagem, **não confirmada**, não modelada.

### 6.3g dim_provider — estrutura em fechamento (2026-08-12)

Prestador credenciado (guincho, chaveiro, encanador, eletricista etc.). Grão: uma linha por prestador (pessoa jurídica) — especialidade e preço **não** são atributos da dimensão, são relação (bridge), pois um mesmo prestador pode atender múltiplas categorias de serviço.

|Coluna|Tipo|SCD|Descrição|
|---|---|---|---|
|`sk_provider`|int|—|Surrogate key (versionada)|
|`bk_provider`|varchar|—|CNPJ. **Premissa de negócio assumida, não confirmada pela fonte:** prestador é sempre PJ (nunca autônomo PF) — mitigação de risco de vínculo trabalhista, a validar com o diretor|
|`fk_address`|int|Type 2|FK → dim_address — endereço de sede/domicílio do prestador (não confundir com área de cobertura, ver pendência abaixo). Mesmo padrão de `dim_client.fk_address`: permite comparar `fk_address_origin` do atendimento com a sede do prestador (atendeu dentro ou fora da própria região)|
|`status_cd`|int|Type 2|Ativo / Inativo / Suspenso — mesmo padrão de `dim_contract.status_cd`|
|`fk_provider_status_reason`|int|Type 1|FK → `dim_provider_status_reason` (nova mini-dimensão, ver decisão abaixo)|
|`dt_start` / `dt_end`|date|—|Vigência da versão|
|`fl_current`|boolean|—|Flag de versão vigente|

**`dim_provider_status_reason` (nova mini-dimensão, não reaproveita `dim_status_reason`):** mesma estrutura de `dim_status_reason` (`sk_provider_status_reason`, `reason_cd`, `reason_desc`, sentinela), porém catálogo **próprio e independente**. Decisão testada com o mesmo critério já usado para rejeitar supertype/subtype em `dim_asset`: existe necessidade analítica de consultar motivo de status de contrato e motivo de descredenciamento de prestador na mesma lista? Não — são domínios de negócio sem sobreposição de significado (contrato é governado por cobrança/operações; prestador por gestão de fornecedores), só coincidem em formato (código+descrição+sentinela). Reuso via role-playing foi descartado; alternativa de coluna discriminadora (`domain_cd`) na mesma tabela também descartada — acoplaria governança de dois catálogos independentes por uma economia de estrutura mínima (tabelas pequenas).

**`bridge_provider_service_type` (N:N versionada, SCD Type 2 na bridge):**

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_provider_service_type`|int|Surrogate key da versão|
|`fk_provider`|int|FK → dim_provider|
|`fk_service_type`|int|FK → dim_service_type|
|`vl_price`|decimal|Preço-padrão do prestador para aquela categoria de serviço|
|`dt_start` / `dt_end`|date|Vigência|
|`fl_current`|boolean|Flag de versão vigente|

Mesmo padrão de `dim_plan_coverage`: N:N real (prestador atende múltiplas especialidades; especialidade tem múltiplos prestadores credenciados), relação muda no tempo sem que `dim_provider` mude (credenciamento/descredenciamento), e o mesmo damage-first reasoning (auditoria de atendimento antigo precisa responder "ele estava credenciado e a que preço, naquela data").

**`bridge_provider_service_region_price` (exceção negociada, N:N versionada — estrutura definida, aguardando confirmação de existência do cenário):**

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_provider_service_region_price`|int|Surrogate key da versão|
|`fk_provider`|int|FK → dim_provider|
|`fk_service_type`|int|FK → dim_service_type|
|`fk_address`|int|FK → dim_address (região da exceção)|
|`vl_price`|decimal|Preço negociado para aquela combinação específica|
|`dt_start` / `dt_end`|date|Vigência|
|`fl_current`|boolean|Flag de versão vigente|

**Motivação:** discussão levantou um risco de negócio real — se o preço fora da área de atuação padrão do prestador fosse negociado "na hora", atendimento a atendimento (ex.: acionamentos repetidos numa enchente), abriria brecha para inflar custo sem controle/auditoria. Resolvido como **acordo pré-negociado e versionado**, nunca valor decidido no momento do atendimento — mesmo princípio já usado em `dim_contract.vl_monthly_fee_negotiated`, só que aqui a chave da exceção é a combinação prestador×serviço×região, não o contrato. **Valor efetivo na Gold:** `COALESCE(bridge_provider_service_region_price.vl_price, bridge_provider_service_type.vl_price)` — mesmo padrão de cascata específico→genérico já usado na mensalidade PF/PJ.

**Pendências de negócio registradas (2026-08-12), para próxima entrevista com o diretor:**
1. Confirmar premissa "prestador sempre PJ".
2. **Área de atuação/cobertura regional do prestador** (região onde está formalmente credenciado a atender, distinta do endereço de sede) — decisão de **não modelar agora**; registrar a pergunta certa para o diretor: isso é uma métrica de negócio real (ex.: identificar gaps de cobertura de rede por região) ou é regra operacional de despacho que não precisa virar dimensão no Gold? Se confirmado como métrica, vira bridge `bridge_provider_coverage_area` (mesmo padrão de bridge com `dim_address`).
3. Confirmar se o cenário de preço negociado por região (acima) de fato ocorre no negócio.

**Débito técnico registrado (não é decisão de modelagem do Gold):** o Excel de prestadores (fonte sem governança, sem padrão — ver seção 2) provavelmente traz endereço de prestador em texto livre e inconsistente. Antes de resolver a FK para `dim_address`, a Silver precisa de uma tabela de referência/lookup (CEP → bairro/cidade/UF) para normalizar o dado de entrada. Isso é responsabilidade de data quality/master data na camada de padronização, não gera nova dimensão no Gold — `dim_address` já resolve a geografia de forma denormalizada (decisão da seção 6.1, mantida; snowflaking em `dim_uf`/`dim_city`/`dim_cep` separadas foi cogitado e descartado pelo mesmo motivo já usado para justificar a tabela física única de `dim_address`).

### 6.4 dim_period — atributos previstos, não fechada formalmente

|Coluna|Descrição|
|---|---|
|`period_id`|SK|
|`dt_date`|Data|
|`nr_day_of_week`|Dia da semana|
|`nr_day`|Dia do mês|
|`nr_month`|Mês|
|`nr_year`|Ano|
|`fl_holiday`|Feriado|
|`fl_weekend`|Fim de semana|

### 6.5 Demais dimensões — não iniciadas

`dim_analyst`, `dim_service`, `dim_modality`, `dim_channel`, `dim_sector`. (`dim_provider` em fechamento — ver 6.3g.)

### 6.6 SCD — resumo por entidade (referência rápida)

> Tabela atualizada após o fechamento da árvore `dim_contract` (2026-07-05/06). A versão anterior tratava "Contrato/Plano" como um bloco único Type 2 (plano, ativos, limites, responsáveis, status juntos) — essa era uma intenção inicial registrada na Referência, já substituída pelo desenho granular abaixo, onde cada pedaço tem SCD próprio.

|Entidade / atributo|Tipo SCD|Motivo|
|---|---|---|
|`dim_contract.status_cd`|Type 2|Auditoria e disputas — histórico de quando o contrato esteve ativo/suspenso/cancelado precisa ser preservado|
|`dim_contract.fk_status_reason`|Type 1|Catálogo de motivos não versiona; quem versiona é o vínculo em dim_contract|
|`dim_contract.person_type_cd`|Type 1|Replicado de dim_client, atributo nativo que não muda|
|`dim_contract.fk_plan`|Não versionado|Troca de plano no meio do contrato não é cenário confirmado no levantamento; hoje modelado como renovação = contrato novo (novo `bk_contract`)|
|`dim_plan.plan_desc`|Type 1|Basic/Premium não muda de nome|
|`dim_plan.nr_min_confirmation_sla` / `nr_min_arrival_sla`|Type 2|SLA é 1:1 com o plano (sem N:N que justifique bridge); regra mutável — histórico precisa ser preservado se o board apertar o prazo|
|`dim_plan_coverage` (bridge inteira)|Type 2|Regra de cobertura é mutável; sem vigência histórica gera risco de cobrança retroativa indevida e disputa jurídica|
|`bridge_contract_authorized_contact`|Type 2 (bridge inteira)|Funcionário pode ser desautorizado sem o contrato encerrar; sem vigência, auditoria não sabe quem estava autorizado numa data passada|
|`dim_authorized_contact` (nome, CPF)|Type 1|Atributos da pessoa não mudam por autorização — quem versiona é a relação (bridge), não a pessoa|
|`dim_status_reason`|Type 1|Catálogo não versiona; mini-dimensão, mudança de status é versionada em `dim_contract`, não aqui|
|`dim_client` — endereço|Type 2|Análises por região dependem do endereço no momento do evento|
|`dim_client` — `person_type_cd`, telefone, e-mail|Type 1|Sem valor analítico histórico / atributo nativo estável|
|`dim_address`|—|Tudo Type 1 (completo)|
|`dim_provider.status_cd`|Type 2|Auditoria — mesmo padrão de `dim_contract.status_cd`|
|`dim_provider.fk_provider_status_reason`|Type 1|Catálogo próprio (`dim_provider_status_reason`), não versiona; quem versiona é o vínculo em dim_provider|
|`dim_provider.fk_address`|Type 2|Endereço de sede — mesmo padrão de `dim_client.fk_address`|
|`bridge_provider_service_type` / `bridge_provider_service_region_price` (bridge inteira)|Type 2|Preço e credenciamento mudam no tempo sem alterar o cadastro do prestador; mesmo damage-first reasoning de `dim_plan_coverage`|
|`dim_asset_vehicle.plate`|Type 2|Placa tem peso jurídico/legal (fiscalização, disputa de sinistro) — exige reconstituição histórica|
|`dim_asset_vehicle.model_desc` / `nr_year`|Type 1|Atributos de fábrica, não mudam|
|`dim_asset_vehicle.color_desc`|Type 1|Pode mudar (repintura), mas baixo valor de reconstituição histórica — identificação visual operacional, sem peso jurídico como a placa|
|`dim_asset_property.property_type_desc`|Type 2|Pode mudar por reclassificação cadastral; regra mutável|
|`dim_asset_property.nr_area_m2`|Type 1 (provisório)|Sem regra de precificação/cobertura ligada a metragem hoje confirmada — reclassificar se pendência de negócio confirmar o contrário|
|`dim_asset_property.fk_address`|Type 2|Mesmo padrão de `dim_client.fk_address` — análises por região dependem do endereço no momento do evento|
|`bridge_contract_asset_vehicle` / `bridge_contract_asset_property` (bridge inteira)|Type 2|Ativo pode entrar/sair da frota do contrato sem o contrato encerrar; sem vigência, auditoria não sabe quais ativos estavam cobertos numa data passada|

### 6.7 Mecânica técnica de SCD Type 2 (padrão para qualquer dimensão)

- `sk_*`: única por linha/versão — é o que a FK da fato referencia.
- `bk_*`: repete-se entre versões do mesmo registro (ex.: CPF) — usada só para dedup no ETL.
- `dt_inicio_vigencia` / `dt_fim_vigencia`: necessárias para consultas de estado histórico.
- `fl_current`: flag complementar (não substitui as datas) — otimiza consultas de estado atual.

---

## 7. Regras de transformação por camada

|Regra|Tipo|Camada|
|---|---|---|
|TMA = chegada − aceite|Imutável|Silver|
|TMC = conclusão − chegada|Imutável|Silver|
|SLA por plano|Mutável|Gold|
|Cobertura por serviço|Mutável|Gold|
|Valor pago ao prestador|Mutável|Gold|
|Padronização de formatos|—|Silver|
|Deduplicação e integração entre fontes|—|Silver|
|`vl_expected` vs `vl_realized`|—|Silver/Gold|

---

## 8. Como conduzir a mentoria a partir deste documento

- **Relação:** Tech Lead acompanhando o crescimento de um engenheiro do próprio time — não professor/aluno, não avaliador. Sucesso medido pela evolução técnica de Guilherme, não pela quantidade de respostas certas dadas.
- **Postura adaptativa por contexto:**
    - Conceito novo → ensina como professor: claro, progressivo, com exemplo real.
    - Discussão de arquitetura/modelagem → questiona premissas, apresenta alternativas, compara trade-offs, não entrega resposta pronta de cara.
    - Implementação → age como Tech Lead: destrava, revisa raciocínio, complementa.
- **Erro do aprendiz:** antes de corrigir, identificar a premissa equivocada por trás da conclusão errada. O objetivo é consertar o raciocínio, não só a resposta.
- **Nível de discussão:** escalar naturalmente conforme o domínio demonstrado — não repetir conceito básico já dominado.
- **Trade-offs:** sempre que houver mais de uma solução válida, comparar manutenção, performance, escalabilidade, governança, observabilidade, custo operacional e maturidade do time que vai manter — sem apresentar decisão de contexto como verdade absoluta.
- **Discordância:** não concordar automaticamente. Se a ideia for boa, explicar por quê. Se tiver problema, apontar com argumento técnico e alternativa.
- **Documento como referência, não como prisão:** os documentos do projeto (`PROJECT_CONTEXT.md`, Referência, Levantamento) são memória compartilhada — mantêm consistência, mas não travam uma solução melhor se ela surgir na discussão.
- **Proatividade:** trazer riscos, oportunidades, débitos técnicos e boas práticas de mercado sem esperar pergunta direta — inclusive sugestões de como aproximar o projeto do que uma empresa madura faria.
- **Fechamento de decisões importantes:** ao final de discussões relevantes, sinalizar quando algo deveria virar atualização no documento canônico (changelog/pendências), explicando brevemente o motivo.
- **Tom:** conversa de sessão de arquitetura entre dois engenheiros num quadro branco — não checklist, não manual, não formato de exame.

---

## 9. Pendências e próximos passos

|Item|Status||
|---|---|---|
|`dim_client`|✅ FECHADA||
|`dim_contract`|✅ FECHADA (estrutura)||
|`dim_plan`|✅ FECHADA (estrutura) — SLA e mensalidade (`vl_monthly_fee`) como colunas Type 2; negociação PJ via `dim_contract.vl_monthly_fee_negotiated`||
|`dim_plan_coverage`|✅ FECHADA (estrutura) — `limit_type_cd` genérico comporta os 3 tipos; diferença fica no cálculo (Gold)||
|`dim_authorized_contact`|✅ FECHADA — bridge versionada (SCD Type 2), pessoa Type 1||
|`dim_status_reason`|✅ FECHADA — `reason_cd`/`reason_desc` + `ds_status_reason_detail` (sentinela)||
|Elo `fact_attendance` → `dim_contract`|✅ Resolvido — `fk_contract` adicionado à fato||
|Elo `fact_attendance` → `dim_asset`|✅ Resolvido — `fk_asset_vehicle`/`fk_asset_property` adicionados à fato, nullable por design de domínio, discriminados por `dim_service_type.category_desc`||
|`dim_service_type`|🟡 Falta definir SCD e `bk_service_type`||
|`dim_period`|🟡 Falta fechamento formal||
|`dim_asset`|✅ FECHADA (estrutura) — duas tabelas físicas (`dim_asset_vehicle`, `dim_asset_property`) + bridges versionadas; BK do imóvel e impacto de metragem pendentes de validação com o diretor||
|`dim_analyst`|🔴 Não iniciada||
|`dim_service`|🔴 Não iniciada||
|`dim_modality`|🔴 Não iniciada||
|`dim_provider`|🟡 Estrutura em fechamento (2026-08-12) — cadastro + `bridge_provider_service_type` + `bridge_provider_service_region_price` definidos; pendências de negócio (premissa PJ, cobertura regional) para próxima entrevista||
|`dim_channel`|🔴 Não iniciada||
|`dim_sector`|🔴 Não iniciada||
|**Fato Interação**|🟡 Falta detalhar colunas||
|**Fato Financeiro**|🟡 Falta detalhar componentes||
|Diagrama ER|🟡 Primeira versão produzida (DBML), atualizada em 19/07 com `fk_contract`|
|Dicionário de Dados|🔴 Não iniciado|
|**Pendências de negócio — todas resolvidas em 2026-07-19 (entrevista com o diretor).** Ver changelog e seções 5.1, 6.3a, 6.3b, 6.3c para o desenho resultante de cada uma: (1) renovação de contrato não tem vínculo na fonte → `fk_previous_contract` inferido + `fl_previous_contract_inferred`; (2) janela do `QTD_ANUAL` é aniversário do contrato, não ano-calendário; (3) mensalidade fixa por plano (PF) ou negociada por contrato (PJ), reajuste anual → `dim_plan.vl_monthly_fee` (Type 2) + `dim_contract.vl_monthly_fee_negotiated` (override nullable).|

---

## Changelog de decisões arquiteturais

> Adicionar uma linha aqui sempre que uma decisão relevante for tomada, além de atualizar a seção correspondente acima.

| Data       | Decisão                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 2026-08-12 | `dim_provider` aberta, estrutura em fechamento (não fechada — 2 pendências de negócio abertas). Grão: uma linha por prestador (PJ); especialidade e preço resolvidos como relação, não atributo — `bridge_provider_service_type` (N:N versionada prestador×tipo de serviço, `vl_price`, SCD Type 2, mesmo padrão de `dim_plan_coverage`). Exceção de preço negociado por região resolvida como **acordo pré-negociado e versionado** (`bridge_provider_service_region_price`, chave prestador×serviço×região via `dim_address`), não valor decidido ad-hoc por atendimento — motivado por risco de custo inflado sem controle (ex.: acionamentos repetidos fora da área numa enchente); valor efetivo via `COALESCE` na Gold, mesmo padrão da mensalidade PF/PJ. Endereço de sede do prestador via `fk_address` (Type 2), reaproveitando `dim_address`. Status do prestador (`status_cd`, Type 2) segue padrão de `dim_contract.status_cd`; motivo de status vira **nova mini-dimensão independente** `dim_provider_status_reason` — avaliado e descartado reaproveitar `dim_status_reason` via role-playing ou coluna discriminadora de domínio, porque os catálogos não têm sobreposição de significado nem de governança (times diferentes), mesmo teste de reuso/grão já usado para separar `dim_asset_vehicle`/`dim_asset_property`. Cobertura regional (área de atuação) **não modelada agora** — vira pendência de negócio (é métrica real ou regra de despacho?). Premissa "prestador sempre PJ" assumida, não confirmada. Débito técnico registrado: normalização de CEP/endereço do Excel de prestadores fica na Silver (data quality), não gera nova dimensão de geografia — cogitado e descartado snowflaking de `dim_address` em tabelas separadas de UF/cidade/CEP, pelo mesmo motivo já usado para justificar a tabela física única. Propagado para `AssistBR_Modelagem_Tecnica.md`, `Diagrama ER.md` e `AssistBR_Atividades_Portfolio.md`. |
| 2026-07-05 | Fato Atendimento fechada; `dim_address` (ex-`dim_location`) fechada com SCD Type 1, role-playing, degenerate dimension e linha sentinela; `dim_service_type` criada; `fl_scheduled` removido da fato; mecânica de SCD Type 2 documentada via `dim_client`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| 2026-07-05 | dim_client fechada (PF/PJ nativo via person_type_cd, Type 1; endereço Type 2 já existente; múltiplos contratos resolvidos via N:1 em dim_contract, sem estrutura extra). dim_contract aberta e fechada: grão 1 linha/contrato, status_cd versionado Type 2, fk_status_reason Type 1, person_type_cd replicado de dim_client (denormalização proposital, valor estável/alta frequência de filtro). dim_plan criada, separada de contrato (alta cardinalidade de reuso). dim_plan_coverage criada: bridge plano×service_type×limite, SCD Type 2 (regra mutável precisa de vigência histórica para não gerar cobrança retroativa indevida). dim_authorized_contact criada: N:N com contrato via bridge, autorização por contrato (não por ativo — nível de granularidade confirmado contra levantamento, evitando over-engineering). dim_status_reason criada: mini-dimensão Type 1 com valor sentinela "Outro" + campo livre complementar. Pendência de negócio identificada: confirmar com fonte operacional se existe vínculo explícito de renovação de contrato (fk_previous_contract) antes de modelar linhagem.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| 2026-07-06 | `fact_attendance` ganha `fk_contract` (FK → `dim_contract.sk_contract`). Motivação: o contrato usado já é conhecido no momento da abertura do atendimento — não deve ser inferido depois via `client_id` + `service_type_id`, o que seria ambíguo em casos de múltiplos contratos ativos do mesmo cliente na mesma categoria de serviço. A FK aponta para a versão (`sk_contract`) vigente no momento do atendimento, preservando o estado histórico do contrato (mesmo padrão já aplicado em `dim_client`/endereço). Isso resolve o gap identificado na sessão anterior e viabiliza o cálculo de excedente via `dim_plan_coverage`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| 2026-07-19 | **Correção de consistência documental (sem mudança de decisão arquitetural):** `Diagrama ER.md` estava desatualizado — gerado antes da decisão de `fk_contract` em `fact_attendance`, não refletia o elo. Corrigido. Tabela-resumo de SCD por entidade (seção 6.6 aqui, e seção 6 "Contratos e planos" em `AssistBR_Modelagem_Tecnica.md`) ainda usava o desenho genérico pré-`dim_contract` (ex.: "Plano contratado: Type 2", quando na prática plano não é versionado — troca de plano gera contrato novo via renovação). Ambas reescritas para refletir a decisão granular real, por entidade/coluna. Nenhuma decisão nova — apenas sincronização dos documentos técnicos com o que já estava fechado.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| 2026-07-19 | Fechamento de `dim_plan_coverage`: `limit_type_cd`/`limit_value` seguem genéricos (sem mudança de schema); os três tipos de limite (KM, VALOR, QTD_ANUAL) se diferenciam apenas no cálculo, não na estrutura. Dois padrões definidos: KM/VALOR comparam o valor do próprio atendimento contra o limite, direto na Gold; QTD_ANUAL exige contagem acumulada, resolvida com nova coluna `fact_attendance.nr_contract_attendance_seq` (calculada na Silver via window function particionada por `fk_contract`, seguindo o mesmo padrão de TMA/TMC — fato imutável, não regra mutável). Avaliada e descartada a alternativa de mover o cálculo de excedente inteiro para a Silver — corrigiria um limite errado exigiria reprocessar partição inteira, contra o princípio já fixado de regra mutável ficar na Gold. Pendência de negócio nova registrada: janela de contagem do QTD_ANUAL (ano-calendário vs. aniversário do contrato). Propagado para `AssistBR_Modelagem_Tecnica.md`, `Diagrama ER.md` e `AssistBR_Atividades_Portfolio.md`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| 2026-07-19 | `dim_plan` fechada (estrutura): `bk_plan` como BK real (código no sistema operacional); `plan_desc` Type 1; SLA (`nr_min_confirmation_sla`, `nr_min_arrival_sla`) modelado como **colunas Type 2 dentro da própria `dim_plan`**, não como dimensão/bridge separada — decisão baseada no teste de reuso/grão já usado pra justificar `dim_plan_coverage` como bridge: SLA é 1:1 com o plano (sem N:N que justificasse tabela própria), enquanto cobertura é N:N real (plano×tipo de serviço). SCD Type 2 pelo mesmo damage-first reasoning já aplicado a outras regras mutáveis. Mensalidade (`vl_monthly_fee`) fica **fora da estrutura por enquanto** — pendência de negócio nova registrada: valor é fixo por plano ou negociável por contrato PJ, e se há reajuste periódico (reforçaria necessidade de Type 2 de qualquer forma).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| 2026-08-01 | `dim_status_reason` fechada: adicionado `reason_cd` (faltava código pra filtro/agrupamento, só tinha `reason_desc`) e `ds_status_reason_detail` formalizado na tabela de colunas (antes só existia na prosa — mesmo tipo de gap changelog×seção estruturada já corrigido antes em outras dimensões). `dim_authorized_contact` fechada: a bridge `bridge_contract_authorized_contact` passou a ser **versionada** (`dt_start`/`dt_end`/`fl_current`, SCD Type 2 aplicado à própria bridge, com `sk_contract_authorized_contact` próprio) — motivado pelo levantamento (seção 8: funcionário autorizado pode sair sem o contrato encerrar) e pelo mesmo damage-first reasoning já aplicado em `dim_plan_coverage`/SLA: sem vigência, auditoria de atendimento antigo não saberia quem estava autorizado naquela data. `dim_authorized_contact` (pessoa) continua Type 1 — quem muda no tempo é a relação, não os atributos da pessoa. Propagado para `AssistBR_Modelagem_Tecnica.md`, `Diagrama ER.md` e `AssistBR_Atividades_Portfolio.md`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| 2026-08-03 | **As 3 pendências de negócio resolvidas via entrevista simulada com o diretor.** (1) Renovação de contrato: confirmado que gera número de contrato novo, sem vínculo gravado no sistema operacional entre o antigo e o novo — hoje a empresa não sabe de forma confiável o tempo real de relacionamento de um cliente. Modelado como `dim_contract.fk_previous_contract` (FK autorreferenciada, **inferida na Silver** via heurística: mesmo `bk_client` + `dt_start` do novo até 30 dias após `dt_end` do anterior) + `fl_previous_contract_inferred = true` (sempre, quando preenchido) — dado inferido nunca se disfarça de dado confirmado pela fonte, mesmo princípio já usado em `vl_expected`×`vl_realized`. Premissa assumida e documentada (não confirmada pela fonte): todo contrato tem vigência de 1 ano — nenhuma exceção mencionada pelo diretor. (2) Janela do `QTD_ANUAL`: confirmado aniversário do contrato (12 meses a partir de `dt_start`), não ano-calendário — resolve sozinho via particionamento por `fk_contract`, já que renovação sempre gera `fk_contract` novo (ver decisão 1). (3) Mensalidade: PF paga preço de tabela fixo (sem negociação); PJ negocia desconto por volume, valor já registrado no próprio contrato na fonte; reajuste anual pra ambos, mas em PJ pode ser renegociado. Modelado como `dim_plan.vl_monthly_fee` (Type 2, preço de tabela) + `dim_contract.vl_monthly_fee_negotiated` (não versionado, nullable, override PJ) — valor efetivo calculado na Gold via `COALESCE(negotiated, table)`. Reajuste percentual não precisa de dimensão própria — é derivável comparando versões consecutivas de `vl_monthly_fee`. Corrigidas também duas pequenas inconsistências encontradas durante a revisão: célula de `limit_type_cd` em `dim_plan_coverage` ainda marcada "pendente" apesar da estrutura já fechada; seção 9 ainda listava as 3 pendências como abertas. Propagado para `AssistBR_Modelagem_Tecnica.md`, `Diagrama ER.md` e `AssistBR_Atividades_Portfolio.md`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| 2026-08-09 | **`dim_asset` fechada (estrutura).** Duas tabelas físicas separadas, `dim_asset_vehicle` e `dim_asset_property` (sem supertype/subtype comum) — justificado pela ausência de necessidade de consulta analítica unificada entre auto e residencial (dinâmica de custo/precificação estruturalmente incomparável entre os segmentos). BK de `dim_asset_vehicle` = chassi (imutabilidade, mesmo critério de CPF/CNPJ); BK de `dim_asset_property` fica **pendente**, registrada como pendência de negócio pra próxima entrevista com o diretor (endereço sozinho não é exclusivo o suficiente). Cardinalidade contrato×ativo: PF 1:1, PJ 1:N (frota) — confirmado pela nota já existente em `dim_authorized_contact` sobre ativos entrando/saindo de contrato. Relacionamento modelado via **duas bridges versionadas** (`bridge_contract_asset_vehicle`, `bridge_contract_asset_property`; SCD Type 2 na bridge), mesmo padrão de `bridge_contract_authorized_contact` — evita perda de histórico quando um ativo sai da frota sem o contrato encerrar. `fact_attendance` ganha `fk_asset_vehicle` e `fk_asset_property`, duas FKs nullable **por design de domínio** (nunca ambas preenchidas na mesma linha), discriminadas por `dim_service_type.category_desc` já existente — alternativa de FK polimórfica única descartada porque o motor Tabular do Power BI (VertiPaq/Import Mode) exige relacionamento fixo, não suporta FK apontando pra tabela variável; a mesma restrição foi aplicada às bridges (duas bridges por subtipo, não uma única polimórfica), por consistência arquitetural. Classificação SCD por atributo: `plate` Type 2 (peso jurídico/legal, exige reconstituição histórica); `model_desc`/`nr_year` Type 1 (atributos de fábrica); `color_desc` Type 1 (baixo valor de reconstituição histórica — identificação visual operacional, sem peso jurídico); `property_type_desc` Type 2; `nr_area_m2` Type 1 provisório (condicionado a pendência de negócio sobre impacto em precificação); `fk_address` de `dim_asset_property` Type 2 (mesmo padrão de `dim_client.fk_address`). Duas pendências de negócio novas registradas pra próxima entrevista com o diretor: (1) identificador único do imóvel na fonte operacional; (2) se metragem influencia precificação/cobertura. Hipótese de planos diferenciados por ativo dentro de um mesmo contrato de frota (ex.: carro popular vs. executivo vs. carga) levantada mas **não confirmada, não modelada** — registrada como pergunta em aberto, não como estrutura. Propagado para `AssistBR_Modelagem_Tecnica.md`, `Diagrama ER.md` e `AssistBR_Atividades_Portfolio.md`. |