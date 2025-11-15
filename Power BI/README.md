## 📄 **DOCUMENTAÇÃO POWER BI**

# 📊 Implementação Power BI - Dashboards E-Commerce

## 🎨 Arquitetura de Visualização

### 🔗 Conexão com Databricks
```python
# Configuração de Conexão:
• Data Source: Azure Databricks
• Authentication: Personal Access Token
• Server: [workspace].databricks.com
• HTTP Path: /sql/1.0/warehouses/[warehouse-id]
• Refresh: Daily 6:00 AM
```

### 📊 Fontes de Dados
```python
Tabelas Principais:
• powerbi_dashboard_principal     # Visão 360° produtos
• powerbi_evolucao_vendas         # Dados temporais
• powerbi_feedback_detalhado      # Análise satisfação
• powerbi_comportamento_detalhado # Comportamento usuário
• powerbi_top10_simplificado      # Ranking produtos
• powerbi_metricas_gerais         # KPIs gerais
```

## 🎯 Dashboards Implementados

### 📄 Página 1: Dashboard Executivo

#### **Objetivo**
Visão estratégica do negócio para tomada de decisão de alto nível

#### **Componentes e Lógica**

**1. Cards de KPI (Superior)**
```python
# Configuração:
• Total Vendas: SUM(total_vendas) + ícone 📈
• Faturamento Total: SUM(faturamento_total) + formato R$
• Satisfação Média: AVERAGE(taxa_satisfacao) + formato %
• Conversão Média: AVERAGE(taxa_conversao) + formato %

# Cores:
• Azul (#1F4E79), Azul Médio (#2E75B6), Verde (#4CAF50), Laranja (#FF6B35)
```

**2. Top 10 Produtos - Score Geral**
```python
# Visual: Gráfico de Barras Horizontais
• Eixo Y: nome_produto (ordenado por score_geral DESC)
• Eixo X: score_geral
• Cor: categoria_score (Excelente/Bom/Precisa Melhorar)

# Cores Condicionais:
• Excelente (≥80): #4CAF50
• Bom (60-79): #FFC107  
• Precisa Melhorar (<60): #F44336

# Interatividade: Clique filtra outros visuais
```

**3. Matriz Performance - Satisfação vs Conversão**
```python
# Visual: Gráfico de Dispersão
• Eixo X: taxa_satisfacao
• Eixo Y: taxa_conversao
• Tamanho: faturamento_total
• Legenda: classificacao_risco

# Quadrantes:
• Alto Satisfação + Alto Conversão: Expandir
• Alto Satisfação + Baixo Conversão: Otimizar Funil
• Baixo Satisfação + Alto Conversão: Investigar Qualidade
• Baixo Satisfação + Baixo Conversão: Revisar Produto
```

**4. Grafico rosca por Categoria**
```python
# Satisfação Média:
• Categoria: categoria
• Valores: Média de taxa_satisfacao
• Cores: Escala verde (#E8F5E8 a #4CAF50)

# Conversão Média:
• Categoria: categoria  
• Valores: Média de taxa_conversao
• Cores: Escala laranja (#FFECB3 a #FF6B35)

# Desistências Totais:
• Categoria: categoria
• Valores: Soma de total_desistencias
• Cores: Escala vermelha (#FFEBEE a #F44336)
```
<img width="893" height="499" alt="image" src="https://github.com/user-attachments/assets/8bb2292b-4fb1-4445-a759-d508720391fc" />

---

### 📄 Página 2: Análise de Satisfação

#### **Objetivo**
Deep dive na experiência do cliente e identificação de oportunidades de melhoria

#### **Componentes e Lógica**

**1. Distribuição de Sentimentos**
```python
# Visual: Gráfico de Rosquinha (Donut)
• Legenda: sentimento
• Valores: Contagem de feedback_id
• Cores: Positivo (#4CAF50), Neutro (#FFC107), Negativo (#F44336)

# Insights: Percentual de clientes insatisfesos para ação imediata
```

**2. Nota Média por Produto**
```python
# Visual: Barras Horizontais
• Eixo Y: nome_produto (Top 10 por nota_media)
• Eixo X: nota_media
• Cores: Escala sequencial (#FF6B35 a #1F4E79)

# Ação: Focar em produtos com nota < 3.5 para melhorias
```

**3. Satisfação vs Performance Comercial**
```python
# Visual: Gráfico de Dispersão
• Eixo X: taxa_satisfacao
• Eixo Y: faturamento_total
• Tamanho: unidades_vendidas
• Legenda: categoria

# Análise: Identificar produtos com alta satisfação mas baixo faturamento
```

**4. Análise de Motivos de Insatisfação**
```python
# Visual: Barras Horizontais
• Eixo Y: Categoria Motivo (agrupamento de motivo_feedback)
• Eixo X: Contagem de feedback_id
• Filtro: sentimento = "Negativo"

# Categorização:
• Problemas Técnicos: Bugs, lentidão, erros
• Dificuldade de Uso: Interface complexa, configuração
• Qualidade: Não atende expectativas
• Entrega: Atrasos, problemas logísticos
• Atendimento: Suporte ao cliente
```
<img width="890" height="502" alt="image" src="https://github.com/user-attachments/assets/3a26a14d-b540-4c22-a1a7-0758fdff8fb5" />

---

### 📄 Página 3: Comportamento & Conversão

#### **Objetivo**
Otimização da jornada do usuário e taxas de conversão

#### **Componentes e Lógica**

**1. Taxa de Conversão por Produto**
```python
# Visual: Barras Horizontais
• Eixo Y: nome_produto (ordenado por taxa_conversao DESC)
• Eixo X: taxa_conversao
• Cor: nivel_chamativo

# Insights: Produtos com alto engajamento mas baixa conversão precisam de otimização
```

**2. TEMPO DE SESSÃO VS TAXA DE CONVERSÃO**
```python
# Visual: Gráfico de pizza
• Eixo X: tempo_medio_segundos
• Eixo Y: taxa_conversao
• Tamanho: total_sessoes
• Legenda: nivel_engajamento

# Análise: Tempo ideal de sessão para máxima conversão
```

**3. Cliques vs Conversão**
```python
# Visual: Gráfico de Bolhas
• Eixo X: cliques_medio_por_sessao
• Eixo Y: taxa_conversao
• Tamanho: total_sessoes
• Legenda: categoria

# Otimização: Número ideal de cliques por jornada
```

**4. Top Páginas por Engajamento**
```python
# Visual: Colunas Empilhadas
• Eixo X: nome_produto
• Colunas: Sessões por nivel_engajamento

# Breakdown:
• Baixo Engajamento: < 60 segundos
• Médio Engajamento: 60-120 segundos  
• Alto Engajamento: > 120 segundos
```
<img width="886" height="502" alt="image" src="https://github.com/user-attachments/assets/33dd140f-7635-4c5b-af1f-305c88815117" />

---

### 📄 Página 4: Análise de Desistências

#### **Objetivo**
Redução de churn e otimização de receita

#### **Componentes e Lógica**

**1. Produtos com Mais Desistências**
```python
# Visual: Barras Horizontais
• Eixo Y: nome_produto (Top 10 por total_desistencias)
• Eixo X: total_desistencias
• Cores: Escala vermelha

# Ação: Foco em produtos com > 10 desistências
```

**2. Valor Perdido por Categoria**
```python
# Visual: Grafico de rosca
• Categoria: categoria
• Valores: Soma de total_valor_perdido
• Cores: Escala vermelha

# Impacto: Identificar categorias com maior perda financeira
```

**3. Matriz Risco vs Faturamento**
```python
# Visual: Matriz
• Eixo X: total_desistencias
• Eixo Y: faturamento_total
• Tamanho: unidades_vendidas
• Legenda: classificacao_risco

# Estratégia:
• Alto Risco + Alto Faturamento: Otimizar urgentemente
• Alto Risco + Baixo Faturamento: Considerar descontinuar
• Baixo Risco + Alto Faturamento: Expandir e replicar
```
<img width="889" height="502" alt="image" src="https://github.com/user-attachments/assets/36c1f7e2-171e-42bd-8256-05a12a5d01f6" />


### 📄 Página 5: Análise Temporal

#### **Objetivo**
Identificação de tendências, sazonalidade e performance histórica

#### **Componentes e Lógica**

**1. Evolução de Vendas**
```python
# Visual: Gráfico de Linha
• Eixo X: data (hierarquia Ano → Mês → Dia)
• Eixo Y: total_vendas
• Cor: #1F4E79

# Análise: Tendência de crescimento e sazonalidade
```

**2. Sazonalidade por Dia da Semana**
```python
# Visual: Gráfico de Pizza
• Eixo X: dia_semana
• Eixo Y: Média de total_vendas
• Cores: Gradiente azul

# Insights: Melhores dias para campanhas promocionais
```

**3. Evolução por Categoria
```python
# Visual: Grafico de rosca 
• Eixo X: mes
• Eixo Y: total_vendas
• Legenda: categoria

# Análise: Crescimento comparativo entre categorias
```

**4. Calendário de Vendas**
```python
# Visual: Matriz
• Data: data
• Valores: total_vendas
• Cores: Escala azul (claro → escuro)

# Utilidade: Identificação visual de picos e vales
```
<img width="891" height="501" alt="image" src="https://github.com/user-attachments/assets/46bf3823-8fe2-4c61-aebd-f2b288072509" />

## 🎨 Design System

### Paleta de Cores
```python
Primária: #1F4E79 (Azul Marinho)
Secundária: #2E75B6 (Azul Médio)
Destaque: #FF6B35 (Laranja)
Sucesso: #4CAF50 (Verde)
Alerta: #FFC107 (Amarelo)
Erro: #F44336 (Vermelho)
Neutros: #F8F9FA, #E0E0E0, #2C3E50
```

### Tipografia
```css
Títulos: Segoe UI Bold, 14-16pt
Subtítulos: Segoe UI Semibold, 12pt
Corpo: Segoe UI Regular, 10-11pt
Destaques: Segoe UI Light, 18-28pt (cards)
```

### Layout Principles
```python
• Consistência: Mesmo padrão em todas as páginas
• Hierarquia: Informação mais importante em destaque
• Simplicidade: Gráficos claros e objetivos
• Interatividade: Filtros cruzados entre todos os visuais
• Responsividade: Adaptável a diferentes telas
```

## 🔧 Medidas DAX Principais

### KPIs Básicos
```dax
Total Vendas = SUM(powerbi_metricas_gerais[total_vendas])
Faturamento Total = SUM(powerbi_metricas_gerais[faturamento_total])
Ticket Médio = DIVIDE([Faturamento Total], [Total Vendas])
```

### Scores e Análises
```dax
Score Produto = 
VAR Satisfacao = SELECTEDVALUE(powerbi_dashboard_principal[taxa_satisfacao])
VAR Conversao = SELECTEDVALUE(powerbi_dashboard_principal[taxa_conversao])
VAR Desistencias = SELECTEDVALUE(powerbi_dashboard_principal[total_desistencias])
RETURN
    (Satisfacao * 0.4) + (Conversao * 0.4) + ((100 - (Desistencias * 10)) * 0.2)
```

### Crescimento e Comparativos
```dax
Crescimento Mensal = 
    VAR Current = [Faturamento Total]
    VAR Previous = CALCULATE([Faturamento Total], PREVIOUSMONTH('Date'[Date]))
    RETURN DIVIDE(Current - Previous, Previous, 0)
```

## 🚀 Performance Optimization

### Configurações
```python
• Modo Consulta: DirectQuery para tabelas grandes
• Agregações: Tabelas de resumo em Import Mode
• Filtros: Aplicados no nível mais alto possível
• Consultas: Otimizadas com query folding
```

### Monitoramento
```python
• Tempo de Carregamento: < 3 segundos
• Atualização: < 5 minutos
• Consumo Memória: Otimizado com agregações
• Concorrência: Suporte a 50+ usuários simultâneos
```

---

