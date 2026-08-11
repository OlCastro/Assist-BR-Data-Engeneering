// ============================================================
// AssistBR — Modelo Dimensional (estado atual)
// Constellation Schema: 3 fatos ligados por bk_attendance
// Gerado a partir das decisões fechadas até a sessão de hoje
// ============================================================
// COMO VISUALIZAR:
// 1. Acesse https://dbdiagram.io
// 2. Clique em "Go to App" (não precisa criar conta pra testar)
// 3. Apague o conteúdo de exemplo e cole este arquivo inteiro
// 4. O diagrama ER é desenhado automaticamente, com as setas
//    de relacionamento já conectadas
// ============================================================

// ---------------------------
// DIMENSÕES FECHADAS
// ---------------------------

Table dim_address {
  sk_address int [pk]
  bk_address varchar [note: 'composta: uf_cd + city_cd + nbhd_cd']
  nbhd_cd int
  nbhd_desc varchar
  city_cd int
  city_desc varchar
  uf_cd int
  uf_desc varchar
  region_cd int
  region_desc varchar
  country_cd int
  country_desc varchar

  Note: 'SCD Type 1. Linha sentinela sk_address = -1 ("Não aplicável"). Role-playing dimension (origem/destino).'
}

Table dim_client {
  sk_client int [pk]
  bk_client varchar [note: 'CPF ou CNPJ']
  person_type_cd varchar [note: 'PF / PJ — Type 1, nativo do cadastro']
  nm_client varchar
  fk_address int [ref: > dim_address.sk_address]
  dt_address_start date
  dt_address_end date
  phone varchar [note: 'Type 1']
  email varchar [note: 'Type 1']
  fl_current boolean

  Note: 'FECHADA. Endereço é Type 2 (versionado). Múltiplos contratos resolvidos via N:1 em dim_contract.'
}

Table dim_plan {
  sk_plan int [pk]
  bk_plan varchar [note: 'BK real — código do plano no sistema operacional']
  plan_desc varchar [note: 'Basic / Premium — Type 1']
  nr_min_confirmation_sla int [note: 'Prazo máx. confirmação do prestador (min). Premium: 15, Basic: 30. Type 2.']
  nr_min_arrival_sla int [note: 'Prazo máx. início do atendimento (min). Premium: 60, Basic: 120. Type 2.']
  vl_monthly_fee decimal [note: 'Preço de tabela do plano. Type 2 (reajuste anual confirmado com o diretor). PF paga direto; PJ usa como referência se não houver valor negociado.']
  dt_start date
  dt_end date
  fl_current boolean

  Note: 'FECHADA (estrutura). SLA e mensalidade modelados como colunas Type 2 na própria dimensão. Mensalidade PJ negociada: ver dim_contract.vl_monthly_fee_negotiated.'
}

Table dim_plan_coverage {
  sk_plan_coverage int [pk]
  fk_plan int [ref: > dim_plan.sk_plan]
  fk_service_type int [ref: > dim_service_type.sk_service_type]
  limit_type_cd varchar [note: 'KM / VALOR / QTD_ANUAL — tipos de limite distintos, ver observação']
  limit_value decimal
  dt_start date
  dt_end date
  fl_current boolean

  Note: 'Bridge versionada (SCD Type 2). Justificativa: risco de cobrança retroativa indevida / disputa jurídica. Estrutura genérica é final (2026-07-19) — KM/VALOR/QTD_ANUAL não exigem colunas separadas. Diferença fica no cálculo: KM/VALOR comparam direto na Gold; QTD_ANUAL usa fact_attendance.nr_contract_attendance_seq (contagem calculada na Silver) comparado ao limite na Gold. Janela do QTD_ANUAL confirmada com o diretor: aniversário do contrato (12 meses a partir de dt_start), não ano-calendário.'
}

Table dim_status_reason {
  sk_status_reason int [pk]
  reason_cd int [note: 'Código do motivo — uso em filtro/agrupamento. Sentinela 99 = Outro']
  reason_desc varchar [note: 'Inadimplência, Solicitação do cliente, Suspeita de fraude, Renegociação, Outro']
  ds_status_reason_detail varchar [note: 'Campo livre, preenchido somente quando reason_cd = 99']

  Note: 'FECHADA. Mini-dimensão, SCD Type 1.'
}

Table dim_contract {
  sk_contract int [pk]
  bk_contract varchar [note: 'Número único do contrato']
  client_id int [ref: > dim_client.sk_client]
  fk_plan int [ref: > dim_plan.sk_plan]
  dt_start date
  dt_end date
  status_cd varchar [note: 'Ativo / Cancelado / Suspenso — Type 2']
  fk_status_reason int [ref: > dim_status_reason.sk_status_reason]
  dt_status_start date
  dt_status_end date
  fl_current boolean
  person_type_cd varchar [note: 'Replicado de dim_client — denormalização proposital (baixa cardinalidade, alta frequência de filtro)']
  fk_previous_contract int [ref: > dim_contract.sk_contract, note: 'Autorreferenciada. Inferida na Silver (mesmo bk_client + dt_start do novo até 30 dias após dt_end do anterior) — fonte confirmou que não existe vínculo gravado. Nullable.']
  fl_previous_contract_inferred boolean [note: 'Sempre true quando fk_previous_contract preenchido — dado inferido nunca se disfarça de confirmado pela fonte']
  vl_monthly_fee_negotiated decimal [note: 'Mensalidade negociada, só PJ com desconto por volume. Nullable — vazio usa dim_plan.vl_monthly_fee. Não versionado.']

  Note: 'FECHADA (estrutura). Renovação = nova linha (novo bk_contract), não versão da mesma. Confirmado com o diretor: não existe vínculo gravado no sistema operacional entre contrato antigo e novo. Premissa assumida: todo contrato tem vigência de 1 ano.'
}

Table dim_authorized_contact {
  sk_authorized_contact int [pk]
  nm_contact varchar
  doc_contact varchar [note: 'CPF do responsável autorizado']

  Note: 'FECHADA. Type 1 — atributos da pessoa não mudam por autorização. Quem versiona é a bridge.'
}

Table bridge_contract_authorized_contact {
  sk_contract_authorized_contact int [pk]
  fk_contract int [ref: > dim_contract.sk_contract]
  fk_authorized_contact int [ref: > dim_authorized_contact.sk_authorized_contact]
  dt_start date
  dt_end date
  fl_current boolean

  Note: 'N:N versionada (SCD Type 2 aplicado à bridge). Autorização no nível de CONTRATO (não por ativo). Vigência necessária: funcionário pode ser desautorizado sem o contrato encerrar (levantamento seção 8) — sem isso, auditoria de atendimento antigo não saberia quem estava autorizado naquela data.'
}

Table dim_asset_vehicle {
  sk_asset_vehicle int [pk]
  bk_asset_vehicle varchar [note: 'Chassi — identificador imutável, mesmo critério de CPF/CNPJ']
  plate varchar [note: 'Placa — Type 2 (peso jurídico/legal, exige reconstituição histórica)']
  model_desc varchar [note: 'Type 1 — atributo de fábrica']
  nr_year int [note: 'Type 1 — atributo de fábrica']
  color_desc varchar [note: 'Type 1 — baixo valor de reconstituição histórica, identificação visual operacional']
  fk_address int [ref: > dim_address.sk_address, note: 'Type 2']
  dt_start date
  dt_end date
  fl_current boolean

  Note: 'FECHADA (estrutura, 2026-08-09). SK versionada (múltiplas linhas por veículo ao longo do tempo). Tabela física separada de dim_asset_property — sem supertype/subtype comum.'
}

Table dim_asset_property {
  sk_asset_property int [pk]
  bk_asset_property varchar [note: 'PENDENTE — pendência de negócio: identificador único do imóvel (matrícula/IPTU/equivalente) a validar com o diretor']
  property_type_desc varchar [note: 'Type 2 — pode mudar por reclassificação cadastral']
  nr_area_m2 decimal [note: 'Type 1 provisório — condicionado a pendência de negócio sobre impacto em precificação']
  fk_address int [ref: > dim_address.sk_address, note: 'Type 2 — mesmo padrão de dim_client.fk_address']
  dt_start date
  dt_end date
  fl_current boolean

  Note: 'Estrutura fechada (2026-08-09). BK pendente de validação de negócio.'
}

Table bridge_contract_asset_vehicle {
  sk_contract_asset_vehicle int [pk]
  fk_contract int [ref: > dim_contract.sk_contract]
  fk_asset_vehicle int [ref: > dim_asset_vehicle.sk_asset_vehicle]
  dt_start date
  dt_end date
  fl_current boolean

  Note: '1:N versionada (SCD Type 2 aplicado à bridge). PF = 1:1, PJ = 1:N (frota). Vigência necessária: veículo pode entrar/sair da frota sem o contrato encerrar.'
}

Table bridge_contract_asset_property {
  sk_contract_asset_property int [pk]
  fk_contract int [ref: > dim_contract.sk_contract]
  fk_asset_property int [ref: > dim_asset_property.sk_asset_property]
  dt_start date
  dt_end date
  fl_current boolean

  Note: '1:N versionada (SCD Type 2 aplicado à bridge), mesma estrutura de bridge_contract_asset_vehicle.'
}

Table dim_service_type {
  sk_service_type int [pk]
  bk_service_type varchar [note: 'PENDENTE de definição']
  category_desc varchar [note: 'Residencial / Automotivo']
  urgency_desc varchar [note: 'Emergencial / Agendado']

  Note: 'Estrutura fechada. SCD ainda pendente de decisão.'
}

Table dim_period {
  period_id int [pk]
  dt_date date
  nr_day_of_week int
  nr_day int
  nr_month int
  nr_year int
  fl_holiday boolean
  fl_weekend boolean

  Note: 'Atributos previstos — fechamento formal ainda pendente.'
}

// ---------------------------
// FATOS
// ---------------------------

Table fact_attendance {
  sk_attendance int [pk]
  bk_attendance varchar [note: 'Elo oficial entre as 3 fatos — ex: 123456/1']
  analyst_id int
  client_id int [ref: > dim_client.sk_client]
  fk_contract int [ref: > dim_contract.sk_contract, note: 'Versão vigente do contrato no momento do atendimento — evita inferência ambígua quando o cliente tem múltiplos contratos ativos']
  service_id int
  service_type_id int [ref: > dim_service_type.sk_service_type]
  modality_id int
  provider_id int
  fk_address_origin int [ref: > dim_address.sk_address]
  fk_address_destination int [ref: > dim_address.sk_address]
  ds_address_origin varchar [note: 'degenerate dimension: rua+número']
  ds_address_destination varchar [note: 'degenerate dimension: rua+número']
  fl_accepted boolean
  id_chnl_accept int
  id_dt_open int
  hr_open time
  id_dt_accept int
  hr_accept time
  arrival_estimated int
  id_dt_arrival int
  hr_arrival time
  id_dt_conclusion int
  hr_conclusion time
  nr_nps int
  nr_contract_attendance_seq int [note: 'Sequência do atendimento na janela de contagem do contrato (ex.: 4º acionamento do ano). Calculada na Silver via window function particionada por fk_contract — mesmo padrão de TMA/TMC. Alimenta comparação contra dim_plan_coverage.limit_value (tipo QTD_ANUAL) na Gold. Janela confirmada com o diretor: aniversário do contrato, não ano-calendário.']
  fk_asset_vehicle int [ref: > dim_asset_vehicle.sk_asset_vehicle, note: 'Nullable por design de domínio — preenchida só quando dim_service_type.category_desc = Automotivo']
  fk_asset_property int [ref: > dim_asset_property.sk_asset_property, note: 'Nullable por design de domínio — preenchida só quando dim_service_type.category_desc = Residencial']

  Note: 'FECHADA. Grão: 1 linha por serviço prestado dentro de um chamado. TMA/TMC calculados na Silver (imutáveis). fk_contract adicionado em 2026-07-06 para eliminar ambiguidade de qual contrato cobriu o atendimento. fk_asset_vehicle/fk_asset_property adicionados em 2026-08-09 — duas FKs nullable por subtipo em vez de FK polimórfica única, por restrição do motor Tabular do Power BI (relacionamento fixo).'
}

Table fact_interaction {
  bk_attendance varchar [note: 'FK conceitual -> fact_attendance']

  Note: 'Grão fechado (1 linha por interação analista/setor). Colunas ainda não detalhadas — placeholder no diagrama.'
}

Table fact_financial {
  bk_attendance varchar [note: 'FK conceitual -> fact_attendance']

  Note: 'Regras fechadas (tipo_lancamento, vl_expected/realized/difference). Componentes (mão de obra, km, adicionais) ainda não detalhados — placeholder no diagrama.'
}
