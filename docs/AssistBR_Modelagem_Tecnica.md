# # AssistBR — Resumo Técnico de Modelagem

**Projeto:** BI para assistência 24h  
**Fase:** definições técnicas para início da modelagem dimensional  
**Documento complementar:** [[Levantamento_AssistBR.md]] (requisitos de negócio)

---

## 1. Arquitetura de referência

```
SQL Server (Operacional)  ─┐
SQL Server (Financeiro)   ─┼→ Bronze → Silver → Gold → BI
Excel (Prestadores)       ─┘
```

|Camada|Responsabilidade|
|---|---|
|**Transacional (SSMS)**|Sistemas operacionais legados — escrita transacional|
|**Bronze**|Dados brutos, preservados como na fonte|
|**Silver**|Limpeza, padronização, integração entre fontes, cálculos imutáveis|
|**Gold**|Modelo dimensional (fatos e dimensões), regras mutáveis|
|**Consumo**|Power BI / Databricks SQL|

---

## 2. Padrão de nomenclatura

Inglês abreviado, com dicionário de dados para legibilidade.

|Prefixo|Uso|Exemplo|
|---|---|---|
|`sk_`|Surrogate key (PK técnica)|`sk_attendance`|
|`bk_`|Business key (identificador de origem)|`bk_attendance`|
|`*_id`|Chave estrangeira (sufixo)|`client_id`, `provider_id`|
|`id_dt_*`|FK para dimensão de período|`id_dt_open`|
|`hr_`|Hora (tipo time)|`hr_open`, `hr_accept`|
|`fl_`|Flag booleana|`fl_scheduled`, `fl_accepted`|
|`nr_`|Número / quantidade|`nr_nps`|
|`vl_`|Valor monetário|`vl_expected`, `vl_realized`|
|`ds_`|Descrição|—|
|`nm_`|Nome|—|

**Chaves:**

- `sk_attendance` — gerada pelo pipeline; garante unicidade técnica
- `bk_attendance` — formato `assistência/chamado` (ex.: `123456/1`); elo com sistema operacional

---

## 3. Esquema dimensional — visão geral

**Tipo de schema:** Constellation Schema (Galaxy) — múltiplas fatos compartilhando dimensões.

**Elo entre fatos:** `bk_attendance`

```
                    ┌─────────────────┐
                    │  Dim_Period     │
                    └────────┬────────┘
                             │
    ┌────────────────────────┼────────────────────────┐
    │                        │                        │
┌───┴──────────┐    ┌────────┴────────┐    ┌──────────┴───────┐
│ Fato         │    │ Fato            │    │ Fato             │
│ Atendimento  │    │ Interacao       │    │ Financeiro       │
└───┬──────────┘    └────────┬────────┘    └──────────┬───────┘
    │                        │                        │
    └────────────────────────┼────────────────────────┘
                             │
              Dimensões compartilhadas:
              Client, Location, Period, Service, Provider...
```

---

## 4. Fatos

### 4.1 Fato Atendimento (`fact_attendance`)

|Atributo|Valor|
|---|---|
|**Granularidade**|Uma linha por serviço prestado dentro de um chamado|
|**Processo**|Operacional — do acionamento à conclusão|
|**Responde**|SLA, NPS, TMA, TMC, origem/destino, custo operacional|

**Colunas definidas:**

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_attendance`|int|Surrogate key|
|`bk_attendance`|varchar|Business key (`assistência/chamado`)|
|`analyst_id`|int|FK → dim_analyst (quem abriu)|
|`client_id`|int|FK → dim_client|
|`fk_contract`|int|FK → dim_contract (`sk_contract` — versão vigente no momento do atendimento)|
|`service_id`|int|FK → dim_service|
|`service_type_id`|int|FK → dim_service_type (category_desc + urgency_desc)|
|`modality_id`|int|FK → dim_modality|
|`provider_id`|int|FK → dim_provider|
|`fk_address_origin`|int|FK → dim_address|
|`ds_address_origin`|varchar|Degenerada: rua+número da origem|
|`fk_address_destination`|int|FK → dim_address|
|`ds_address_destination`|Varchar|Degenerada: rua+número do destino|
|`fl_accepted`|boolean|Prestador aceitou o chamado|
|`id_chnl_accept`|int|FK → dim_channel (canal de aceite: sistema/telefone)|
|`id_dt_open`|int|FK → dim_period (data de abertura)|
|`hr_open`|time|Hora de abertura|
|`id_dt_accept`|int|FK → dim_period (data de aceite)|
|`hr_accept`|time|Hora de aceite|
|`arrival_estimated`|int|Previsão de chegada em minutos|
|`id_dt_arrival`|int|FK → dim_period (data de chegada)|
|`hr_arrival`|time|Hora de chegada|
|`id_dt_conclusion`|int|FK → dim_period (data de conclusão)|
|`hr_conclusion`|time|Hora de conclusão|
|`nr_nps`|int|Nota do cliente (1–5)|
|`nr_contract_attendance_seq`|int|Sequência do atendimento dentro da janela de contagem do contrato (ex.: "4º acionamento do ano"). Calculada na Silver via window function particionada por `fk_contract` — fato imutável, mesmo padrão de TMA/TMC. Alimenta a comparação contra `dim_plan_coverage.limit_value` (tipo `QTD_ANUAL`) na Gold. **Janela confirmada com o diretor (2026-07-19): aniversário do contrato (12 meses a partir de `dt_start`), não ano-calendário** — reseta sozinho a cada renovação, já que renovação sempre gera `fk_contract` novo|
|`fk_asset_vehicle`|int|FK → `dim_asset_vehicle` (nullable por design de domínio — só preenchida quando `category_desc = 'Automotivo'`)|
|`fk_asset_property`|int|FK → `dim_asset_property` (nullable por design de domínio — só preenchida quando `category_desc = 'Residencial'`)|

**Métricas calculadas (Silver → Gold):**

|Métrica|Fórmula|Camada|
|---|---|---|
|TMA|`hr_arrival - hr_accept`|Silver|
|TMC|`hr_conclusion - hr_arrival`|Silver|
|`nr_contract_attendance_seq`|Window function particionada por `fk_contract`|Silver|

**Origem e destino:** `fk_address_origin` e `fk_address_destination` referenciam `dim_address` (role-playing dimension — ver seção 5.4). Quando não aplicável (ex.: atendimento residencial sem destino), a FK aponta para a linha sentinela `sk_address = -1` ("Não aplicável"), evitando `NULL` ou texto solto tipo `"N/A"` na fato.

`ds_address_origin`/`ds_address_destination` são **degenerate dimensions**: texto solto (rua + número) direto na fato, sem FK, pois não têm valor de agrupamento em BI e não se repetem entre atendimentos (principalmente em emergências automotivas, onde o endereço varia a cada chamado).

**Removido do modelo original:** `fl_scheduled` foi retirado da fato — informação redundante com `dim_service_type.urgency_desc` (ver seção 5.5). Mantida uma única fonte de verdade para "agendado vs. emergencial".

---

### 4.2 Fato Interação (`fact_interaction`)

|Atributo|Valor|
|---|---|
|**Granularidade**|Uma linha por interação de analista/setor no chamado|
|**Processo**|Operacional — rastreamento de handoffs entre setores|
|**Responde**|Produtividade por analista, volume por setor, tempo por etapa, interações até resolução|

**Motivação:** um chamado pode passar por Central → Backoffice → Operações de Campo, com analistas diferentes em cada etapa. Um único `analyst_id` na Fato Atendimento não captura esse fluxo.

**Colunas:** pendente de detalhamento.

**Relacionamento:** `bk_attendance` conecta à Fato Atendimento.

**Dimensões previstas:** `dim_analyst`, `dim_sector` (+ dimensões compartilhadas).

---

### 4.3 Fato Financeiro (`fact_financial`)

|Atributo|Valor|
|---|---|
|**Granularidade**|Uma linha por lançamento financeiro vinculado a um atendimento|
|**Processo**|Financeiro — custos e receitas extras|
|**Responde**|Margem, custo por serviço/região, divergências de faturamento|

**Distinção de lançamentos (`tipo_lancamento`):**

|Tipo|Descrição|
|---|---|
|`CUSTO_PRESTADOR`|Pagamento ao prestador (mão de obra, km, adicionais)|
|`EXCEDENTE_CLIENTE`|Cobrança ao cliente por ultrapassar cobertura|

**Controle de divergência:**

|Coluna|Descrição|
|---|---|
|`vl_expected`|Valor calculado pela tabela de preços no momento do atendimento|
|`vl_realized`|Valor informado pelo prestador na fatura|
|`vl_difference`|Diferença entre esperado e realizado|

**Colunas:** pendente de detalhamento completo.

**Relacionamento:** `bk_attendance` conecta à Fato Atendimento.

**Margem:** receita de mensalidade − `CUSTO_PRESTADOR` + `EXCEDENTE_CLIENTE` (via dimensões compartilhadas).

---

## 5. Dimensões

### 5.1 Identificadas pelas FKs da Fato Atendimento

|Dimensão|Origem / FK|Status|
|---|---|---|
|`dim_analyst`|`analyst_id`|Pendente|
|`dim_client`|`client_id`|Pendente|
|`dim_service`|`service_id`|Pendente|
|`dim_modality`|`modality_id`|Pendente|
|`dim_provider`|`provider_id`|Estrutura em fechamento (2026-08-12) — ver 5.7a|
|`dim_channel`|`id_chnl_accept`|Pendente|
|`dim_period`|`id_dt_open`, `id_dt_accept`, `id_dt_arrival`, `id_dt_conclusion`|Pendente|

### 5.2 Identificadas pelo levantamento de negócio

|Dimensão|Motivação|Status|
|---|---|---|
|`dim_address`|Origem/destino de atendimentos (guincho automotivo) + endereço de `dim_client`/`dim_asset`. Renomeada de `dim_location` (correção ortográfica)|**Fechada** (estrutura em 5.4)|
|`dim_service_type`|Categoria (residencial/automotivo) × urgência (emergencial/agendado). Criada durante fechamento da Fato Atendimento — não estava prevista no levantamento original|**Fechada** (estrutura em 5.5), SCD pendente|
|`dim_contract`|Contratos PF/PJ, vigência, status|**Fechada (estrutura)** — ver 5.6|
|`dim_plan`|Planos Basic/Premium, limites, SLA|**Fechada** (estrutura em 5.6)|
|`dim_plan_coverage`|Bridge plano×tipo_serviço×limite|SCD Type 2 — colunas finais pendentes (ver 5.6)|
|`dim_authorized_contact`|Responsável PJ autorizado a acionar|**Fechada** — bridge versionada (ver 5.6)|
|`dim_status_reason`|Motivo de mudança de status do contrato|**Fechada** (ver 5.6)|
|`dim_asset_vehicle` / `dim_asset_property`|Veículos e imóveis cobertos|**Fechada** (estrutura em 5.7)|
|`dim_sector`|Central, Backoffice, Operações de Campo|Pendente|

### 5.3 Atributos previstos — dim_period

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

### 5.4 dim_address — estrutura fechada

**Conceito de design:** _role-playing dimension_. Uma única tabela física, referenciada por múltiplas FKs com papéis distintos: `fk_address_origin` e `fk_address_destination` na Fato Atendimento, além de FKs de endereço em `dim_client` e `dim_asset`. Cidade/bairro/estado existem **uma única vez** na dimensão, evitando duplicação.

**Granularidade:** nível macro (bairro/cidade/estado) — **não** inclui rua/número, que fica como _degenerate dimension_ na fato (ver 4.1), por ser único por atendimento e sem valor de agrupamento.

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_address`|int|Surrogate key|
|`bk_address`|varchar|Business key composta: `uf_cd + city_cd + nbhd_cd`. Não existe código nacional único de bairro (IBGE não padroniza), por isso a BK precisa ser composta|
|`nbhd_cd` / `nbhd_desc`|int / varchar|Código e descrição do bairro|
|`city_cd` / `city_desc`|int / varchar|Código e descrição da cidade|
|`uf_cd` / `uf_desc`|int / varchar|Código e descrição do estado|
|`region_cd` / `region_desc`|int / varchar|Reservado para expansão futura — atualmente sempre constante (Brasil não tem subdivisão de região usada)|
|`country_cd` / `country_desc`|int / varchar|Reservado para expansão futura — atualmente sempre `Brasil`|

**SCD:** **Type 1**. Mudanças (renomeação de bairro, novos CEPs) são raras e não justificam manter histórico; sobrescreve sem preservar valor antigo.

**Linha sentinela:** `sk_address = -1`, com descrições `"Não aplicável"`, usada quando a FK de origem/destino não se aplica (em vez de `NULL` ou string `"N/A"` solta).

**Regra de agrupamento em BI:** `GROUP BY`/filtros sempre por **código** (`city_cd`, `uf_cd`, `nbhd_cd`), nunca por `_desc` — nomes de cidade se repetem entre estados diferentes (ex.: mais de uma "Bela Vista" no Brasil), o que causaria agrupamento incorreto se feito por texto.

**Implementação na Gold (Power BI):** como a mesma dimensão é referenciada duas vezes pela fato (origem/destino), e o Power BI só permite um relacionamento ativo por padrão entre duas tabelas, a Gold expõe **duas views** — `dim_address_origin` e `dim_address_destination` — sobre a tabela física única, evitando depender de `USERELATIONSHIP` em DAX.

### 5.5 dim_service_type — estrutura fechada

**Motivação:** o campo `tp_atendimento` (residencial/automotivo × emergencial/agendado) foi identificado como lista pequena e fechada, com risco de inconsistência de texto solto (`"Residencial"` vs `"residencial"`) e potencial de ganhar atributos futuros (regra de SLA por tipo, precificação). Modelado como dimensão, com dois atributos separados em vez de uma única string concatenada — permite filtrar por urgência independente da categoria (ex.: "todos os emergenciais, residencial ou automotivo").

|Coluna|Tipo|Descrição|
|---|---|---|
|`sk_service_type`|int|Surrogate key|
|`bk_service_type`|varchar|Business key (a definir — provável combinação category+urgency)|
|`category_desc`|varchar|Residencial / Automotivo|
|`urgency_desc`|varchar|Emergencial / Agendado|

**Valores conhecidos hoje:** Residencial+Emergencial, Residencial+Agendado, Automotivo+Emergencial, Automotivo+Agendado.

**Pendente:** definição de SCD (lista pequena e estável — provável Type 1, mas não fechado formalmente) e da `bk_service_type`.

**Decisão relacionada:** `fl_scheduled`, que existia solta na Fato Atendimento, foi **removida** — era redundante com `urgency_desc`. Avaliou-se também manter `fl_scheduled` como atributo boolean dentro da própria `dim_service_type` (redundância interna, atalho de leitura), mas optou-se por retirar e manter `urgency_desc` como única fonte de verdade.

### 5.6 Árvore de dim_contract

**dim_client (FECHADA):**

|Coluna|Tipo|SCD|Descrição|
|---|---|---|---|
|`sk_client`|int|—|Surrogate key|
|`bk_client`|varchar|—|CPF (PF) ou CNPJ (PJ)|
|`person_type_cd`|char|Type 1|PF/PJ — nativo da fonte operacional (documentos de cadastro distintos)|
|`nm_client`|varchar|Type 1|Nome ou razão social|
|`fk_address`|int|Type 2|FK → dim_address|
|`phone`, `email`|varchar|Type 1|—|

Múltiplos contratos: resolvido via `dim_contract.client_id` N:1, sem estrutura extra em `dim_client`.

**dim_contract (FECHADA — estrutura):**

|Coluna|Tipo|SCD|Descrição|
|---|---|---|---|
|`sk_contract`|int|—|Surrogate key|
|`bk_contract`|varchar|—|Número único do contrato|
|`client_id`|int|—|FK → dim_client|
|`fk_plan`|int|—|FK → dim_plan|
|`dt_start`/`dt_end`|date|—|Vigência do contrato|
|`status_cd`|int|Type 2|Ativo/Cancelado/Suspenso|
|`fk_status_reason`|int|Type 1|FK → dim_status_reason|
|`person_type_cd`|char|Type 1|Replicado de dim_client — denormalização proposital|
|`fk_previous_contract`|int|Não versionado|FK autorreferenciada → `sk_contract` do contrato anterior, quando há renovação. **Inferida na Silver** (mesmo `bk_client` + `dt_start` do novo até 30 dias após `dt_end` do anterior) — fonte confirmou que não existe esse vínculo gravado|
|`fl_previous_contract_inferred`|boolean|—|Sempre `true` quando `fk_previous_contract` preenchido — dado inferido nunca se disfarça de dado confirmado pela fonte|
|`vl_monthly_fee_negotiated`|decimal|Não versionado|Mensalidade negociada, só pra contratos PJ com desconto por volume. Nullable — vazio usa `dim_plan.vl_monthly_fee`|

Renovação = nova linha (novo `bk_contract`), nunca versão Type 2 da anterior. **Confirmado com o diretor (2026-07-19):** não existe vínculo gravado no sistema operacional entre contrato antigo e novo.

**Premissa assumida (não confirmada pela fonte):** todo contrato tem vigência de 1 ano — sustenta `nr_contract_attendance_seq` particionar só por `fk_contract` sem janela móvel dentro do mesmo contrato.

**Elo com `fact_attendance`:** `fk_contract` na fato (ver 4.1), apontando para `sk_contract` — o contrato usado já é conhecido no momento da abertura do atendimento, evitando inferência ambígua quando o cliente tem múltiplos contratos ativos. A FK referencia a versão vigente naquele momento, preservando o estado histórico do contrato.

**dim_plan** (separada de contrato — alto reuso, FECHADA): `sk_plan`, `bk_plan` (BK real — código no sistema operacional), `plan_desc` (Type 1), `nr_min_confirmation_sla`/`nr_min_arrival_sla` (Type 2, valores do levantamento seção 4.1: Premium 15/60min, Basic 30/120min), `vl_monthly_fee` (Type 2, preço de tabela — confirmado com o diretor: PF paga fixo, reajuste anual), `dt_start`/`dt_end`, `fl_current`. **SLA modelado como colunas Type 2 na própria dimensão**, não como tabela separada — SLA é 1:1 com o plano (sem N:N que justificasse bridge, diferente de `dim_plan_coverage`).

**Mensalidade — resolvida (2026-07-19), confirmado com o diretor:** PF paga preço de tabela fixo, sem negociação; PJ negocia desconto por volume, valor já registrado no próprio contrato na fonte; reajuste anual pra ambos, mas PJ pode ser renegociado. `vl_monthly_fee` (tabela, Type 2) em `dim_plan`; `vl_monthly_fee_negotiated` (override, nullable) em `dim_contract`. **Valor efetivo na Gold:** `COALESCE(dim_contract.vl_monthly_fee_negotiated, dim_plan.vl_monthly_fee)`. Reajuste percentual é derivável comparando versões consecutivas — não precisa de dimensão própria.

**dim_plan_coverage** (bridge, SCD Type 2 — regra mutável, risco de cobrança retroativa indevida se não versionar): `sk_plan_coverage`, `fk_plan`, `fk_service_type`, `limit_type_cd` (KM/VALOR/QTD_ANUAL), `limit_value`, `dt_start`/`dt_end`, `fl_current`. **Estrutura da tabela é genérica e final** — a diferença entre os tipos de limite não exige colunas separadas, e sim dois caminhos de cálculo distintos na Gold (decisão de 2026-07-19, ver seção 6): `KM`/`VALOR` comparam o próprio atendimento contra `limit_value` direto na Gold; `QTD_ANUAL` depende de uma contagem acumulada calculada na Silver (`fact_attendance.nr_contract_attendance_seq`) e comparada contra `limit_value` na Gold.

**dim_authorized_contact** ("responsável PJ", N:N com contrato via bridge versionada, autorização por contrato inteiro e não por ativo, FECHADA): `sk_authorized_contact`, `nm_contact`, `doc_contact` (Type 1 — atributos da pessoa não mudam por autorização). Bridge `bridge_contract_authorized_contact` (versionada, SCD Type 2): `sk_contract_authorized_contact`, `fk_contract`, `fk_authorized_contact`, `dt_start`/`dt_end`, `fl_current`. Decisão: funcionário pode ser desautorizado sem o contrato encerrar (levantamento seção 8) — sem vigência, auditoria de atendimento antigo não saberia quem estava autorizado naquela data. Quem versiona é a relação (bridge), não a pessoa.

**dim_status_reason** (mini-dimensão Type 1, FECHADA): `sk_status_reason`, `reason_cd` (filtro/agrupamento por código), `reason_desc`, `ds_status_reason_detail` (campo livre, só preenchido quando `reason_cd = 99`/"Outro").

**As 3 pendências de negócio foram resolvidas via entrevista com o diretor em 2026-07-19** — ver changelog do `PROJECT_CONTEXT.md` para os detalhes completos das respostas.

### 5.7 dim_asset — estrutura fechada (2026-08-09)

**Motivação:** veículos e imóveis cobertos por contrato. Precisava resolver dois problemas: (1) como representar dois tipos de ativo com atributos estruturalmente diferentes sem gerar colunas nulas em massa; (2) como um contrato PJ pode cobrir múltiplos ativos (frota) sem perder histórico quando um ativo entra/sai.

**Decisão estrutural:** duas tabelas físicas separadas, `dim_asset_vehicle` e `dim_asset_property`, sem tabela núcleo supertype/subtype compartilhada. Avaliado e descartado o padrão supertype/subtype (tabela núcleo com SK comum + tabelas satélite por subtipo) — só compensaria se houvesse necessidade real de análise unificada entre auto e residencial, e não há: as dinâmicas de custo/precificação dos dois segmentos são estruturalmente incomparáveis (auto majoritariamente tabelado, residencial com dinâmica de sinistro diferente).

**Cardinalidade com `dim_contract`:** PF = 1:1; PJ = 1:N (frota). Resolvida via bridge versionada, não FK direta — o levantamento já confirmava que ativos podem entrar/sair de um contrato sem ele encerrar (mesma nota já registrada na decisão de `dim_authorized_contact`).

**`dim_asset_vehicle`:**

|Coluna|Tipo|SCD|Descrição|
|---|---|---|---|
|`sk_asset_vehicle`|int|—|Surrogate key (versionada)|
|`bk_asset_vehicle`|varchar|—|Chassi — identificador imutável, mesmo critério já usado em CPF/CNPJ (`dim_client`). Preferido sobre placa, que muda por transferência/remarcação|
|`plate`|varchar|Type 2|Placa — peso jurídico/legal, exige reconstituição histórica|
|`model_desc`|varchar|Type 1|Atributo de fábrica, não muda|
|`nr_year`|int|Type 1|Atributo de fábrica, não muda|
|`color_desc`|varchar|Type 1|Pode mudar (repintura), mas baixo valor de reconstituição histórica — identificação visual operacional, sem peso jurídico como a placa|
|`fk_address`|int|Type 2|FK → dim_address|
|`dt_start` / `dt_end` / `fl_current`|date / boolean|—|Vigência da versão|

**`dim_asset_property`:**

|Coluna|Tipo|SCD|Descrição|
|---|---|---|---|
|`sk_asset_property`|int|—|Surrogate key (versionada)|
|`bk_asset_property`|varchar|—|**Pendente de negócio** — endereço sozinho não é exclusivo o suficiente (imóveis podem compartilhar endereço). A validar com o diretor: existe matrícula/IPTU/cadastro municipal na fonte?|
|`property_type_desc`|varchar|Type 2|Pode mudar por reclassificação cadastral|
|`nr_area_m2`|decimal|Type 1 (provisório)|Sem regra de precificação/cobertura ligada a metragem confirmada hoje — reclassificar se a pendência de negócio confirmar o contrário|
|`fk_address`|int|Type 2|FK → dim_address, mesmo padrão de `dim_client.fk_address`|
|`dt_start` / `dt_end` / `fl_current`|date / boolean|—|Vigência da versão|

**Bridges (SCD Type 2 na própria bridge, mesmo padrão de `bridge_contract_authorized_contact`):** `bridge_contract_asset_vehicle` (`sk_contract_asset_vehicle`, `fk_contract`, `fk_asset_vehicle`, `dt_start`/`dt_end`/`fl_current`) e `bridge_contract_asset_property` (equivalente, com `fk_asset_property`). Duas bridges separadas, não uma única polimórfica — mesma restrição de relacionamento fixo do Power BI Tabular já aplicada à fato.

**Elo com `fact_attendance`:** `fk_asset_vehicle` e `fk_asset_property`, duas FKs nullable **por design de domínio** (nunca as duas preenchidas na mesma linha), discriminadas por `dim_service_type.category_desc` já existente. FK polimórfica única (`fk_asset` apontando pra tabela variável) avaliada e descartada: o motor Tabular do Power BI (VertiPaq/Import Mode) exige relacionamento fixo entre tabelas.

**Pendências de negócio registradas (2026-08-09):**
1. BK de `dim_asset_property` (matrícula/IPTU/equivalente).
2. Se metragem do imóvel influencia precificação/cobertura (impacta SCD de `nr_area_m2` e possivelmente `dim_plan_coverage`).
3. Hipótese (não confirmada, não modelada): planos/coberturas diferenciados por ativo dentro de um mesmo contrato de frota PJ (ex.: popular vs. executivo vs. carga).

---

### 5.7a dim_provider — estrutura em fechamento (2026-08-12)

Prestador credenciado (guincho, chaveiro, encanador, eletricista etc.). Grão: uma linha por prestador (PJ). Especialidade e preço resolvidos como **relação** (bridge), não atributo da dimensão — um mesmo prestador pode atender múltiplas categorias de serviço.

**`dim_provider`:** `sk_provider`, `bk_provider` (CNPJ — premissa assumida "prestador sempre PJ", não confirmada), `fk_address` (Type 2, sede do prestador, reaproveita `dim_address`), `status_cd` (Type 2, mesmo padrão de `dim_contract.status_cd`), `fk_provider_status_reason` (Type 1, FK → nova mini-dimensão `dim_provider_status_reason`), `dt_start`/`dt_end`/`fl_current`.

**`dim_provider_status_reason`:** catálogo próprio e independente de `dim_status_reason` (não reaproveitado via role-playing) — motivo de contrato e motivo de descredenciamento de prestador não têm sobreposição de significado nem de governança (times diferentes), mesmo teste já usado para rejeitar supertype/subtype em `dim_asset`.

**`bridge_provider_service_type`** (N:N versionada, SCD Type 2): `fk_provider`, `fk_service_type`, `vl_price` (preço-padrão), vigência. Mesmo padrão de `dim_plan_coverage`.

**`bridge_provider_service_region_price`** (exceção negociada, N:N versionada): `fk_provider`, `fk_service_type`, `fk_address` (região), `vl_price`, vigência. Resolve o risco de preço "ad-hoc" por atendimento (ex.: acionamentos repetidos fora da área numa enchente inflando custo sem controle) — acordo sempre **pré-negociado e versionado**, nunca decidido na hora. Valor efetivo na Gold: `COALESCE(bridge_provider_service_region_price.vl_price, bridge_provider_service_type.vl_price)`.

**Pendências de negócio (2026-08-12):** (1) confirmar premissa "prestador sempre PJ"; (2) área de atuação/cobertura regional — não modelada agora, registrada a pergunta certa para o diretor (métrica real de gap de rede vs. regra operacional de despacho); (3) confirmar existência do cenário de preço negociado por região.

**Débito técnico:** normalização de CEP/endereço do Excel de prestadores fica na Silver (data quality/master data), não gera dimensão de geografia nova — `dim_address` continua denormalizada (5.4); snowflaking (`dim_uf`/`dim_city`/`dim_cep` separadas) cogitado e descartado pelo mesmo motivo já usado para justificar a tabela física única de `dim_address`.

## 6. SCD — decisões por entidade

### Contratos e planos

> Tabela atualizada após o fechamento da árvore `dim_contract`/`dim_plan`/`dim_plan_coverage`/`dim_authorized_contact` (ver seção 5.6). A versão anterior desta tabela tratava plano, ativos, limites e responsáveis como um bloco único Type 2 — era uma intenção inicial pré-desenho granular, hoje substituída pelas decisões específicas por entidade abaixo.

|Atributo|Tipo SCD|Onde vive|Motivo|
|---|---|---|---|
|Status do contrato|Type 2|`dim_contract.status_cd`|Auditoria e disputas (ativo/suspenso/cancelado) — histórico precisa ser preservado|
|Motivo de status|Type 1|`dim_contract.fk_status_reason` → `dim_status_reason`|Catálogo não versiona; quem versiona é o vínculo em dim_contract|
|PF/PJ|Type 1|`dim_contract.person_type_cd` (replicado de dim_client)|Atributo nativo, não muda|
|Plano contratado|Não versionado|`dim_contract.fk_plan` → `dim_plan`|Troca de plano no meio do contrato não é cenário confirmado no levantamento; hoje modelado como renovação = contrato novo (novo `bk_contract`), não como Type 2 dentro do mesmo registro|
|Mensalidade (preço de tabela)|Type 2|`dim_plan.vl_monthly_fee`|PF paga fixo, sem negociação; reajuste anual confirmado com o diretor — histórico precisa ser preservado|
|Mensalidade negociada (PJ)|Não versionado|`dim_contract.vl_monthly_fee_negotiated`|Valor fixado no momento da assinatura; renegociação vira contrato novo, não update na mesma linha (nullable, override do valor de tabela via `COALESCE` na Gold)|
|Linhagem de renovação|Não versionado (inferência)|`dim_contract.fk_previous_contract` + `fl_previous_contract_inferred`|Fonte operacional confirmou que não grava vínculo entre contratos; inferência via heurística (30 dias de tolerância) marcada explicitamente como não confirmada|
|SLA (confirmação / início)|Type 2|`dim_plan.nr_min_confirmation_sla` / `nr_min_arrival_sla`|1:1 com o plano (sem N:N que justifique bridge); regra mutável — histórico precisa ser preservado se o board apertar o prazo|
|Limites de cobertura por serviço|Type 2 (bridge inteira)|`dim_plan_coverage`|Regra mutável com risco de auditoria/disputa jurídica se não versionada. **Dois padrões de cálculo na Gold (2026-07-19):** `KM`/`VALOR` — comparação linha a linha contra o próprio atendimento; `QTD_ANUAL` — comparação contra contagem acumulada, calculada na Silver (`fact_attendance.nr_contract_attendance_seq`, mesma natureza de TMA/TMC) e não recalculada na Gold, só comparada|
|Responsáveis autorizados (PJ)|Type 2 (bridge `bridge_contract_authorized_contact`)|`dim_authorized_contact` + bridge|Funcionário pode ser desautorizado sem o contrato encerrar (levantamento seção 8); sem vigência, auditoria não sabe quem estava autorizado numa data passada. Pessoa (nome/CPF) continua Type 1 — quem versiona é a relação|
|Placa do veículo|Type 2|`dim_asset_vehicle.plate`|Peso jurídico/legal (fiscalização, disputa de sinistro) — exige reconstituição histórica|
|Modelo / ano do veículo|Type 1|`dim_asset_vehicle.model_desc` / `nr_year`|Atributos de fábrica, não mudam|
|Cor do veículo|Type 1|`dim_asset_vehicle.color_desc`|Pode mudar (repintura), mas baixo valor de reconstituição histórica — identificação visual operacional, sem peso jurídico como a placa|
|Tipo de imóvel|Type 2|`dim_asset_property.property_type_desc`|Pode mudar por reclassificação cadastral|
|Metragem do imóvel|Type 1 (provisório)|`dim_asset_property.nr_area_m2`|Sem regra de precificação/cobertura confirmada hoje — condicionado a pendência de negócio|
|Endereço do imóvel|Type 2|`dim_asset_property.fk_address`|Mesmo padrão de `dim_client.fk_address` — análises por região dependem do endereço no momento do evento|
|Vínculo ativo×contrato (frota)|Type 2 (bridge inteira)|`bridge_contract_asset_vehicle` / `bridge_contract_asset_property`|Ativo pode entrar/sair da frota sem o contrato encerrar; sem vigência, auditoria não sabe quais ativos estavam cobertos numa data passada|
|Telefone, e-mail (cliente)|Type 1|`dim_client`|Sem valor analítico histórico|

### Prestadores

|Atributo|Tipo SCD|Motivo|
|---|---|---|
|Preço-padrão por prestador×serviço (`bridge_provider_service_type`)|Type 2 (bridge inteira)|N:N real; preço e credenciamento mudam sem alterar o cadastro do prestador — mesmo damage-first reasoning de `dim_plan_coverage`|
|Preço negociado por região (`bridge_provider_service_region_price`)|Type 2 (bridge inteira)|Exceção pré-negociada, não decidida ad-hoc por atendimento (risco de custo inflado sem controle)|
|Status (ativo/inativo/suspenso)|Type 2|Auditoria — mesmo padrão de `dim_contract.status_cd`|
|Endereço de sede (`fk_address`)|Type 2|Mesmo padrão de `dim_client.fk_address`|
|Área de atuação/cobertura regional|—|**Não modelada** — pendência de negócio (métrica real vs. regra de despacho)|

### Cliente

|Atributo|Tipo SCD|Motivo|
|---|---|---|
|Endereço|Type 2|Análises por região dependem do endereço no momento do evento|
|Telefone, e-mail|Type 1|Contato atual|

### Padrão técnico de implementação — SCD Type 2

Discutido em detalhe ao fechar `dim_client`. Quando um atributo é Type 2, a dimensão passa a ter **múltiplas linhas por entidade de negócio** (uma por versão). Isso exige:

- **`sk_*`**: única por linha (por versão) — é o que a FK da fato deve referenciar.
- **`bk_*`**: se repete entre as versões do mesmo registro (ex.: CPF do cliente) — usada só para identificar/deduplicar no ETL, nunca como FK na fato.
- **`dt_inicio_vigencia`** / **`dt_fim_vigencia`**: delimitam o período em que aquela versão foi válida. Necessárias para responder "qual era o valor do atributo em uma data específica no passado" (ex.: endereço do cliente no momento de um atendimento antigo).
- **`fl_current`** (ou `fl_vigente`): flag booleana complementar — não substitui as datas de vigência, apenas otimiza consultas de "estado atual" sem precisar comparar datas. Consultas de estado **histórico** exigem as datas; consultas de estado **atual** podem usar a flag.

---

## 7. Regras de transformação por camada

|Regra|Tipo|Camada|
|---|---|---|
|TMA = chegada − aceite|Imutável|Silver|
|TMC = conclusão − chegada|Imutável|Silver|
|SLA por plano|Mutável|Gold|
|Cobertura por serviço|Mutável|Gold|
|Valor pago ao prestador|Mutável|Gold|
|Padronização de formatos (telefone, status)|—|Silver|
|Deduplicação e integração entre fontes|—|Silver|
|`vl_expected` vs `vl_realized`|—|Silver/Gold|

---

## 8. Fontes de dados e tratamentos na Silver

|Fonte|Problemas conhecidos|Tratamento na Silver|
|---|---|---|
|SQL Server operacional|Normalizado, legado|Padronização de status, tipos|
|SQL Server financeiro|Servidor separado, sem integração|Join via `bk_attendance`|
|Excel prestadores|Sem governança, sem padrão, sem lineage|Padronização de região, telefone, especialidades|

**Bronze:** preserva dado bruto do Excel intacto.

---

## 9. Decisões de design registradas

|Decisão|Escolha|Justificativa|
|---|---|---|
|Separar operacional × financeiro|Fatos distintas|Métricas e granularidades diferentes|
|Prioridade de modelagem|Operacional primeiro|Financeiro depende do operacional|
|Múltiplos analistas por chamado|Fato Interação separada|Evita fato esparsa com muitos nulls|
|Margem|Dimensões compartilhadas entre fatos|Constellation Schema|
|Excedente do cliente|Mesma fato financeira, `tipo_lancamento` distinto|Natureza diferente, mesma granularidade|
|Datas múltiplas por evento|`id_dt_*` + `hr_*` separados|Dimensão de período reutilizável; hora para cálculos|
|Métricas deriváveis|Pré-calculadas na Silver|Performance em volume alto|
|Cadastro de prestadores|Modelar como SQL (evolução futura)|Portfólio demonstra maturidade arquitetural|
|Endereço de atendimento|`dim_address` como role-playing dimension (FK dupla: origem/destino) em vez de códigos duplicados soltos na fato|JOIN único por `sk_address`; troca futura de BK não impacta a fato|
|Endereço completo (rua+número)|Degenerate dimension (texto solto na fato)|Único por atendimento, sem valor de agrupamento; não justifica dimensão própria|
|Valor ausente em FK opcional|Linha sentinela `sk = -1` ("Não aplicável") na dimensão, em vez de `NULL` ou string `"N/A"`|Toda FK sempre resolve em JOIN válido; evita texto solto inconsistente|
|Consumo de role-playing dimension no Power BI|Views separadas na Gold (`dim_address_origin`, `dim_address_destination`) sobre a tabela física única|Evita depender de `USERELATIONSHIP` manual em DAX|
|Agrupamento/filtro em BI|Sempre por código (`_cd`/`sk`), nunca por descrição (`_desc`)|Descrições podem se repetir entre entidades diferentes (ex.: nomes de cidade duplicados entre estados)|
|Tipo de atendimento (residencial/automotivo × emergencial/agendado)|Nova dimensão `dim_service_type`, com dois atributos separados (`category_desc`, `urgency_desc`) em vez de string única ou flags soltas|Evita inconsistência de texto livre; permite filtrar urgência independente da categoria|
|`fl_scheduled` (fato)|Removido|Redundante com `dim_service_type.urgency_desc`; evita duas fontes de verdade divergentes|
|Limites `QTD_ANUAL` em `dim_plan_coverage`|Estrutura da bridge continua genérica; contagem acumulada vira coluna `fact_attendance.nr_contract_attendance_seq`, calculada na Silver (imutável); comparação contra o limite fica na Gold|Mesmo raciocínio já validado em TMA/TMC × SLA: separar o que é fato imutável (contagem de eventos passados) do que é regra mutável (o limite em si). Mover o cálculo inteiro pra Silver foi avaliado e descartado — correção de limite exigiria reprocessar partição inteira|
|Linhagem de renovação de contrato|`fk_previous_contract` inferido na Silver (heurística 30 dias) + `fl_previous_contract_inferred = true`|Confirmado com o diretor: fonte não grava vínculo entre contratos; dado inferido nunca se disfarça de confirmado|
|Janela do limite `QTD_ANUAL`|Aniversário do contrato (12 meses a partir de `dt_start`), não ano-calendário|Confirmado com o diretor; reseta sozinho a cada renovação via particionamento por `fk_contract`|
|Mensalidade PF vs. PJ|`dim_plan.vl_monthly_fee` (Type 2, tabela) + `dim_contract.vl_monthly_fee_negotiated` (override PJ, não versionado)|Confirmado com o diretor: PF fixo, PJ negociado por volume, reajuste anual; valor efetivo via `COALESCE` na Gold|
|Estrutura de `dim_asset`|Duas tabelas físicas separadas (`dim_asset_vehicle`, `dim_asset_property`), sem supertype/subtype comum|Ausência de necessidade de consulta analítica unificada entre auto e residencial — dinâmicas de custo/precificação estruturalmente incomparáveis entre os segmentos|
|BK do veículo|Chassi (`dim_asset_vehicle.bk_asset_vehicle`)|Imutabilidade — mesmo critério já usado em CPF/CNPJ; placa descartada por ser mutável (transferência, remarcação)|
|Relacionamento contrato×ativo|Bridges versionadas (`bridge_contract_asset_vehicle`/`property`), SCD Type 2 na bridge, não FK direta em `dim_asset`|Frota PJ pode ganhar/perder ativos sem o contrato encerrar; FK fixa perderia a capacidade de reconstituir "quais ativos estavam cobertos numa data passada"|
|Elo `fact_attendance` × ativo|Duas FKs nullable por design (`fk_asset_vehicle`, `fk_asset_property`), discriminadas por `dim_service_type.category_desc`|Motor Tabular do Power BI exige relacionamento fixo — FK polimórfica única (`fk_asset` apontando pra tabela variável) não é nativamente suportada; mesma restrição aplicada às bridges por consistência|

---

## 10. Pendências técnicas

| Item                                                         | Descrição                                                                                                                                                                |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Colunas — Fato Interação                                     | Detalhar FKs, timestamps, flags por setor                                                                                                                                |
| Colunas — Fato Financeiro                                    | Detalhar componentes (mão de obra, km, adicionais)                                                                                                                       |
| ~~`dim_location`~~ `dim_address`                             | **Resolvido** — hierarquia macro (uf/city/nbhd), SCD Type 1 (não Type 2 como constava antes), BK composta, role-playing, linha sentinela -1. Estrutura em 5.4            |
| `dim_service_type`                                           | Estrutura básica fechada (5.5); falta definir SCD formalmente e `bk_service_type`                                                                                        |
| `dim_client`                                                 | **Resolvido** — PF/PJ nativo via `person_type_cd` (Type 1), múltiplos contratos resolvidos via N:1 em dim_contract. Ver 5.6                                              |
| `dim_contract`                                               | **Resolvido (estrutura)** — grão 1 linha/contrato, `status_cd` Type 2. Ver 5.6. Elo com fato resolvido via `fk_contract`                                    |
| `dim_plan_coverage`                                          | **Resolvido (estrutura)** — bridge SCD Type 2, colunas finais fechadas. `limit_type_cd` genérico comporta os 3 tipos de limite sem mudança de schema; diferença fica no cálculo (Gold). Janela do QTD_ANUAL confirmada com o diretor: aniversário do contrato |
| `dim_plan`                                                    | **Resolvido (estrutura)** — `bk_plan` BK real, `plan_desc` Type 1, SLA e mensalidade (`vl_monthly_fee`) como colunas Type 2. Confirmado com o diretor: PF fixo, PJ via `dim_contract.vl_monthly_fee_negotiated` |
| `dim_authorized_contact`, `dim_status_reason` | **Resolvido** — bridge versionada (SCD Type 2) e mini-dimensão com `reason_cd`/`ds_status_reason_detail` formalizados. Ver 5.6 |
| `dim_analyst`                                                | Estrutura completa — não iniciada                                                                                                                                        |
| `dim_service`, `dim_modality`, `dim_channel`                | Estrutura completa — não iniciadas                                                                                                                                       |
| `dim_asset`                                                  | **Resolvido (estrutura)** — duas tabelas físicas (`dim_asset_vehicle`, `dim_asset_property`), bridges versionadas pra vigência de cobertura. Ver 5.7. Elo com fato resolvido via `fk_asset_vehicle`/`fk_asset_property`. Pendências de negócio: BK do imóvel, impacto de metragem em precificação |
| `dim_provider`                                               | **Resolvido (estrutura em fechamento)** — cadastro + `bridge_provider_service_type` + `bridge_provider_service_region_price`. Ver 5.7a. 2 pendências de negócio: premissa PJ, cobertura regional |
| `dim_sector`                                                 | Estrutura completa — não iniciada                                                                                                                                        |
| Diagrama ER                                                  | Representação visual do modelo completo                                                                                                                                  |
| Dicionário de dados                                          | Mapeamento abreviação → descrição em português                                                                                                                           |