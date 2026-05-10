# Implementação Prática: Datas Relativas no Looker Studio
## Código Completo e Pronto para Copiar/Colar

---

## PARTE 1: SQL COMPLETO PARA BIGQUERY

### 1.1 Setup Inicial - Criar Tabelas Base

```sql
-- ========================================
-- SCRIPT 1: Criar Tabela Fato com Períodos
-- ========================================
-- Executar uma vez para setup

CREATE OR REPLACE TABLE `seu_projeto.analytics.fct_daily_metrics_comparative` (
  metric_date DATE OPTIONS(description="Data da métrica"),
  reference_date DATE OPTIONS(description="Data de referência"),
  period_label STRING OPTIONS(description="d0, d1, d2, d7, d30, yoy"),
  days_back INT64 OPTIONS(description="Dias de diferença"),
  users INT64,
  sessions INT64,
  revenue FLOAT64,
  events INT64,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP()
)
PARTITION BY metric_date
CLUSTER BY reference_date, period_label
OPTIONS(
  description="Tabela fato com períodos comparativos",
  labels=[("domain", "analytics"), ("sensitivity", "public")]
);

-- Criar índices (simular com clustering)
-- Criar tabela de dimensão de data
CREATE OR REPLACE TABLE `seu_projeto.analytics.dim_date` AS
WITH RECURSIVE dates AS (
  SELECT DATE_SUB(CURRENT_DATE(), INTERVAL 400 DAY) as date_key
  UNION ALL
  SELECT DATE_ADD(date_key, INTERVAL 1 DAY)
  FROM dates
  WHERE date_key < CURRENT_DATE()
)
SELECT
  date_key,
  EXTRACT(YEAR FROM date_key) as year,
  EXTRACT(MONTH FROM date_key) as month,
  EXTRACT(WEEK FROM date_key) as week,
  EXTRACT(DAYOFWEEK FROM date_key) as day_of_week,
  FORMAT_DATE('%Y-%m-%d', date_key) as date_formatted,
  FORMAT_DATE('%A', date_key) as day_name,
  CASE
    WHEN EXTRACT(DAYOFWEEK FROM date_key) IN (1, 7) THEN 'Weekend'
    ELSE 'Weekday'
  END as weekend_flag
FROM dates
ORDER BY date_key;
```

### 1.2 Script de Transformação Diária (RECOMENDADO)

```sql
-- ========================================
-- SCRIPT 2: Transformação Diária (Cloud Scheduler)
-- ========================================
-- Frequência: Diariamente às 01:00 AM (São Paulo)
-- Executar como: Cloud Function ou BigQuery Scheduled Query

DECLARE processing_date DATE DEFAULT CURRENT_DATE() - 1;  -- Ontem
DECLARE reference_date DATE DEFAULT CURRENT_DATE();

BEGIN
  -- Limpar dados da data que será reprocessada
  DELETE FROM `seu_projeto.analytics.fct_daily_metrics_comparative`
  WHERE metric_date = processing_date
    AND reference_date >= DATE_SUB(processing_date, INTERVAL 365 DAY);

  -- Inserir novo lote
  INSERT INTO `seu_projeto.analytics.fct_daily_metrics_comparative`
  WITH raw_events AS (
    SELECT
      DATE(TIMESTAMP_MILLIS(event_date * 1000)) as metric_date,
      user_pseudo_id,
      SUM(CAST(event_value_in_usd AS FLOAT64)) as event_value,
      COUNT(*) as event_count,
      COUNT(DISTINCT session_id) as session_count
    FROM `seu_projeto.analytics_1.events_*`
    WHERE _TABLE_SUFFIX = FORMAT_DATE('%Y%m%d', processing_date)
      AND event_name NOT IN ('page_view', 'scroll')  -- Filtrar ruído se necessário
    GROUP BY metric_date, user_pseudo_id
  ),
  daily_agg AS (
    SELECT
      metric_date,
      COUNT(DISTINCT user_pseudo_id) as users,
      SUM(session_count) as sessions,
      SUM(event_value) as revenue,
      SUM(event_count) as events
    FROM raw_events
    GROUP BY metric_date
  ),
  with_periods AS (
    SELECT
      metric_date,
      reference_date,
      DATE_DIFF(reference_date, metric_date, DAY) as days_back,
      CASE
        WHEN DATE_DIFF(reference_date, metric_date, DAY) = 0 THEN 'd0'
        WHEN DATE_DIFF(reference_date, metric_date, DAY) = 1 THEN 'd1'
        WHEN DATE_DIFF(reference_date, metric_date, DAY) = 2 THEN 'd2'
        WHEN DATE_DIFF(reference_date, metric_date, DAY) = 7 THEN 'd7'
        WHEN DATE_DIFF(reference_date, metric_date, DAY) = 30 THEN 'd30'
        WHEN DATE_DIFF(reference_date, metric_date, DAY) = 365 THEN 'yoy'
        ELSE NULL  -- Ignorar períodos que não são targets
      END as period_label,
      users,
      sessions,
      revenue,
      events
    FROM daily_agg
    CROSS JOIN (SELECT reference_date)
  )
  SELECT * FROM with_periods
  WHERE period_label IS NOT NULL
    AND metric_date >= DATE_SUB(reference_date, INTERVAL 400 DAY);

  -- Vacuum (limpeza de dados antigos)
  DELETE FROM `seu_projeto.analytics.fct_daily_metrics_comparative`
  WHERE metric_date < DATE_SUB(CURRENT_DATE(), INTERVAL 400 DAY);

END;
```

### 1.3 Query de Verificação/Debug

```sql
-- ========================================
-- SCRIPT 3: Validação de Dados
-- ========================================

-- Verificar quais períodos foram calculados
SELECT
  reference_date,
  period_label,
  COUNT(*) as row_count,
  SUM(users) as total_users,
  SUM(revenue) as total_revenue
FROM `seu_projeto.analytics.fct_daily_metrics_comparative`
GROUP BY reference_date, period_label
ORDER BY reference_date DESC, period_label;

-- Comparar períodos
WITH metrics_by_period AS (
  SELECT
    period_label,
    SUM(users) as total_users,
    SUM(revenue) as total_revenue,
    COUNT(DISTINCT metric_date) as days_count
  FROM `seu_projeto.analytics.fct_daily_metrics_comparative`
  WHERE reference_date = CURRENT_DATE()
  GROUP BY period_label
)
SELECT
  period_label,
  total_users,
  total_revenue,
  days_count,
  ROUND(total_revenue / days_count, 2) as avg_daily_revenue,
  ROUND(
    (total_revenue - LAG(total_revenue) OVER (ORDER BY period_label)) 
    / LAG(total_revenue) OVER (ORDER BY period_label) * 100, 
    2
  ) as variation_pct
FROM metrics_by_period;

-- Detectar gaps (dias faltando)
WITH expected_dates AS (
  SELECT DISTINCT metric_date
  FROM `seu_projeto.analytics.dim_date`
  WHERE metric_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
),
actual_dates AS (
  SELECT DISTINCT metric_date
  FROM `seu_projeto.analytics.fct_daily_metrics_comparative`
  WHERE reference_date = CURRENT_DATE()
)
SELECT
  expected_dates.metric_date,
  CASE WHEN actual_dates.metric_date IS NULL THEN 'MISSING' ELSE 'OK' END as status
FROM expected_dates
LEFT JOIN actual_dates ON expected_dates.metric_date = actual_dates.metric_date
WHERE actual_dates.metric_date IS NULL;
```

---

## PARTE 2: SETUP NO LOOKER STUDIO

### 2.1 Conectar Fonte de Dados

**Passo 1:** Novo Relatório → Conectar Dados
```
Fonte: BigQuery
Projeto: seu_projeto
Dataset: analytics
Tabela: fct_daily_metrics_comparative
Autorizar acesso
```

**Passo 2:** Alternativamente, usar como visualização (SQL customizado)

```sql
-- Query para Looker Studio (se quiser fazer blend)
SELECT
  metric_date,
  reference_date,
  period_label,
  users,
  sessions,
  revenue,
  events
FROM `seu_projeto.analytics.fct_daily_metrics_comparative`
WHERE reference_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 365 DAY)
ORDER BY metric_date DESC
```

### 2.2 Criar Filtro de Data (Opcional - apenas para contexto)

```
Nome: Date Range Filter
Tipo: Date Range
Aplicar a: todas as visualizações
Padrão: Últimos 30 dias

Configuração:
  ☑ Incluir parâmetro
  ☑ Mostrar filtro no relatório
  ☑ Mostrar todos os valores
```

---

## PARTE 3: CAMPOS CALCULADOS

### 3.1 Campos Essenciais

```javascript
// ========================================
// CAMPO 1: Period Label (Display)
// ========================================
CASE
  WHEN period_label = 'd0' THEN 'Hoje'
  WHEN period_label = 'd1' THEN 'Ontem (D-1)'
  WHEN period_label = 'd2' THEN '2 dias atrás (D-2)'
  WHEN period_label = 'd7' THEN '7 dias atrás (D-7)'
  WHEN period_label = 'd30' THEN '30 dias atrás (D-30)'
  WHEN period_label = 'yoy' THEN 'Ano passado (YoY)'
  ELSE period_label
END

// Nome da métrica: period_label_display

// ========================================
// CAMPO 2: Sort Order (para gráficos)
// ========================================
CASE
  WHEN period_label = 'd0' THEN 1
  WHEN period_label = 'd1' THEN 2
  WHEN period_label = 'd2' THEN 3
  WHEN period_label = 'd7' THEN 4
  WHEN period_label = 'd30' THEN 5
  WHEN period_label = 'yoy' THEN 6
  ELSE 99
END

// Nome: period_sort_order

// ========================================
// CAMPO 3: Cor por Período
// ========================================
CASE
  WHEN period_label = 'd0' THEN '#1f77b4'   // Azul escuro
  WHEN period_label = 'd7' THEN '#ff7f0e'   // Laranja
  WHEN period_label = 'd30' THEN '#2ca02c'  // Verde
  WHEN period_label = 'yoy' THEN '#d62728'  // Vermelho
  ELSE '#7f7f7f'                            // Cinza
END

// Nome: period_color

// ========================================
// CAMPO 4: Revenue Formatted
// ========================================
CONCAT(
  'R$ ',
  TEXT(ROUND(revenue, 2), '#,##0.00')
)

// Nome: revenue_formatted

// ========================================
// CAMPO 5: Users com Separador de Milhares
// ========================================
TEXT(ROUND(users), '#,##0')

// Nome: users_formatted

// ========================================
// CAMPO 6: Variação % vs D-7
// ========================================
// ⚠️ IMPORTANTE: Use apenas em SCORECARDS com agregação!

// Se período = 'd0':
(SUM(revenue WHERE period_label = 'd0') - 
 SUM(revenue WHERE period_label = 'd7')) / 
SUM(revenue WHERE period_label = 'd7') * 100

// Nome: variation_vs_d7_pct

// ========================================
// CAMPO 7: Indicador de Tendência
// ========================================
CASE
  WHEN variation_vs_d7_pct > 5 THEN '📈 Crescimento Forte'
  WHEN variation_vs_d7_pct > 0 THEN '↗️ Crescimento'
  WHEN variation_vs_d7_pct < -5 THEN '📉 Queda Forte'
  WHEN variation_vs_d7_pct < 0 THEN '↘️ Queda'
  ELSE '➡️ Estável'
END

// Nome: trend_indicator

// ========================================
// CAMPO 8: Cor Condicional (Verde/Vermelho)
// ========================================
CASE
  WHEN variation_vs_d7_pct > 0 THEN '#00C853'
  WHEN variation_vs_d7_pct < 0 THEN '#E53935'
  ELSE '#9E9E9E'
END

// Nome: variation_color
```

### 3.2 Campos Avançados (Opcional)

```javascript
// ========================================
// CAMPO 9: Ranking de Dias
// ========================================
RANK() OVER (PARTITION BY period_label ORDER BY revenue DESC)

// Nome: rank_by_period

// ========================================
// CAMPO 10: Diferença Absoluta
// ========================================
// ⚠️ Usar em scorecards com filtro individual!

revenue - (SELECT SUM(revenue) 
           FROM tabela 
           WHERE period_label = 'd7')

// Nome: absolute_difference

// ========================================
// CAMPO 11: Comparação com Múltiplos Períodos
// ========================================
CASE
  WHEN period_label = 'd0' THEN revenue
  WHEN period_label = 'd7' THEN revenue * 0.95  // Ajuste exemplo
  WHEN period_label = 'd30' THEN revenue * 0.88
  ELSE revenue
END

// Nome: normalized_revenue

// ========================================
// CAMPO 12: Flag de Período Especial
// ========================================
CASE
  WHEN period_label IN ('d0', 'd1') THEN 'Recent'
  WHEN period_label = 'd7' THEN 'Last Week'
  WHEN period_label = 'd30' THEN 'Last Month'
  WHEN period_label = 'yoy' THEN 'Year Ago'
  ELSE 'Other'
END

// Nome: period_group
```

---

## PARTE 4: LAYOUT RECOMENDADO

### 4.1 Estrutura do Relatório

```
┌─────────────────────────────────────────────┐
│  HEADER: Analytics Dashboard - Períodos     │
│  Últimos 90 dias                             │
└─────────────────────────────────────────────┘

┌─── LINHA 1: KPIs PRINCIPAIS ───────────────┐
│                                              │
│  ┌──────────────┐  ┌──────────────┐         │
│  │  Receita     │  │  Usuários    │         │
│  │  Hoje: R$ 5K │  │  Hoje: 1.2K  │         │
│  │  vs D-7: +12%│  │  vs D-7: +8% │         │
│  └──────────────┘  └──────────────┘         │
│                                              │
│  ┌──────────────┐  ┌──────────────┐         │
│  │  Sessões     │  │  Eventos     │         │
│  │  Hoje: 850   │  │  Hoje: 3.4K  │         │
│  │  vs D-7: +5% │  │  vs D-7: +9% │         │
│  └──────────────┘  └──────────────┘         │
│                                              │
└────────────────────────────────────────────┘

┌─── LINHA 2: GRÁFICO COMPARATIVO ──────────┐
│                                              │
│  Receita - Últimos 30 dias                  │
│  (Comparando: Hoje vs D-7 vs D-30)          │
│                                              │
│  [Gráfico Linha com 3 séries]               │
│   Azul: Hoje                                │
│   Laranja: D-7                              │
│   Verde: D-30                               │
│                                              │
└────────────────────────────────────────────┘

┌─── LINHA 3: TABELA DETALHADA ─────────────┐
│                                              │
│  Métricas por Período                       │
│                                              │
│  Período  │ Receita │ Usuários │ Variação │
│  ─────────┼─────────┼──────────┼──────────│
│  Hoje     │ R$5.2K  │ 1.2K     │ —        │
│  D-1      │ R$4.8K  │ 1.1K     │ +8.3%    │
│  D-7      │ R$4.6K  │ 1.0K     │ +13.0%   │
│  D-30     │ R$3.8K  │ 0.9K     │ +36.8%   │
│  YoY      │ R$2.1K  │ 0.6K     │ +147%    │
│                                              │
└────────────────────────────────────────────┘
```

### 4.2 Configuração de Cada Elemento

#### Scorecard: Receita - Hoje

```
Propriedades:
  Métrica Principal: SUM(revenue)
  Filtro: period_label = 'd0'
  
  Comparação:
    ☑ Mostrar comparação
    Métrica: SUM(revenue) where period_label = 'd7'
    Tipo: Variação percentual
    Formato: Mostrar tendência (↑/↓)
    
  Estilo:
    Tamanho do texto: Grande (40pt)
    Mostrar cores: Verde se positivo, Vermelho se negativo
    Sufixo: "% vs D-7"

Interatividade:
  ☑ Usar como filtro
  ☑ Permitir seleção
```

#### Gráfico: Receita - Últimos 30 Dias

```
Tipo: Série de Linhas

Dimensão:
  Eixo X: metric_date
  
Métricas:
  Eixo Y: SUM(revenue)
  
Breakout:
  Period: period_label
  
Filtros:
  period_label in ('d0', 'd7', 'd30')
  metric_date >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
  
Cores:
  d0: #1f77b4 (Azul)
  d7: #ff7f0e (Laranja)
  d30: #2ca02c (Verde)
  
Estilos:
  Grossura da linha: 2px
  Mostrar pontos: Sim
  Suavizar linhas: Não
```

#### Tabela: Métricas por Período

```
Dimensões:
  Coluna 1: period_label_display
  
Métricas:
  Coluna 2: SUM(revenue)
  Coluna 3: SUM(users)
  Coluna 4: variation_vs_d7_pct (com cor condicional)
  
Ordenação:
  Por: period_sort_order (crescente)
  
Formatação:
  Coluna de Receita: Formatação de moeda (R$)
  Coluna de Variação: Decimal com 2 casas + símbolo %
  
Filtros (opcional):
  period_label in ('d0', 'd1', 'd7', 'd30', 'yoy')
```

---

## PARTE 5: TESTES E VALIDAÇÃO

### 5.1 Checklist de Testes

```
TESTE 1: Números Corretos?
[ ] Receita de hoje bate com GA4?
[ ] Usuários combinam com fonte?
[ ] D-1 = D-0 menos 1 dia?
[ ] D-7 = D-0 menos 7 dias?

TESTE 2: Lógica de Comparação
[ ] % Variação está positiva quando deve?
[ ] Cores mudam (verde/vermelho)?
[ ] Gráfico mostra 3 linhas?

TESTE 3: Performance
[ ] Página carrega em < 5 segundos?
[ ] Gráfico é responsivo?
[ ] Tabela scrolleia bem?

TESTE 4: Casos Extremos
[ ] E se houver 0 usuários em um período?
[ ] E se revenice for negativa?
[ ] E se dados estiverem atrasados?
```

### 5.2 Query de Validação Cruzada

```sql
-- Comparar Looker Studio vs BigQuery direto

-- VALIDAÇÃO: Receita de Hoje
SELECT
  'source' as check_type,
  SUM(revenue) as revenue_total
FROM `seu_projeto.analytics.fct_daily_metrics_comparative`
WHERE metric_date = CURRENT_DATE() - 1  -- Ontem
  AND period_label = 'd0'
  AND reference_date = CURRENT_DATE()

UNION ALL

SELECT
  'expected',
  SUM(CAST(event_value_in_usd AS FLOAT64))
FROM `seu_projeto.analytics_1.events_*`
WHERE _TABLE_SUFFIX = FORMAT_DATE('%Y%m%d', CURRENT_DATE() - 1)
  AND event_name = 'purchase';

-- Resultado: Os dois valores devem ser idênticos
```

---

## PARTE 6: AUTOMAÇÃO (CLOUD SCHEDULER)

### 6.1 Configurar Agendamento Automático

```bash
# Via gcloud CLI

gcloud scheduler jobs create bigquery comparative-metrics-daily \
  --schedule="0 1 * * *" \
  --time-zone="America/Sao_Paulo" \
  --location=us-central1 \
  --display-name="Comparative Metrics Daily" \
  --bigquery-query="""
    DECLARE processing_date DATE DEFAULT CURRENT_DATE() - 1;
    -- [COLE O SCRIPT 2 AQUI]
  """
```

### 6.2 Alternativa: Cloud Function (Python)

```python
# main.py - Google Cloud Function

import functions_framework
from google.cloud import bigquery
from datetime import datetime, timedelta
import logging

@functions_framework.cloud_event
def process_comparative_metrics(cloud_event):
    """
    Processa métricas comparativas diariamente
    Trigger: Cloud Pub/Sub ou Cloud Scheduler
    """
    
    client = bigquery.Client()
    
    try:
        # Executar query SQL
        query = """
        DECLARE processing_date DATE DEFAULT CURRENT_DATE() - 1;
        -- [COLE O SCRIPT 2 AQUI]
        """
        
        job = client.query(query, location="US")
        results = job.result()
        
        logging.info(f"✅ Processados {job.total_rows} registros")
        
        # Validação
        check_query = """
        SELECT COUNT(*) as total FROM `seu_projeto.analytics.fct_daily_metrics_comparative`
        WHERE metric_date = CURRENT_DATE() - 1
        """
        
        check_result = client.query(check_query).result()
        count = next(check_result)[0]
        
        if count > 0:
            logging.info(f"✅ Validação OK: {count} registros inseridos")
        else:
            logging.error("❌ Validação FALHOU: Nenhum registro!")
            
    except Exception as e:
        logging.error(f"❌ Erro ao processar: {str(e)}")
        raise

# deployment:
# gcloud functions deploy process_comparative_metrics \
#   --runtime python39 \
#   --trigger-topic comparative-metrics \
#   --entry-point process_comparative_metrics
```

### 6.3 Notificação de Erros

```yaml
# alerting-policy.yaml (Google Cloud Monitoring)

displayName: "Comparative Metrics - Daily Job Failed"
conditions:
  - displayName: "BigQuery Job Error"
    conditionThreshold:
      filter: |
        resource.type="bigquery_project"
        AND metric.type="bigquery.googleapis.com/job/num_failed_jobs"
      comparison: COMPARISON_GT
      thresholdValue: 0
      duration: "60s"

notificationChannels:
  - "projects/seu_projeto/notificationChannels/12345"  # Email ou Slack

documentation:
  content: "O job diário de métricas comparativas falhou. Verificar logs em BigQuery."
```

---

## PARTE 7: TROUBLESHOOTING

### 7.1 Problema: "Período aparece vazio"

```sql
-- Diagnóstico
SELECT DISTINCT period_label, COUNT(*) as rows
FROM `seu_projeto.analytics.fct_daily_metrics_comparative`
WHERE reference_date = CURRENT_DATE()
GROUP BY period_label;

-- Se algum período tiver 0 rows:
SELECT *
FROM `seu_projeto.analytics.fct_daily_metrics_comparative`
WHERE period_label IS NULL
LIMIT 100;

-- Solução: Verificar lógica de CASE WHEN no Script 2
```

### 7.2 Problema: "Números não batem com GA4"

```sql
-- Comparação direta
WITH looker_data AS (
  SELECT
    metric_date,
    SUM(revenue) as lds_revenue
  FROM `seu_projeto.analytics.fct_daily_metrics_comparative`
  WHERE period_label = 'd0'
  GROUP BY metric_date
),
ga4_data AS (
  SELECT
    DATE(TIMESTAMP_MILLIS(event_date * 1000)) as event_date,
    SUM(CAST(event_value_in_usd AS FLOAT64)) as ga4_revenue
  FROM `seu_projeto.analytics_1.events_*`
  WHERE _TABLE_SUFFIX >= '20240101'
  GROUP BY event_date
)
SELECT
  looker_data.metric_date,
  looker_data.lds_revenue,
  ga4_data.ga4_revenue,
  ROUND(
    ABS(looker_data.lds_revenue - ga4_data.ga4_revenue) / 
    NULLIF(ga4_data.ga4_revenue, 0) * 100, 2
  ) as variance_pct
FROM looker_data
FULL OUTER JOIN ga4_data 
  ON looker_data.metric_date = ga4_data.event_date
WHERE ABS(looker_data.lds_revenue - ga4_data.ga4_revenue) > 1  -- Mostrar apenas discrepâncias > R$1
ORDER BY metric_date DESC;
```

### 7.3 Problema: "Gráfico demora 30+ segundos"

```sql
-- Otimizar query: Usar tabela pré-agregada
-- ❌ Lento: Ler dados brutos
SELECT
  DATE(timestamp),
  SUM(value)
FROM raw_events
GROUP BY DATE(timestamp)

-- ✅ Rápido: Ler tabela agregada
SELECT
  metric_date,
  SUM(revenue)
FROM fct_daily_metrics_comparative  -- Já agregada!
GROUP BY metric_date

-- Verificar execution plan
EXPLAIN SELECT * FROM fct_daily_metrics_comparative LIMIT 1;
```

### 7.4 Problema: "Blend mostra valores duplicados"

```
Causa: Misma métrica sendo contada 2x no join

Solução:
1. Garantir que as chaves de join são exatas (não há espaços)
2. Usar INNER JOIN ao invés de FULL OUTER JOIN
3. Agregar dados antes do blend:
```

```sql
SELECT
  metric_date,
  period_label,
  SUM(revenue) as revenue,  -- Agregação explícita!
  SUM(users) as users
FROM fct_daily_metrics_comparative
GROUP BY metric_date, period_label
```

---

## PARTE 8: EXEMPLO COMPLETO (COPY/PASTE)

### 8.1 Implementação Mínima Viável

**PASSO 1:** Execute este SQL no BigQuery

```sql
-- Criar tabela (executar uma vez)
CREATE TABLE IF NOT EXISTS `seu_projeto.analytics.metrics_simple` (
  data DATE,
  periodo STRING,
  receita FLOAT64,
  usuarios INT64
);

-- Popular com dados (executar diariamente)
INSERT INTO `seu_projeto.analytics.metrics_simple`
WITH hoje AS (
  SELECT
    CURRENT_DATE() - 1 as data,
    'hoje' as periodo,
    1500.00 as receita,
    250 as usuarios
)
SELECT * FROM hoje
UNION ALL
SELECT CURRENT_DATE() - 8, 'd7', 1350.00, 230
UNION ALL
SELECT CURRENT_DATE() - 31, 'd30', 1100.00, 200;

-- Validar
SELECT * FROM `seu_projeto.analytics.metrics_simple`
ORDER BY data DESC;
```

**PASSO 2:** Looker Studio

1. Criar novo relatório
2. Conectar tabela `metrics_simple`
3. Adicionar Scorecard:
   - Métrica: SUM(receita)
   - Filtro: periodo = 'hoje'
4. Adicionar Gráfico Linha:
   - X: data
   - Y: receita
   - Cor: periodo

**Resultado:** Dashboard com 2 visualizações em 10 minutos! 🎉

---

## RESUMO DOS SCRIPTS

| Script | Propósito | Frequência |
|--------|-----------|-----------|
| Script 1 | Criar tabelas | Uma vez (setup) |
| Script 2 | Transformar dados | Diariamente (1 AM) |
| Script 3 | Validar dados | Após cada execução |
| Cloud Scheduler | Automação | Agendado |

---

**Pronto para implementar? Comece pelo Script 1!**
