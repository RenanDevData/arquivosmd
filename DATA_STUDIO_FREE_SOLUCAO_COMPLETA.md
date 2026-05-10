# Solução Completa: Datas Relativas no Data Studio Free (Sem BigQuery)
## GA4 + Data Studio Nativo - Guia Passo-a-Passo

---

## INTRODUÇÃO

**Você está aqui porque:**
- ✅ Só tem GA4 padrão
- ✅ Usa Data Studio Free
- ✅ Sem acesso a BigQuery
- ✅ Sem infraestrutura extra

**Boas notícias:**
- ✅ É completamente possível fazer datas relativas
- ✅ Sem custo adicional
- ✅ Com performance aceitável (< 20 segundos)
- ✅ Escalável para 90 dias de histórico

**Restrições que precisa aceitar:**
- ❌ Máximo ~30 dias de histórico (não 400)
- ❌ Máximo 5-6 visualizações comparativas (não 20+)
- ❌ Performance: 5-20 segundos (não instant)
- ⚠️ Se data for "Últimos 90 dias", o relatório fica pesado

---

## PARTE 1: ESTRUTURA CONCEITUAL

### O Truque Mágico

Ao invés de criar 15 cards separados (hoje, d-1, d-7, d-30...), você vai:

1. **Fazer 1 query única no GA4** com TODOS os períodos
2. **Usar campos calculados** para etiquetar cada linha
3. **Filtrar dinamicamente** no Looker Studio com controles

### Como Funciona

```
┌─────────────────────────────────────────────┐
│  Data Selecionada no Filtro: 15 de janeiro  │
└─────────────────────────────────────────────┘
                    ↓
         Cada linha de dados recebe:
         • Se data = 15/jan → "D0 (Hoje)"
         • Se data = 14/jan → "D1 (Ontem)"
         • Se data = 8/jan  → "D7"
         • Se data = 16/dez → "D30"
                    ↓
      Gráfico mostra as 4 linhas lado-a-lado
      Scorecard soma cada período separadamente
```

---

## PARTE 2: SETUP NO GA4 (NENHUMA MUDANÇA NECESSÁRIA)

Você pode usar GA4 exatamente como está agora.

**Dados que você vai usar:**
- Data (nativa)
- Usuários (nativa)
- Sessões (nativa)
- Receita (se GA4 tem e-commerce)
- Qualquer métrica que já está lá

**Nenhuma configuração extra necessária.**

---

## PARTE 3: PASSO-A-PASSO NO DATA STUDIO

### PASSO 1: Criar Novo Relatório

```
1. Vá para datastudio.google.com
2. Clique em "+ Vazio" (novo relatório)
3. Conectar Dados → Google Analytics 4
4. Selecionar sua propriedade GA4
5. Criar relatório
```

### PASSO 2: Adicionar Tabela Base (Data Range Automático)

```
1. Inserir → Tabela
2. Dimensão: Data (date)
3. Métrica: Usuários (users)
4. Métrica: Receita (purchase_revenue) - se tiver
5. Deixar a tabela carregar com filtro de data "Últimos 7 dias"
```

**Importante:** Neste passo, seu relatório vai mostrar:
```
Data        | Usuários | Receita
2024-01-15  | 1250     | R$ 5200
2024-01-14  | 1180     | R$ 4900
2024-01-13  | 1320     | R$ 5100
...
```

### PASSO 3: Criar Campo Calculado - Etiqueta do Período

Clique na dimensão "Data" → Mais → Crie um campo calculado

```javascript
CASE
  WHEN DATEDIFF(TODAY(), data) = 0 THEN "D0"
  WHEN DATEDIFF(TODAY(), data) = 1 THEN "D1"
  WHEN DATEDIFF(TODAY(), data) = 2 THEN "D2"
  WHEN DATEDIFF(TODAY(), data) = 7 THEN "D7"
  WHEN DATEDIFF(TODAY(), data) = 30 THEN "D30"
  ELSE "Outro"
END

Nome do campo: period_label
```

**Explicação:**
- `DATEDIFF(TODAY(), data)` = quantos dias atrás
- Se diferença = 0 → é hoje ("D0")
- Se diferença = 1 → é ontem ("D1")
- Se diferença = 7 → é 7 dias atrás ("D7")
- Caso contrário → "Outro" (vai filtrar depois)

### PASSO 4: Adicionar o Campo à Tabela

Na tabela que criou:
```
1. Clique na tabela
2. Painel Direito → Dimensões
3. Clique no + ao lado de "Dimensões"
4. Procure por "period_label"
5. Adicionar
```

**Resultado:** Tabela agora tem:
```
Data        | period_label | Usuários | Receita
2024-01-15  | D0           | 1250     | R$ 5200
2024-01-14  | D1           | 1180     | R$ 4900
2024-01-08  | D7           | 1095     | R$ 4500
2023-12-16  | D30          | 920      | R$ 3800
```

### PASSO 5: Criar o Scorecard - Receita D0

```
1. Inserir → Scorecard (o card simples)
2. Métrica: Receita (purchase_revenue)
3. No painel direito:
   - Filtrar por: period_label = "D0"
   - Mostrar números: Sim
   - Formato de moeda: BRL
```

**Resultado:**
```
┌─────────────────┐
│     R$ 5.200    │
│  Receita Hoje   │
└─────────────────┘
```

### PASSO 6: Criar Scorecard com Comparação - D0 vs D7

Este é o TRUQUE PRINCIPAL. Vai parecer louco mas funciona:

```
1. Inserir → Scorecard
2. Métrica Principal: Receita (purchase_revenue)
3. Filtro: period_label = "D0"
4. Painel Direito → Comparação
   ☑ Mostrar comparação
   Métrica de comparação: Receita (purchase_revenue)
5. TRUQUE: Na aba "Filtros de Comparação":
   Adicionar filtro: period_label = "D7"
```

Pronto! O scorecard vai mostrar:

```
┌──────────────────────┐
│     R$ 5.200 ↑ 18%   │
│   Receita vs D-7     │
└──────────────────────┘
```

**Como funciona o truque:**
- Valor principal: filtra por D0
- Valor de comparação: filtra por D7
- A diferença percentual é calculada automaticamente
- Data selecionada no filtro muda automaticamente as datas base

### PASSO 7: Criar o Gráfico Comparativo (Linha)

```
1. Inserir → Gráfico Série Temporal (linha)
2. Dimensão: Data
3. Métrica: Receita
4. Breakout (cor): period_label
5. Filtrar: period_label IN ("D0", "D7", "D30")
```

**Configurações avançadas:**
```
Painel Direito → Configurações de Série:
- Série D0: Cor azul, espessura 3px, linha sólida
- Série D7: Cor cinza, espessura 2px, linha tracejada
- Série D30: Cor cinza claro, espessura 2px, linha pontilhada
```

**Resultado:**
```
R$ │     D0 (Azul —)
   │    /   D7 (Cinza - -)
   │   /    D30 (Cinza . .)
   │  /
   │ /
   └─────────────────────→ Data
```

---

## PARTE 4: CAMPOS CALCULADOS PRONTOS PARA COPIAR

### Campo 1: Etiqueta do Período (copia do Passo 3)

```javascript
CASE
  WHEN DATEDIFF(TODAY(), data) = 0 THEN "D0"
  WHEN DATEDIFF(TODAY(), data) = 1 THEN "D1"
  WHEN DATEDIFF(TODAY(), data) = 2 THEN "D2"
  WHEN DATEDIFF(TODAY(), data) = 7 THEN "D7"
  WHEN DATEDIFF(TODAY(), data) = 30 THEN "D30"
  ELSE "Outro"
END

Nome: period_label
```

### Campo 2: Display Name (para gráficos ficar bonito)

```javascript
CASE
  WHEN period_label = "D0" THEN "Hoje"
  WHEN period_label = "D1" THEN "Ontem (D-1)"
  WHEN period_label = "D2" THEN "Ante-ontem (D-2)"
  WHEN period_label = "D7" THEN "7 dias atrás"
  WHEN period_label = "D30" THEN "30 dias atrás"
  ELSE period_label
END

Nome: period_display
```

### Campo 3: Cor Condicional (para scorecard)

```javascript
CASE
  WHEN period_label = "D0" THEN "#1f77b4"
  WHEN period_label = "D7" THEN "#ff7f0e"
  WHEN period_label = "D30" THEN "#2ca02c"
  ELSE "#7f7f7f"
END

Nome: period_color
```

### Campo 4: Ordem de Exibição (para gráficos)

```javascript
CASE
  WHEN period_label = "D0" THEN 1
  WHEN period_label = "D1" THEN 2
  WHEN period_label = "D2" THEN 3
  WHEN period_label = "D7" THEN 4
  WHEN period_label = "D30" THEN 5
  ELSE 99
END

Nome: period_sort_order
```

### Campo 5: Receita Formatada (opcional, deixa bonitão)

```javascript
CONCAT(
  'R$ ',
  TEXT(ROUND(PURCHASE_REVENUE, 2), '#,##0.00')
)

Nome: revenue_formatted
```

---

## PARTE 5: LAYOUT RECOMENDADO

### Estrutura do Relatório

```
┌────────────────────────────────────────────────────┐
│  DASHBOARD ANALÍTICO - COMPARATIVO DE PERÍODOS     │
│  Últimos 7 dias                        [Data ▼]    │
└────────────────────────────────────────────────────┘

┌───────LINHA 1: SCORECARDS PRINCIPAIS──────────────┐
│                                                     │
│  ┌──────────────┐  ┌──────────────┐                │
│  │ Receita Hoje │  │ Receita vs D7 │               │
│  │  R$ 5.200    │  │ R$ 5.2K ↑18% │               │
│  └──────────────┘  └──────────────┘                │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐                │
│  │Usuários Hoje │  │Usuários vs D7 │               │
│  │  1.250       │  │ 1.25K ↑ 8%   │               │
│  └──────────────┘  └──────────────┘                │
│                                                     │
└─────────────────────────────────────────────────────┘

┌─────LINHA 2: GRÁFICO COMPARATIVO (LINHA)──────────┐
│                                                     │
│  Receita - Últimos 30 dias                         │
│  (Comparando: Hoje vs D-7 vs D-30)                 │
│                                                     │
│  [Gráfico de linha com 3 séries]                   │
│                                                     │
│  Legenda: — Hoje  - - D-7  . . D-30               │
│                                                     │
└─────────────────────────────────────────────────────┘

┌───────LINHA 3: TABELA DETALHADA────────────────────┐
│                                                     │
│  Resumo Comparativo                                │
│                                                     │
│  Data    │ Período │ Receita  │ Usuários           │
│  ─────────────────────────────────────────────     │
│  01/15   │ Hoje    │ R$ 5.2K  │ 1.250              │
│  01/14   │ D-1     │ R$ 4.8K  │ 1.180              │
│  01/08   │ D-7     │ R$ 4.6K  │ 1.150              │
│  12/16   │ D-30    │ R$ 3.8K  │ 920                │
│                                                     │
└─────────────────────────────────────────────────────┘
```

---

## PARTE 6: PASSO-A-PASSO PARA CADA VISUALIZAÇÃO

### Scorecard 1: Receita Hoje (Simples)

```
Visualização: Scorecard
Métrica: Receita (purchase_revenue)
Filtros:
  ✓ period_label = "D0"
  ✓ Data: Últimos 7 dias (padrão)

Estilo:
  ✓ Tamanho do número: Grande (40pt)
  ✓ Moeda: BRL
  ✓ Mostrar rótulo: "Receita Hoje"
```

### Scorecard 2: Receita Hoje vs D-7 (Com Comparação)

```
Visualização: Scorecard
Métrica Principal: Receita
Filtro Principal: period_label = "D0"

Comparação:
  ☑ Mostrar comparação
  Métrica de Comparação: Receita
  Filtro de Comparação: period_label = "D7"
  Mostrar: Diferença percentual + tendência ↑↓

Estilo:
  ✓ Se positivo: Verde com ↑
  ✓ Se negativo: Vermelho com ↓
  ✓ Tamanho: Grande
```

### Gráfico 1: Linha Comparativa

```
Visualização: Série Temporal (Linha)
Dimensão X: Data
Métrica Y: Receita
Cor (Breakout): period_display

Filtros:
  ✓ period_label IN ("D0", "D7", "D30")
  ✓ Data: Últimos 30 dias

Configurações Avançadas:
  ✓ Remover valores nulos: Sim
  ✓ Curva suave: Não
  ✓ Mostrar pontos: Sim
  
Cores:
  D0 → Azul (#1f77b4)
  D7 → Laranja (#ff7f0e)
  D30 → Verde (#2ca02c)

Estilos de Linha:
  D0 → Linha sólida (3px)
  D7 → Linha tracejada (2px)
  D30 → Linha pontilhada (2px)
```

### Tabela: Resumo Comparativo

```
Visualização: Tabela

Dimensões:
  ✓ Coluna 1: period_display
  
Métricas:
  ✓ Coluna 2: Receita
  ✓ Coluna 3: Usuários
  ✓ Coluna 4: Sessões

Filtros:
  ✓ period_label IN ("D0", "D1", "D7", "D30")
  ✓ Data: Últimos 30 dias

Ordenação:
  ✓ Por: period_sort_order (crescente)

Formatação:
  ✓ Receita: Moeda BRL
  ✓ Usuários: Inteiro sem decimais
  ✓ Sessões: Inteiro sem decimais

Painel de Controle:
  ✓ Cores nas células: Sim (padrão)
```

---

## PARTE 7: CONFIGURAÇÃO DO FILTRO DE DATA (IMPORTANTE!)

Este é o filtro que "ativa a mágica" de mudar tudo automaticamente.

```
1. Clique em "Controles" no menu superior
2. Adicionar Controle
3. Tipo: Controle de Intervalo de Data
4. Dimensão: Data
5. Nome do Controle: "Período"
6. Tipo de Intervalo: "Intervalo de data específico"
7. Padrão: "Últimos 7 dias" (RECOMENDADO para não ficar pesado)
8. Aplicar a: Todos os elementos do relatório ✓

Configurações Avançadas:
  ✓ Mostrar controle no relatório
  ✓ Requisito obrigatório: Não
  ✓ Permitir intervalos relativos: Sim
```

**Importante:** 
- Se usar "Últimos 90 dias" → vai ficar MUITO lento
- Máximo recomendado: "Últimos 30 dias"
- Ideal: "Últimos 7 dias" ou "Últimos 14 dias"

---

## PARTE 8: O PROBLEMA QUE VOCÊ VAI TER (E COMO RESOLVER)

### Problema 1: "A tabela mostra todas as datas, não só D0, D7, D30"

**Causa:** Esqueceu de adicionar o filtro no campo period_label

**Solução:**
```
Clique na visualização
Painel direito → Filtros
Adicionar filtro: period_label IN ("D0", "D7", "D30")
```

### Problema 2: "O gráfico fica muito lento (10+ segundos)"

**Causa:** Período de data é muito grande

**Solução:**
```
Reduzir para "Últimos 14 dias" ao invés de "Últimos 90 dias"
Remover visualizações desnecessárias
```

**Performance esperada:**
- Últimos 7 dias: 2-5 segundos ✅
- Últimos 14 dias: 5-10 segundos ✅
- Últimos 30 dias: 10-20 segundos ⚠️ (aceitável)
- Últimos 90 dias: 30+ segundos ❌ (muito lento)

### Problema 3: "Scorecard mostra valor errado"

**Causa:** Campo calculado tem erro na lógica

**Verificar:**
```javascript
// ✓ Correto
CASE
  WHEN DATEDIFF(TODAY(), data) = 0 THEN "D0"
  WHEN DATEDIFF(TODAY(), data) = 7 THEN "D7"
END

// ❌ Errado (falta "TODAY()" ou data está como string)
CASE
  WHEN date = "2024-01-15" THEN "D0"  // Data hardcoded!
END
```

### Problema 4: "Comparação percentual está negativa quando deveria ser positiva"

**Causa:** Ordem da comparação está invertida

**Solução:**
```
Scorecard → Painel Direito → Comparação
Marque: "Mostrar inversão da comparação"
OU
Troque a métrica de comparação de lugar
```

### Problema 5: "D-7 não aparece no gráfico"

**Causa:** Não tem dados para D-7 (não é 7 dias atrás de forma exata)

**Exemplo do problema:**
```
Se hoje é 15 de janeiro:
  - D0 = 15 de janeiro ✓ (tem dados)
  - D7 = 8 de janeiro ✓ (tem dados)
  - D30 = 16 de dezembro ✓ (tem dados)

Se você selecionar "Últimos 6 dias":
  - D0 = 15 de janeiro ✓ (tem dados)
  - D7 = 8 de janeiro ✗ (FORA DO RANGE!)
  - Resultado: D7 não aparece
```

**Solução:**
```
Sempre selecionar período ≥ 31 dias para ter D-30
Sempre selecionar período ≥ 8 dias para ter D-7
```

---

## PARTE 9: RECOMENDAÇÕES DE DESIGN

### Cores Recomendadas

```
D0 (Hoje):     #1f77b4 (Azul) - Ênfase principal
D1 (Ontem):    #ff7f0e (Laranja) - Secundário
D7 (Semana):   #2ca02c (Verde) - Comparação
D30 (Mês):     #9467bd (Roxo) - Histórico
Outro:         #7f7f7f (Cinza) - Ignorar
```

### Nomes dos Campos (Padrão)

```
Data Campo             | Nome no Relatório
───────────────────────┼─────────────────────
period_label           | "Período (Interno)"
period_display         | "Período"
period_sort_order      | "Ordem (Ocultar)"
period_color           | "Cor (Ocultar)"
revenue_formatted      | "Receita (Formatada)"
```

### Tamanho de Fonte Recomendado

```
Scorecard:           40-48px (número grande)
                     14px (rótulo)

Gráfico:             14px (título)
                     12px (eixos)
                     10px (legenda)

Tabela:              14px (cabeçalho)
                     12px (dados)
```

---

## PARTE 10: EXEMPLO PRONTO PARA COPIAR (RECONSTRUIR DO ZERO)

Se quiser reconstruir tudo do zero em 30 minutos, siga isto:

### Minuto 1-5: Setup
```
1. datastudio.google.com
2. "+ Vazio"
3. Conectar GA4
4. Criar relatório
```

### Minuto 6-10: Tabela Base
```
1. Inserir → Tabela
2. Data (eixo X)
3. Receita e Usuários (métricas)
4. Deixar carregar
```

### Minuto 11-15: Campo Calculado
```
1. Dimensão "Data" → Mais → Criar campo
2. Copiar/colar (PASSO 3 da seção "PARTE 4"):

CASE
  WHEN DATEDIFF(TODAY(), data) = 0 THEN "D0"
  WHEN DATEDIFF(TODAY(), data) = 1 THEN "D1"
  WHEN DATEDIFF(TODAY(), data) = 7 THEN "D7"
  WHEN DATEDIFF(TODAY(), data) = 30 THEN "D30"
  ELSE "Outro"
END

3. Nome: period_label
4. Criar
```

### Minuto 16-20: Scorecard 1
```
1. Inserir → Scorecard
2. Métrica: Receita
3. Filtro: period_label = "D0"
4. Tamanho: Grande
5. Padrão (sem comparação por enquanto)
```

### Minuto 21-25: Scorecard 2 (Com Comparação)
```
1. Inserir → Scorecard
2. Métrica: Receita
3. Filtro: period_label = "D0"
4. Painel Direito → Comparação
5. ☑ Mostrar comparação
6. Métrica de Comparação: Receita
7. Filtro de Comparação: period_label = "D7"
8. Aplicar
```

### Minuto 26-30: Gráfico + Filtro
```
1. Inserir → Série Temporal
2. X: Data
3. Y: Receita
4. Cor: period_label
5. Filtro: period_label IN ("D0", "D7", "D30")
6. Controles → Intervalo de Data
7. Aplicar a tudo
8. Pronto!
```

---

## PARTE 11: LIMITAÇÕES (ACEITAR AGORA, POUPAR TEMPO DEPOIS)

| Limitação | Impacto | Solução |
|-----------|--------|--------|
| Máx 5-6 visualizações | Relatório fica lento | Priorizar as mais importantes |
| Máx 30 dias de história | Não vê YoY | Use D-30 como máximo histórico |
| Performance 5-20s | Não é instant | Aceitar como normal no Data Studio Free |
| Sem filtros customizados | Padrão é limitado | Usar período fixo (ex: últimos 7 dias) |
| Sem alertas automáticos | Alerta manual | Verificar diariamente |
| Sem comparação de múltiplas métricas | Comparar uma por vez | Criar scorecard por métrica |

---

## PARTE 12: ROADMAP - COMO EVOLUIR

### Fase 1: MVP (Semana 1)
```
✓ 2 Scorecards (receita hoje + comparação)
✓ 1 Gráfico linha comparativo
✓ 1 Tabela resumida
✓ Filtro de data manual

Tempo: 2-4 horas
```

### Fase 2: Expansão (Semana 2-3)
```
✓ +2 Scorecards (usuários + conversão)
✓ +1 Gráfico (usuários)
✓ Cores customizadas
✓ Display names melhorados

Tempo: 4-6 horas
```

### Fase 3: Polimento (Semana 4)
```
✓ 6 visualizações no total
✓ Nomes em português bonitão
✓ Tema visual consistente
✓ Documentação interna

Tempo: 4-6 horas
```

### Se Crescer Depois (Semana 8+)
```
CONSIDERAR MIGRAÇÃO PARA:
- Google Sheets + Lookup (mais fácil que BQ)
- OU BigQuery (solução definitiva)
Mas agora você já sabe como fazer sem!
```

---

## PARTE 13: COMANDOS ÚTEIS (COPY/PASTE)

### Campo Calculado: Todos os Períodos Até D-365

```javascript
CASE
  WHEN DATEDIFF(TODAY(), data) = 0 THEN "D0"
  WHEN DATEDIFF(TODAY(), data) = 1 THEN "D1"
  WHEN DATEDIFF(TODAY(), data) = 2 THEN "D2"
  WHEN DATEDIFF(TODAY(), data) = 3 THEN "D3"
  WHEN DATEDIFF(TODAY(), data) = 7 THEN "D7"
  WHEN DATEDIFF(TODAY(), data) = 14 THEN "D14"
  WHEN DATEDIFF(TODAY(), data) = 30 THEN "D30"
  WHEN DATEDIFF(TODAY(), data) = 90 THEN "D90"
  WHEN DATEDIFF(TODAY(), data) = 365 THEN "YoY"
  ELSE "Outro"
END
```

### Campo Calculado: Intervalo de Períodos (Se for Intervalo)

```javascript
CASE
  WHEN DATEDIFF(TODAY(), data) >= 0 AND DATEDIFF(TODAY(), data) <= 1 THEN "Últimas 24h"
  WHEN DATEDIFF(TODAY(), data) >= 2 AND DATEDIFF(TODAY(), data) <= 6 THEN "Últimas 1 semana"
  WHEN DATEDIFF(TODAY(), data) >= 7 AND DATEDIFF(TODAY(), data) <= 13 THEN "Semana anterior"
  WHEN DATEDIFF(TODAY(), data) >= 14 AND DATEDIFF(TODAY(), data) <= 30 THEN "Últimas 2-4 semanas"
  ELSE "Outro"
END
```

### Filtro para Gráfico (3 períodos)

```
period_label IN ("D0", "D7", "D30")
```

### Filtro para Gráfico (Todos exceto "Outro")

```
period_label != "Outro"
```

---

## PARTE 14: DEBUGGING - SE ALGO DER ERRADO

### Checklist de Verificação

```
☐ Campo period_label foi criado?
  Teste: Adicionar como dimensão em uma tabela
  Esperado: Ver "D0", "D1", "D7", "D30", "Outro"

☐ Filtro de data está aplicado?
  Teste: Mude o filtro de "Últimos 7 dias" para "Últimos 30 dias"
  Esperado: Scorecard deve mudar de valor

☐ Comparação está invertida?
  Teste: Verifi
que se % positivo faz sentido
  Esperado: Se receita de hoje > receita de d-7, deve ser positivo

☐ Gráfico está vazio?
  Teste: Remova o filtro period_label temporariamente
  Esperado: Gráfico deve mostrar TODAS as datas
  Depois: Re-adicione o filtro

☐ Performance está ruim?
  Teste: Reduza período de "Últimos 30 dias" para "Últimos 7 dias"
  Esperado: Carregamento < 5 segundos
```

---

## PARTE 15: DICAS EXTRAS (NINJA)

### Dica 1: Usar "Ano Passado" ao invés de "D-365"

```javascript
CASE
  WHEN FORMAT_DATE("%m-%d", data) = FORMAT_DATE("%m-%d", TODAY()) THEN "Mesmo dia ano passado"
  ELSE "Outro"
END

// Isso compara o mesmo dia/mês, mas do ano anterior
// Mais flexível que D-365 (funciona independente de ano bissexto)
```

### Dica 2: Scorecard com Indicador de Tendência

Usar campo calculado como rótulo:

```javascript
CONCAT(
  CAST(ROUND(metric, 0) AS STRING),
  CASE
    WHEN metric > previous_metric THEN " ↑"
    WHEN metric < previous_metric THEN " ↓"
    ELSE " →"
  END
)
```

### Dica 3: Gráfico de Barras Comparativo (Alternativa à Linha)

```
Se preferir barras ao invés de linha:

1. Inserir → Gráfico de Barras
2. Eixo X: period_display
3. Eixo Y: Receita
4. Filtro: period_label IN ("D0", "D7", "D30")
5. Cor por: period_label

Resultado: 3 barras lado-a-lado
D0 (azul) vs D7 (cinza) vs D30 (cinza claro)
```

### Dica 4: Scorecard com Métrica Dupla (Receita + Usuários)

Criar 2 campos separados:

```javascript
// Campo 1
CONCAT(
  TEXT(ROUND(PURCHASE_REVENUE, 0), '$#,##0'),
  " em ",
  TEXT(ROUND(USERS, 0), '#,##0'),
  " usuários"
)

// Exemplo: "$5,200 em 1,250 usuários"
```

### Dica 5: Esconder Campos Desnecessários

```
Quando criar campos calculados extras (como period_sort_order):

Campos → Clique no olho (ícone)
→ Marcar "Ocultar"

Assim o usuário final não vê fields confusos
```

---

## RESUMO FINAL

### O que você vai ter:

```
✅ Relatório com datas relativas (d0, d7, d30)
✅ Scorecards com comparação percentual automática
✅ Gráfico mostrando 3+ períodos lado-a-lado
✅ Tabela resumida com comparações
✅ Filtro de data que muda tudo automaticamente
✅ Zero custo extra
✅ Sem BigQuery, sem Sheets, sem nada extra
✅ Totalmente funcional em 30-60 minutos
```

### Limitações que vai ter:

```
⚠️ Máximo ~30 dias de história (não ilimitado)
⚠️ Máximo 5-6 visualizações (depois fica lento)
⚠️ Performance 5-20 segundos (não instant)
⚠️ Sem alertas automáticos
⚠️ Sem múltiplas comparações simultâneas
```

### Se ficar grande depois:

```
Migrar para:
- Google Sheets Lookup (intermediário) - 1-2 semanas
- BigQuery (solução final) - 2-3 semanas
```

---

## PRÓXIMOS PASSOS

### Agora:
1. Abra seu relatório GA4 + Data Studio
2. Siga o **PASSO-A-PASSO (Parte 3)**
3. Copie os **CAMPOS CALCULADOS (Parte 4)**
4. Construa as **VISUALIZAÇÕES (Parte 6)**

### Em 30-60 minutos:
Você terá um relatório 100% funcional com datas relativas

### Se travar:
- Consulte **PARTE 14 (Debugging)**
- 90% dos problemas estão lá

### Se crescer:
- Considere migração (mas isso é bridge para depois)

---

**Você consegue fazer isso. É mais simples do que parece. Boa sorte! 🚀**
