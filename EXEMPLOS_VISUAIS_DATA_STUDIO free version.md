# Exemplos Visuais e Casos de Uso - Data Studio Free

## EXEMPLO 1: Estrutura Básica (5 minutos)

### Antes (SEM campo calculado):

```
Data        | Receita | Usuários
────────────┼─────────┼──────────
2024-01-15  | 5200    | 1250
2024-01-14  | 4800    | 1180
2024-01-13  | 5100    | 1320
2024-01-12  | 4900    | 1200
2024-01-11  | 5300    | 1350
2024-01-10  | 4700    | 1100
2024-01-09  | 4600    | 1050
2024-01-08  | 4900    | 1150
...
```

**Problema:** Como você sabe qual é qual período?

---

### Depois (COM campo calculado):

```
Data        | period_label | Receita | Usuários
────────────┼──────────────┼─────────┼──────────
2024-01-15  | D0           | 5200    | 1250     ← Hoje
2024-01-14  | D1           | 4800    | 1180     ← Ontem
2024-01-13  | D2           | 5100    | 1320
2024-01-12  | D3           | 4900    | 1200
2024-01-11  | D4           | 5300    | 1350
2024-01-10  | D5           | 4700    | 1100
2024-01-09  | D6           | 4600    | 1050
2024-01-08  | D7           | 4900    | 1150     ← 7 dias atrás
...
2023-12-16  | D30          | 3800    | 920      ← 30 dias atrás
```

**Benefício:** Cada linha é etiquetada automaticamente

---

## EXEMPLO 2: Campo Calculado - Como Funciona

### A Fórmula:

```javascript
CASE
  WHEN DATEDIFF(TODAY(), data) = 0 THEN "D0"
  WHEN DATEDIFF(TODAY(), data) = 1 THEN "D1"
  WHEN DATEDIFF(TODAY(), data) = 7 THEN "D7"
  WHEN DATEDIFF(TODAY(), data) = 30 THEN "D30"
  ELSE "Outro"
END
```

### Simulação do que acontece:

```
Se TODAY() = 15 de janeiro de 2024:

Linha 1: data = 15 de janeiro
  → DATEDIFF(2024-01-15, 2024-01-15) = 0
  → Retorna "D0" ✓

Linha 2: data = 14 de janeiro
  → DATEDIFF(2024-01-15, 2024-01-14) = 1
  → Retorna "D1" ✓

Linha 8: data = 8 de janeiro
  → DATEDIFF(2024-01-15, 2024-01-08) = 7
  → Retorna "D7" ✓

Linha 36: data = 16 de dezembro
  → DATEDIFF(2024-01-15, 2023-12-16) = 30
  → Retorna "D30" ✓

Linha 37: data = 15 de dezembro
  → DATEDIFF(2024-01-15, 2023-12-15) = 31
  → Não bate nenhum CASE
  → Retorna "Outro" ✓
```

### O Truque Mágico:

Quando você muda o filtro de data:

```
Cenário 1: Se mudar para "16 de janeiro"

A TODAY() muda para "16 de janeiro"
A fórmula recalcula automaticamente:
  Linha 1 (data = 15 de janeiro) → DATEDIFF(jan-16, jan-15) = 1 → "D1" (não é mais D0!)
  Linha 2 (data = 16 de janeiro) → DATEDIFF(jan-16, jan-16) = 0 → "D0" (é hoje!)

Resultado: Todos os valores se reajustam automaticamente!
```

---

## EXEMPLO 3: Scorecard Simples vs Scorecard com Comparação

### Scorecard 1: Receita Hoje (Simples)

```
┌─────────────────────────┐
│      R$ 5.200,00        │
│    Receita - Hoje       │
└─────────────────────────┘

Configuração:
├─ Métrica: PURCHASE_REVENUE
├─ Filtro: period_label = "D0"
└─ Padrão (sem comparação)
```

**Problema:** Você não sabe se isso é bom ou ruim

---

### Scorecard 2: Receita Hoje vs D-7 (Com Comparação)

```
┌──────────────────────────┐
│  R$ 5.200,00           │
│  ↑ 13%                  │
│  vs D-7                 │
└──────────────────────────┘

Configuração:
├─ Métrica Principal: PURCHASE_REVENUE
├─ Filtro Principal: period_label = "D0"
├─ Mostrar Comparação: SIM
├─ Métrica de Comparação: PURCHASE_REVENUE
├─ Filtro de Comparação: period_label = "D7"
└─ Mostrar Tendência: ↑↓
```

**Benefício:** Você vê que hoje está 13% acima de 7 dias atrás

---

### Scorecard 3: Multi-Período (Avançado)

Se quiser comparação com múltiplos períodos, você criar 3 scorecards:

```
┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐
│    R$ 5.200,00      │  │    R$ 5.200,00      │  │    R$ 5.200,00      │
│    ↑ 13% vs D-7     │  │    ↑ 37% vs D-30    │  │    ↑ 147% vs YoY    │
│  (Últimos 7 dias)   │  │ (Últimos 30 dias)   │  │  (Ano passado)      │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘

Card 1:
├─ Comparação com: D7 (7 dias atrás)

Card 2:
├─ Comparação com: D30 (30 dias atrás)

Card 3:
├─ Comparação com: YoY (ano passado, se quiser adicionar)
```

---

## EXEMPLO 4: Gráfico de Linha Comparativo

### Visualização Final:

```
R$ 5.500 ┤     D0 (Hoje - Azul)
         │    /│
R$ 5.300 ┤   / │  D7 (Cinza tracejado)
         │  /  │
R$ 5.100 ┤ /   │ D30 (Cinza pontilhado)
         │/    ▼
R$ 4.900 ├─────────────────────────────
         │
R$ 4.700 ┤
         │
         └─────────────────────────────→ Data
           01/09  01/10  01/11  01/12  01/13  01/14  01/15
```

### Código de Configuração:

```
Tipo: Série Temporal (Linha)
Eixo X: Data
Eixo Y: PURCHASE_REVENUE
Cor: period_display

Filtro: period_label IN ("D0", "D7", "D30")
Data: Últimos 30 dias

Cores Personalizadas:
├─ D0 (Hoje): #1f77b4 (Azul) - Linha sólida 3px
├─ D7: #ff7f0e (Laranja) - Linha tracejada 2px
└─ D30: #2ca02c (Verde) - Linha pontilhada 2px
```

### Interpretação:

```
Se você vê:
- D0 acima de D7 → Hoje está melhor que 7 dias atrás
- D7 acima de D30 → Última semana melhorou vs últimos 30 dias
- Todas subindo → Tendência positiva
- Todas caindo → Tendência negativa
```

---

## EXEMPLO 5: Tabela Resumida

### Layout Ideal:

```
╔════════════════════════════════════════════════════════╗
║           RESUMO COMPARATIVO DE PERÍODOS               ║
╠════════════╦══════════╦══════════╦══════════════════════╣
║  Período   ║ Receita  ║ Usuários ║  Sessões Médias     ║
╠════════════╬══════════╬══════════╬══════════════════════╣
║   Hoje     │ R$ 5.2K  │  1.250   │      2.1K            ║  ← D0
║  Ontem     │ R$ 4.8K  │  1.180   │      2.0K            ║  ← D1
║ 7 dias atr.│ R$ 4.6K  │  1.150   │      1.9K            ║  ← D7
║ 30 dias at.│ R$ 3.8K  │    920   │      1.7K            ║  ← D30
╚════════════╩══════════╩══════════╩══════════════════════╝
```

### Configuração da Tabela:

```
Dimensões:
├─ Coluna 1: period_display (Período)

Métricas:
├─ Coluna 2: PURCHASE_REVENUE (Receita)
├─ Coluna 3: USERS (Usuários)
└─ Coluna 4: SESSION_ID (Sessões - Count Distinct)

Filtros:
├─ period_label IN ("D0", "D1", "D7", "D30")
└─ Data: Últimos 30 dias

Ordenação: period_sort_order (crescente)
```

---

## EXEMPLO 6: Painel Completo (Dashboard Inteiro)

### Layout Recomendado:

```
┌─────────────────────────────────────────────────────────────┐
│  Dashboard de Análise - Períodos Comparativos                │
│  Período: Últimos 7 dias [Alterar ▼]                        │
└─────────────────────────────────────────────────────────────┘

┌─ ROW 1: SCORECARDS PRINCIPAIS (4 cards) ─────────────────────┐
│                                                                 │
│  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐      │
│  │   Receita     │  │ Receita vs D7 │  │   Usuários    │      │
│  │  R$ 5.200    │  │ R$ 5.2K ↑13%  │  │    1.250      │      │
│  │   (Hoje)      │  │   (Comp.)     │  │   (Hoje)      │      │
│  └───────────────┘  └───────────────┘  └───────────────┘      │
│                                                                 │
│  ┌───────────────┐                                            │
│  │ Usu. vs D-7   │                                            │
│  │ 1.25K ↑ 8%   │                                            │
│  │   (Comp.)     │                                            │
│  └───────────────┘                                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─ ROW 2: GRÁFICO COMPARATIVO ──────────────────────────────────┐
│                                                                 │
│  Receita - Últimos 30 Dias (3 Períodos)                       │
│                                                                 │
│  R$ │                                                          │
│      │  ─ D0 (Hoje, Azul)                                      │
│      │  ─ ─ D7 (Cinza tracejado)                              │
│      │  ─ ─ ─ D30 (Cinza pontilhado)                          │
│      │                                                         │
│ 5.2K │   ╱╲                                                    │
│      │  ╱  ╲   ╱╲                                             │
│ 5.0K │ ╱    ╲ ╱  ╲                                            │
│      │╱      ╲    ╲                                           │
│ 4.8K │        ╲╲   ╲                                          │
│      │         ╲ ╲   ╲                                        │
│ 4.6K │          ╲ ╲   ╲                                       │
│      │           ╲ ╲   ╲                                      │
│      └──────────────────────────────────────────→             │
│      01/08 01/09 01/10 01/11 01/12 01/13 01/14 01/15         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

┌─ ROW 3: TABELA DETALHADA ─────────────────────────────────────┐
│                                                                 │
│  Resumo por Período                                            │
│                                                                 │
│  Período      Receita      Usuários      Sessões              │
│  ──────────────────────────────────────────────────────────   │
│  Hoje         R$ 5.200     1.250         2.100                │
│  Ontem        R$ 4.800     1.180         2.000                │
│  7 dias       R$ 4.600     1.150         1.900                │
│  30 dias      R$ 3.800       920         1.700                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## EXEMPLO 7: Variações de Período

### Se quiser D0, D1, D2 (últimos 3 dias):

```javascript
CASE
  WHEN DATEDIFF(TODAY(), data) = 0 THEN "D0"
  WHEN DATEDIFF(TODAY(), data) = 1 THEN "D1"
  WHEN DATEDIFF(TODAY(), data) = 2 THEN "D2"
  ELSE "Outro"
END
```

**Gráfico resultante:**
```
R$ 5.3K │  D0  D1  D2
        │  ▮   ▮   ▮
R$ 5.0K │  ▮   ▮   ▮
        │  ▮   ▮   ▮
```

---

### Se quiser por Semana (últimas 4 semanas):

```javascript
CASE
  WHEN DATEDIFF(TODAY(), data) >= 0 AND DATEDIFF(TODAY(), data) <= 6 THEN "Semana Atual"
  WHEN DATEDIFF(TODAY(), data) >= 7 AND DATEDIFF(TODAY(), data) <= 13 THEN "Semana -1"
  WHEN DATEDIFF(TODAY(), data) >= 14 AND DATEDIFF(TODAY(), data) <= 20 THEN "Semana -2"
  WHEN DATEDIFF(TODAY(), data) >= 21 AND DATEDIFF(TODAY(), data) <= 27 THEN "Semana -3"
  ELSE "Outro"
END
```

**Gráfico resultante:**
```
Semana Atual  │ R$ 35.2K
Semana -1     │ R$ 32.1K
Semana -2     │ R$ 31.8K
Semana -3     │ R$ 28.5K
```

---

### Se quiser por Mês (YoY - Ano Passado):

```javascript
CASE
  WHEN FORMAT_DATE("%m-%d", data) = FORMAT_DATE("%m-%d", TODAY()) THEN "Ano Passado"
  WHEN FORMAT_DATE("%m", data) = FORMAT_DATE("%m", TODAY()) THEN "Mês Atual"
  ELSE "Outro"
END

// Nota: Isso compara o MESMO DIA do ano anterior
```

---

## EXEMPLO 8: Problema Real #1 - Scorecard Mostrava Valor Errado

### Diagnóstico:

```
Você viu:
┌──────────────────┐
│   R$ 250,00      │  ← ERRADO! Deveria ser R$ 5.200
│  Receita Hoje    │
└──────────────────┘
```

### Causa Encontrada:

```
Scorecard configurado com:
├─ Métrica: PURCHASE_REVENUE ✓
├─ Filtro: period_label = "D0" ✗ FALTOU ISSO!
└─ Sem filtro = soma todas as datas!
```

### Solução:

```
1. Clique no scorecard
2. Painel direito → Filtros
3. Adicionar filtro: period_label = "D0"
4. Aplicar

Resultado:
┌──────────────────┐
│   R$ 5.200,00    │  ← CORRETO!
│  Receita Hoje    │
└──────────────────┘
```

---

## EXEMPLO 9: Problema Real #2 - Gráfico Fica Muito Lento

### Situação:

```
Você selecionou: "Últimos 90 dias"
Resultado: O gráfico leva 45 segundos para carregar ❌
```

### Análise:

```
90 dias × 2-3 visualizações × muitos cálculos = TIMEOUT
```

### Solução:

```
Opção 1 (Recomendada): Reduzir período
├─ Mudar de "Últimos 90 dias" → "Últimos 30 dias"
├─ Resultado: Carrega em 8-10 segundos ✓
└─ Trade-off: Perde visibilidade do histórico completo

Opção 2: Remover visualizações
├─ Manter apenas as mais importantes
├─ Resultado: Menos processamento
└─ Trade-off: Menos insights

Opção 3: Simplificar filtros
├─ Reduzir número de períodos (D0, D7 apenas)
├─ Resultado: Menos dados para processar
└─ Trade-off: Menos comparações
```

### Resultado após solução:

```
Antes:  "Últimos 90 dias" → 45 segundos ❌
Depois: "Últimos 30 dias" → 8 segundos ✓
```

---

## EXEMPLO 10: Checklist de Validação Rápida

### Quando terminar, faça este teste:

```
☐ Teste 1: Scorecard mostra número razoável?
   Esperado: R$ 1.000 - R$ 50.000 (depende do seu negócio)
   Se < R$ 100: Provavelmente filtro errado
   Se > R$ 1M: Provavelmente somando múltiplos dias

☐ Teste 2: Comparação percentual faz sentido?
   Esperado: ±20% é normal, ±5% pode ser variação
   Se > 100%: Pode estar certo (crescimento forte)
   Se = 0%: Dados idênticos (raro mas possível)

☐ Teste 3: Mude filtro de data e veja se tudo muda
   Mude de "Últimos 7 dias" → "Últimos 14 dias"
   Esperado: Todos os números devem mudar
   Se não mudar: Filtro de data não está aplicado

☐ Teste 4: Gráfico mostra 3 linhas diferentes?
   Esperado: D0 (azul), D7 (laranja), D30 (verde)
   Se mostra 1 linha: Filtro period_label errado
   Se mostra muitas linhas: period_label ainda tem "Outro"

☐ Teste 5: Performance é aceitável?
   Esperado: Carrega em 5-15 segundos
   Se > 30s: Reduzir período de "Últimos 90" → "Últimos 30"
   Se > 60s: Remover algumas visualizações
```

---

## EXEMPLO 11: Escalação de Complexidade

### Nível 1: MVP Básico (30 min)
```
✓ 1 Scorecard (hoje)
✓ 1 Scorecard (com comparação vs D-7)
✓ 1 Gráfico linha (3 períodos)
```

### Nível 2: Versão Expandida (1-2 horas)
```
✓ 2 Scorecards base (receita + usuários)
✓ 2 Scorecards com comparação
✓ 2 Gráficos (receita + usuários)
✓ 1 Tabela resumida
```

### Nível 3: Versão Completa (2-4 horas)
```
✓ 4 Scorecards base
✓ 4 Scorecards com comparação
✓ 3 Gráficos (receita + usuários + sessões)
✓ 2 Tabelas (resumo + detalhe)
✓ Cores customizadas
✓ Nomes em português
```

**Depois disso, considere upgrade (BigQuery)**

---

## EXEMPLO 12: Transição para Solução Profissional (Futuro)

Se seu relatório ficar muito lento ou grande, o próximo passo é:

### Opção A: Google Sheets + Lookup (Intermediária)
```
Tempo: 1-2 semanas
Custo: R$ 0
Complexidade: Média
Escalabilidade: 1M linhas

Processo:
1. Criar Google Sheet com data mapping
2. Conectar ao Data Studio via blend
3. Usar lookup para comparações
```

### Opção B: BigQuery (Profissional)
```
Tempo: 2-3 semanas
Custo: R$ 50-200/mês
Complexidade: Alta
Escalabilidade: ∞ (ilimitada)

Processo:
1. Exportar GA4 para BigQuery automaticamente
2. Criar tabela de períodos comparativos
3. Conectar ao Data Studio (muito mais rápido)
```

---

## DICA FINAL: Compartilhamento e Permissões

### Para compartilhar o relatório:

```
1. Clique em "Compartilhar" (canto superior direito)
2. Adicionar e-mail ou grupo
3. Permissões:
   ├─ "Editor": Pode modificar o relatório
   ├─ "Visualizador": Pode ver mas não editar ← Recomendado
   └─ "Visualizador de comentários": Pode comentar

Para usuários finais:
├─ Marcar "Visualizador"
├─ Assim não mexem sem querer na estrutura
└─ Eles conseguem mudar filtro de data normalmente
```

---

**Pronto! Agora você tem exemplos reais de tudo! 🎉**
