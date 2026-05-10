# Arquitetura de Relatórios com Datas Relativas no Looker Studio
## Guia Técnico Profissional para Métricas Comparativas Dinâmicas

---

## 1. ENTENDER O PROBLEMA REAL

### O Desafio Fundamental
Looker Studio é uma ferramenta de **visualização**, não de ETL. Ele não tem:
- Variáveis globais persistentes
- Lógica condicional avançada entre elementos
- Recálculo automático de períodos relativos na fonte de dados

**A data selecionada no filtro é apenas um filtro de UI** — não ativa magicamente a transformação de dados que você precisa.

---

## 2. ARQUITETURA RECOMENDADA (4 ABORDAGENS)

### ⭐ ABORDAGEM 1: BigQuery com Tabelas Pre-calculadas (RECOMENDADO PARA PRODUÇÃO)

**Filosofia:** A transformação acontece **ANTES** do Looker Studio

```
ETL no BigQuery (diário/horário)
         ↓
Tabelas pré-calculadas por período
         ↓
Looker Studio (apenas visualiza)
```

**Vantagens:**
- ✅ Performance máxima (dados já agregados)
- ✅ Funciona com Looker Studio nativo
- ✅ Escalável para milhões de registros
- ✅ Reutilizável em outras ferramentas
- ✅ Auditável e versionável
- ✅ Sem limitações de campos calculados

**Desvantagens:**
- ❌ Infraestrutura mais complexa
- ❌ Custo de processamento
- ❌ Latência (dados não são 100% em tempo real)

**Implementação:**

```sql
-- ESTRATÉGIA 1: Tabela Estrelada (Star Schema)
-- Tabela: analytics.facts_daily_metrics

CREATE OR REPLACE TABLE `projeto.dataset.facts_daily_metrics` AS
WITH base AS (
  SELECT
    DATE(timestamp) as data,
    user_id,
    session_id,
    event_value,
    event_category
  FROM `projeto.dataset.events`
  WHERE DATE(timestamp) >= DATE_SUB(CURRENT_DATE(), INTERVAL 400 DAY)
)
SELECT
  data,
  DATE_SUB(data, INTERVAL 0 DAY) as data_d0,
  DATE_SUB(data, INTERVAL 1 DAY) as data_d1,
  DATE_SUB(data, INTERVAL 7 DAY) as data_d7,
  DATE_SUB(data, INTERVAL 30 DAY) as data_d30,
  DATE_SUB(data, INTERVAL 365 DAY) as data_d365,
  COUNT(DISTINCT user_id) as users,
  COUNT(DISTINCT session_id) as sessions,
  SUM(event_value) as revenue,
  COUNT(*) as events
FROM base
GROUP BY data;

-- ESTRATÉGIA 2: Tabela de Períodos Comparativos
-- Tabela: analytics.comparative_periods

CREATE OR REPLACE TABLE `projeto.dataset.comparative_periods` AS
WITH periods AS (
  SELECT
    DATE_SUB(CURRENT_DATE(), INTERVAL 0 DAY) as reference_date,
    'today' as period_type
  UNION ALL
  SELECT DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY), 'd1'
  UNION ALL
  SELECT DATE_SUB(CURRENT_DATE(), INTERVAL 2 DAY), 'd2'
  UNION ALL
  SELECT DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY), 'd7'
  UNION ALL
  SELECT DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY), 'd30'
),
metrics AS (
  SELECT
    reference_date,
    period_type,
    COUNT(DISTINCT user_id) as users,
    COUNT(DISTINCT session_id) as sessions,
    SUM(event_value) as revenue
  FROM `projeto.dataset.events` e
  INNER JOIN periods p 
    ON DATE(e.timestamp) = p.reference_date
  GROUP BY reference_date, period_type
)
SELECT * FROM metrics;

-- ESTRATÉGIA 3: Blend Pronto (mais simples no Looker Studio)
-- Consulta 1: Métricas do período selecionado
CREATE OR REPLACE TABLE `projeto.dataset.metrics_selected_period` AS
SELECT
  DATE(timestamp) as data,
  'selected' as period_type,
  COUNT(DISTINCT user_id) as users,
  SUM(event_value) as revenue
FROM `projeto.dataset.events`
GROUP BY DATE(timestamp);

-- Consulta 2: Métricas do período comparativo (D-7)
CREATE OR REPLACE TABLE `projeto.dataset.metrics_d7` AS
SELECT
  DATE_ADD(DATE(timestamp), INTERVAL 7 DAY) as data,
  'd7' as period_type,
  COUNT(DISTINCT user_id) as users,
  SUM(event_value) as revenue
FROM `projeto.dataset.events`
GROUP BY DATE(timestamp);
```

**No Looker Studio:**
1. Conectar ambas as fontes (blend)
2. Criar chave de join: `data` + `period_type`
3. Adicionar filtro simples de data
4. Visualizar no gráfico

---

### ⭐ ABORDAGEM 2: Campos Calculados + Google Sheets de Lookup (INTERMEDIÁRIA)

**Filosofia:** Looker Studio calcula, mas com ajuda estruturada

**Caso de Uso:** Datasets até ~500K linhas

```
Google Sheets (tabela de dimensões)
         ↓
Blend com sua fonte de dados
         ↓
Campos calculados estruturados
         ↓
Looker Studio (visualiza)
```

**Exemplo: Tabela Lookup no Google Sheets**

```
| data_selecionada | d1_value | d7_value | d30_value | yoy_value |
|------------------|----------|----------|-----------|-----------|
| 2024-01-15       | 2024-01-14| 2024-01-08| 2023-12-16| 2023-01-15|
| 2024-01-16       | 2024-01-15| 2024-01-09| 2023-12-17| 2023-01-16|
```

**Campo Calculado no Looker Studio:**

```javascript
// Campo: reference_date_mapped
CASE
  WHEN EXACT(DATE_FIELD, DATE_SELECIONADA) THEN "current"
  WHEN EXACT(DATE_FIELD, D1_LOOKUP) THEN "d1"
  WHEN EXACT(DATE_FIELD, D7_LOOKUP) THEN "d7"
  WHEN EXACT(DATE_FIELD, D30_LOOKUP) THEN "d30"
  ELSE "other"
END

// Campo: period_revenue
CASE
  WHEN reference_date_mapped = "current" THEN REVENUE
  WHEN reference_date_mapped = "d1" THEN REVENUE * 0.85 // exemplo
  WHEN reference_date_mapped = "d7" THEN REVENUE * 0.90
  ELSE 0
END
```

**Limitações:**
- Lento com muitos registros
- Difícil de manter
- Risco de desincronização

---

### ⭐ ABORDAGEM 3: Data Studio Nativo com Parâmetros (RÁPIDO, LIMITADO)

**Para quando você NÃO pode usar BigQuery**

```javascript
// Campo calculado: days_back
DATE_DIFF(REFERENCE_DATE, DATA_SELECIONADA, DAY)

// Campo: period_label
CASE
  WHEN days_back >= 0 AND days_back < 1 THEN "Hoje"
  WHEN days_back >= 1 AND days_back < 2 THEN "D-1"
  WHEN days_back >= 7 AND days_back < 8 THEN "D-7"
  WHEN days_back >= 30 AND days_back < 31 THEN "D-30"
  ELSE "Outro"
END

// Campo: is_target_period
DATE_DIFF(REFERENCE_DATE, DATA_SELECIONADA, DAY) < 1

// Campo: is_d7_period
DATE_DIFF(REFERENCE_DATE, DATA_SELECIONADA, DAY) >= 6 AND
DATE_DIFF(REFERENCE_DATE, DATA_SELECIONADA, DAY) <= 8
```

**Onde usar:**
- Dados pequenos (<100K linhas)
- Demo/prototipagem rápida
- Relatórios internos

**NÃO usar para:**
- Produção com grandes volumes
- Múltiplas métricas complexas
- Comparações com muitos períodos

---

### ⭐ ABORDAGEM 4: Looker (NÃO DATA STUDIO) com LookML

**Se você tem acesso ao Looker completo:**

```lookml
# looker_model.view.periods.view
view: periods {
  dimension: reference_date {
    type: date
    sql: {% parameter select_date %} ;;
  }

  dimension: period_type {
    type: string
    sql: CASE
      WHEN DATE_DIFF(${created_date}, ${reference_date}, DAY) = 0 THEN 'D0'
      WHEN DATE_DIFF(${created_date}, ${reference_date}, DAY) = 1 THEN 'D1'
      WHEN DATE_DIFF(${created_date}, ${reference_date}, DAY) = 7 THEN 'D7'
      WHEN DATE_DIFF(${created_date}, ${reference_date}, DAY) = 30 THEN 'D30'
    END ;;
  }

  parameter: select_date {
    type: date
  }
}
```

**Vantagens:**
- Muito mais poderoso que Data Studio
- LookML é versionável
- Cache gerenciado automaticamente

**Desvantagens:**
- Requer infraestrutura Looker
- Curva de aprendizado maior

---

## 3. IMPLEMENTAÇÃO PASSO A PASSO (ABORDAGEM RECOMENDADA: BigQuery)

### PASSO 1: Preparar Dados no BigQuery

```sql
-- Script: setup_comparative_analytics.sql
-- Executar diariamente via Cloud Scheduler

DECLARE reference_date DATE DEFAULT CURRENT_DATE();

-- Tabela fato com todos os períodos
CREATE OR REPLACE TABLE `seu_projeto.analytics.daily_metrics_comparative` AS
WITH daily_data AS (
  SELECT
    DATE(event_timestamp) as metric_date,
    user_pseudo_id,
    SUM(CAST(event_value AS FLOAT64)) as event_value,
    COUNT(*) as event_count,
    COUNT(DISTINCT session_id) as session_count
  FROM `seu_projeto.analytics_1.events_*`
  WHERE _TABLE_SUFFIX = FORMAT_DATE('%Y%m%d', reference_date)
  GROUP BY metric_date, user_pseudo_id
),
aggregated AS (
  SELECT
    metric_date,
    COUNT(DISTINCT user_pseudo_id) as users,
    SUM(session_count) as sessions,
    SUM(event_value) as revenue,
    SUM(event_count) as events
  FROM daily_data
  GROUP BY metric_date
),
with_periods AS (
  SELECT
    metric_date,
    DATE_SUB(reference_date, INTERVAL 0 DAY) as reference_date,
    DATE_DIFF(reference_date, metric_date, DAY) as days_before_ref,
    CASE
      WHEN DATE_DIFF(reference_date, metric_date, DAY) = 0 THEN 'D0'
      WHEN DATE_DIFF(reference_date, metric_date, DAY) = 1 THEN 'D1'
      WHEN DATE_DIFF(reference_date, metric_date, DAY) = 2 THEN 'D2'
      WHEN DATE_DIFF(reference_date, metric_date, DAY) = 7 THEN 'D7'
      WHEN DATE_DIFF(reference_date, metric_date, DAY) = 30 THEN 'D30'
    END as period_label,
    users,
    sessions,
    revenue,
    events
  FROM aggregated
  CROSS JOIN (SELECT reference_date)
)
SELECT * FROM with_periods
WHERE days_before_ref <= 365;

-- Índices/Clustering para performance
CLUSTER BY metric_date, reference_date;

-- Tabela de lookup para o Looker Studio
CREATE OR REPLACE TABLE `seu_projeto.analytics.period_mapping` AS
SELECT DISTINCT
  metric_date,
  DATE_SUB(CURRENT_DATE(), INTERVAL 0 DAY) as ref_date_d0,
  DATE_SUB(CURRENT_DATE(), INTERVAL 1 DAY) as ref_date_d1,
  DATE_SUB(CURRENT_DATE(), INTERVAL 7 DAY) as ref_date_d7,
  DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY) as ref_date_d30
FROM `seu_projeto.analytics.daily_metrics_comparative`
WHERE metric_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 400 DAY);
```

### PASSO 2: Configurar Cloud Scheduler

```bash
# gcloud command para automatizar
gcloud scheduler jobs create bigquery hourly-metrics \
  --schedule="0 1 * * *" \
  --time-zone="America/Sao_Paulo" \
  --location=us-central1 \
  --bigquery-body='{
    "query":"EXECUTE IMMEDIATE (SELECT setup_comparative_analytics.sql)",
    "useLegacySql":false
  }'
```

### PASSO 3: Conectar ao Looker Studio

**Opção A: Blend com Two Tables**

```
Tabela 1: daily_metrics_comparative (período selecionado)
Tabela 2: daily_metrics_comparative (período comparativo)

Join: metric_date (adaptado via campos calculados)
```

**Query 1: Current Period**
```sql
SELECT
  metric_date as data,
  users,
  revenue,
  sessions,
  'current' as period_type
FROM `seu_projeto.analytics.daily_metrics_comparative`
WHERE metric_date >= @data_inicio AND metric_date <= @data_fim
```

**Query 2: Comparative Periods**
```sql
SELECT
  DATE_ADD(metric_date, INTERVAL 7 DAY) as data,
  users,
  revenue,
  sessions,
  'comparative_d7' as period_type
FROM `seu_projeto.analytics.daily_metrics_comparative`
WHERE metric_date >= DATE_SUB(@data_inicio, INTERVAL 7 DAY)
  AND metric_date <= DATE_SUB(@data_fim, INTERVAL 7 DAY)
```

### PASSO 4: Campos Calculados no Looker Studio

```javascript
// Campo 1: Display Name for Period
CASE
  WHEN period_type = 'current' THEN "Período Selecionado"
  WHEN period_type = 'comparative_d7' THEN "Mesmo período (7 dias antes)"
  WHEN period_type = 'comparative_d30' THEN "Mesmo período (30 dias antes)"
END

// Campo 2: Sort Order (para gráficos)
CASE
  WHEN period_type = 'current' THEN 1
  WHEN period_type = 'comparative_d7' THEN 2
  WHEN period_type = 'comparative_d30' THEN 3
END

// Campo 3: Variation % (comparação)
(SUM(revenue_current) - SUM(revenue_d7)) / SUM(revenue_d7) * 100

// Campo 4: Conditional Color
IF(Variation > 0, "#00C853", "#E53935")
```

### PASSO 5: Layout e Visualizações

**Card 1: Métrica Atual**
```
Métrica: SUM(revenue)
Filtro: period_type = 'current'
Data Range: 7 dias selecionado
```

**Card 2: D-7 Comparison**
```
Métrica: SUM(revenue)
Filtro: period_type = 'comparative_d7'
Mostrar variação %: ✓
```

**Card 3: Gráfico Linha Comparativa**
```
Eixo X: data
Eixo Y: revenue
Dimensão: period_type
Cores: Verde (current), Cinza (comparative)
```

---

## 4. LIMITAÇÕES DO LOOKER STUDIO (E COMO CONTORNAR)

| Limitação | Impacto | Solução |
|-----------|--------|--------|
| Sem variáveis globais | Data selecionada não persiste entre elementos | BigQuery com tabela pre-calculada |
| Campos calculados lentos em volumes grandes | Timout em datasets > 10M linhas | Usar tabelas agregadas no BigQuery |
| Sem lógica IF/ELSE complexa | Não dá pra comparar 3+ períodos elegantemente | Blend com queries separadas |
| Parâmetros só filtram, não transformam | Não consegue ajustar datas automaticamente | Ajustar dados no BigQuery |
| Cache limitado (12h) | Dados podem desatualizar | Refresh manual ou via integração |
| Blends têm limite de 10M linhas | Performance piora muito | Dados já agregados (fato, não detalhe) |

---

## 5. PADRÃO ARQUITETURAL PROFISSIONAL (ESCALÁVEL)

### Modelo de 3 Camadas

```
┌─────────────────────────────────────────┐
│          LOOKER STUDIO (Viz)            │
│   - Filtros simples de data             │
│   - Cards e gráficos                    │
│   - Blends básicos                      │
└─────────────┬───────────────────────────┘
              │
┌─────────────▼───────────────────────────┐
│   CAMADA DE CONSULTA (Curated)          │
│   - Tabelas agregadas                   │
│   - Joins pré-calculados                │
│   - Dimensões normalizadas              │
└─────────────┬───────────────────────────┘
              │
┌─────────────▼───────────────────────────┐
│     CAMADA DE TRANSFORMAÇÃO (dbt)       │
│   - Modelos staging                     │
│   - Enriquecimento de dados             │
│   - Testes de integridade               │
└─────────────┬───────────────────────────┘
              │
┌─────────────▼───────────────────────────┐
│     CAMADA RAW (BigQuery)               │
│   - Dados brutos (GA4, CRM, etc)        │
│   - Histórico completo                  │
│   - Não transformado                    │
└─────────────────────────────────────────┘
```

### Exemplo com dbt

```yaml
# models/analytics/fct_daily_metrics.yml
models:
  - name: fct_daily_metrics
    description: "Fato diária com períodos comparativos"
    columns:
      - name: metric_date
        description: "Data da métrica"
        tests:
          - not_null
          - unique
      - name: period_label
        tests:
          - accepted_values:
              values: ['d0', 'd1', 'd7', 'd30']
    meta:
      owner: "analytics_team"
      refresh_interval: "daily"
```

```sql
# models/analytics/fct_daily_metrics.sql
{{ config(
    materialized='table',
    clustering=['metric_date', 'reference_date'],
    partition_by={
      "field": "metric_date",
      "data_type": "date",
      "granularity": "day"
    }
) }}

WITH raw_data AS (
  SELECT * FROM {{ ref('stg_events') }}
),
daily_agg AS (
  SELECT
    DATE(event_timestamp) as metric_date,
    COUNT(DISTINCT user_id) as users,
    SUM(event_value) as revenue
  FROM raw_data
  GROUP BY DATE(event_timestamp)
),
with_periods AS (
  SELECT
    *,
    CURRENT_DATE() as reference_date,
    DATE_DIFF(CURRENT_DATE(), metric_date, DAY) as days_back,
    CASE
      WHEN DATE_DIFF(CURRENT_DATE(), metric_date, DAY) = 0 THEN 'd0'
      WHEN DATE_DIFF(CURRENT_DATE(), metric_date, DAY) = 1 THEN 'd1'
      WHEN DATE_DIFF(CURRENT_DATE(), metric_date, DAY) = 7 THEN 'd7'
      WHEN DATE_DIFF(CURRENT_DATE(), metric_date, DAY) = 30 THEN 'd30'
    END as period_label
  FROM daily_agg
)
SELECT * FROM with_periods
```

---

## 6. EXEMPLOS REAIS DE FÓRMULAS

### Exemplo 1: Card com Variação Percentual

```javascript
// Campo: Revenue_Current
SUM(revenue)

// Campo: Revenue_D7
SUM(revenue)  // (com filtro period_type = 'd7')

// Campo: Variation_Pct
(SUM(revenue_current) - SUM(revenue_d7)) / 
SUM(revenue_d7) * 100

// Scorecard:
Valor Principal: Revenue_Current
Valor de Comparação: Variation_Pct
Formatação: 
  - Se > 0: Verde ✓ +X%
  - Se < 0: Vermelho ✗ X%
```

### Exemplo 2: Gráfico Linha com 3 Períodos

```
Dimensão: data
Métrica: revenue
Cor por: period_label (d0, d7, d30)

Estrutura:
  - Série 1 (Hoje): azul, linha contínua
  - Série 2 (D-7): cinza, linha tracejada
  - Série 3 (D-30): cinza claro, linha pontilhada
```

### Exemplo 3: Tabela com Ranking

```javascript
// Campo: Row_Number
ROW_NUMBER() OVER (
  PARTITION BY period_label 
  ORDER BY revenue DESC
)

// Mostrar: Top 10 produtos por período
// Dimensões: product_name, period_label
// Métrica: revenue, Row_Number
// Filtro: Row_Number <= 10
```

### Exemplo 4: Matriz Comparativa

```
           | Hoje   | D-7    | D-30   | Var %
-----------|--------|--------|--------|--------
Usuários   | 5.2K   | 4.8K   | 4.1K   | +8.3%
Receita    | R$125K | R$110K | R$95K  | +13.6%
Taxa Conv  | 2.3%   | 2.1%   | 1.9%   | +9.5%
```

---

## 7. BOAS PRÁTICAS E PADRÕES PROFISSIONAIS

### 1️⃣ Convenção de Nomenclatura
```
fct_daily_metrics        # Fatos agregados (tabela principal)
dim_date                 # Dimensão de data
dim_product              # Dimensão de produto
metric_*                 # Campos calculados de métrica
comparison_*             # Campos de comparação
```

### 2️⃣ Versionamento de Relatórios
```
/reports/
  ├── analytics_dashboard_v1.0.md
  ├── analytics_dashboard_v1.1.md (com comparações)
  ├── analytics_dashboard_v2.0.md (redesign)
  └── CHANGELOG.md
```

### 3️⃣ Governança de Dados
```
┌─ Quem pode editar? (analistas senior)
├─ Refresh automático? (sim, diário)
├─ SLA de dados? (máximo 1h de atraso)
└─ Testes de qualidade? (dbt tests)
```

### 4️⃣ Performance Checklist

- [ ] Agregações feitas no BigQuery (não no Looker)
- [ ] Índices/Clustering por período e data
- [ ] Partition por metric_date
- [ ] Limpar dados antigos (retenção 400 dias)
- [ ] Cache de tabelas materializadas
- [ ] Evitar cálculos em campos calculados (fazê-los no SQL)
- [ ] Blends usando tabelas pequenas (<500K linhas)
- [ ] Filtros simples (não lógica complexa)

---

## 8. ALTERNATIVAS MAIS ELEGANTES

### ❌ O que EVITAR:
- 15 scorecards iguais com filtros diferentes
- Campos calculados com 10+ linhas de CASE WHEN
- Tabelas lookup em Google Sheets
- Recálculo manual de períodos

### ✅ O que FAZER:

**Padrão 1: Unified Fact Table**
```sql
-- Uma tabela com TODAS as combinações de período
SELECT
  metric_date,
  period_label,  -- d0, d1, d7, d30
  users,
  revenue,
  sessions
FROM fct_daily_metrics
-- Filtrar no Looker Studio por period_label
```

**Padrão 2: Parameterized Views**
```sql
-- Usar variáveis SQL no Looker Studio
CREATE OR REPLACE VIEW comparative_metrics AS
WITH params AS (
  SELECT
    CAST(@metric_date AS DATE) as ref_date,
    ARRAY<STRUCT<label STRING, days_back INT64>>[
      STRUCT('today', 0),
      STRUCT('d7', 7),
      STRUCT('d30', 30),
      STRUCT('yoy', 365)
    ] as periods
)
SELECT * FROM ...
```

**Padrão 3: Self-Service com Sugestões**
```
Criar um dropdown único:
  📅 Selecione o Período: [Últimos 7 dias ▼]
     
Ao selecionar "Últimos 7 dias":
  • Automaticamente compara com 7 dias anteriores
  • Mostra YoY também
  • Atualiza todos os cards
```

---

## 9. CASO DE USO REAL: E-COMMERCE

### Contexto:
- Fonte: GA4 + Shopify API
- Volume: ~500K eventos/dia
- Métricas: Receita, Conversão, AOV, Taxa de Retorno

### Solução Implementada:

```sql
-- TABELA PRINCIPAL (BigQuery)
CREATE TABLE ecommerce.daily_metrics_comparative AS
WITH daily_sales AS (
  SELECT
    DATE(order_date) as data,
    SUM(order_value) as revenue,
    COUNT(DISTINCT order_id) as orders,
    COUNT(DISTINCT customer_id) as customers,
    SUM(CASE WHEN is_returning = 1 THEN 1 ELSE 0 END) / 
      COUNT(DISTINCT customer_id) as retention_rate
  FROM raw.orders
  WHERE order_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 400 DAY)
  GROUP BY data
),
expanded_periods AS (
  SELECT
    data,
    CURRENT_DATE() as reference_date,
    CASE
      WHEN DATE_DIFF(CURRENT_DATE(), data, DAY) = 0 THEN 'today'
      WHEN DATE_DIFF(CURRENT_DATE(), data, DAY) = 1 THEN 'd1'
      WHEN DATE_DIFF(CURRENT_DATE(), data, DAY) = 7 THEN 'd7'
      WHEN DATE_DIFF(CURRENT_DATE(), data, DAY) = 30 THEN 'd30'
      WHEN DATE_DIFF(CURRENT_DATE(), data, DAY) = 365 THEN 'yoy'
    END as period,
    revenue,
    orders,
    customers,
    retention_rate
  FROM daily_sales
)
SELECT * FROM expanded_periods
WHERE period IS NOT NULL;
```

```javascript
// LOOKER STUDIO - Cards

// Card 1: Receita de Hoje
Métrica: SUM(revenue)
Filtro: period = 'today'
Subtítulo: "Hoje"

// Card 2: Variação vs D-7
Métrica Principal: SUM(revenue) where period = 'today'
Métrica Comparada: SUM(revenue) where period = 'd7'
Mostrar Variação: Sim
Formato: +R$ 5.230 (+12.3%)

// Card 3: Gráfico de Receita (últimos 30 dias)
Dimensão: data
Métrica: revenue
Cor: period
Filtro: DATE_RANGE_FILTER (Looker nativo)
```

---

## 10. TROUBLESHOOTING COMUM

### ❌ Problema: Dados não sincronizam quando mudo a data

**Causa:** Blend não está funcionando corretamente
**Solução:**
```
1. Verificar chaves de join (ex: data exata, sem horários)
2. Certificar que ambas as tabelas têm a dimensão
3. Inner join → Left join (se necessário)
```

### ❌ Problema: Gráfico mostra valores duplicados

**Causa:** Blend está agrupando incorretamente
**Solução:**
```sql
-- Antes
SELECT data, users FROM fct_metrics

-- Depois (com agregação explícita)
SELECT
  data,
  period_label,
  SUM(users) as users
GROUP BY data, period_label
```

### ❌ Problema: Performance piora quando adiciono novo período

**Causa:** Muitas linhas no blend
**Solução:**
```
- Reduzir período de histórico (não manter 2 anos)
- Pré-agregar dados no BigQuery
- Usar tabelas particionadas
- Limpar cache do navegador
```

### ❌ Problema: Scorecard mostra valores errados

**Causa:** Campo calculado tem lógica bugada
**Solução:**
```javascript
// Verificar ordem de operações
// ✓ Correto
(SUM(A) - SUM(B)) / SUM(B) * 100

// ✗ Errado
SUM(A) - SUM(B) / SUM(B) * 100  // Divisão acontece antes da subtração
```

---

## 11. CHECKLIST DE IMPLEMENTAÇÃO

- [ ] **Planejamento**
  - [ ] Definir períodos comparativos (d1, d7, d30, yoy?)
  - [ ] Identificar métricas-chave
  - [ ] Desenhar fluxo de dados

- [ ] **BigQuery Setup**
  - [ ] Criar tabela fato com períodos
  - [ ] Adicionar clustering/partitioning
  - [ ] Testar performance (EXPLAIN)
  - [ ] Agendar refresh diário

- [ ] **Looker Studio**
  - [ ] Conectar fonte (BigQuery)
  - [ ] Criar blend (se necessário)
  - [ ] Testar campos calculados
  - [ ] Validar números (comparar com fonte)

- [ ] **Documentação**
  - [ ] README com explicação de períodos
  - [ ] Como filtrar/usar cada elemento
  - [ ] Manutenção e atualização

- [ ] **Validação**
  - [ ] Números batem com source (GA4/CRM)?
  - [ ] Comparações fazem sentido?
  - [ ] Performance aceitável?
  - [ ] Usuários conseguem usar sozinhos?

---

## 12. ROADMAP FUTURO

```
Fase 1 (Semana 1-2): Setup básico
├─ Tabela fato em BigQuery
├─ Conexão ao Looker Studio
└─ 3-4 cards iniciais

Fase 2 (Semana 3-4): Expansão
├─ Adicionar mais períodos (yoy)
├─ Gráficos comparativos
└─ Testes de qualidade (dbt)

Fase 3 (Semana 5-6): Otimização
├─ Automação via Cloud Scheduler
├─ Alertas de anomalias
└─ Dashboard executivo

Fase 4 (Semana 7+): Profissionalização
├─ Looker (ao invés de Data Studio)
├─ Governance e permissões
├─ SLA de dados
└─ Treinamento de usuários
```

---

## RESUMO EXECUTIVO

| Abordagem | Simplicidade | Performance | Escalabilidade | Recomendação |
|-----------|--------------|-------------|----------------|--------------|
| BigQuery Pre-calc | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ✅ **PRODUÇÃO** |
| Lookup Sheets | ⭐⭐⭐ | ⭐⭐ | ⭐ | Demo/Interno |
| Data Studio Nativo | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐ | Prototipagem |
| Looker (LookML) | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | **Enterprise** |

---

**Desenvolvido para:** Analistas BI e Web Analytics Sênior  
**Versão:** 2.0  
**Última atualização:** 2024  
**Autor:** Technical BI Architecture  
