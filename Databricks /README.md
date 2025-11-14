# 🚀 Implementação Técnica - Databricks

## 🏗️ Arquitetura do Data Pipeline

### 📁 Estrutura de Camadas

```
bronze/           # Dados crus ingeridos
├── produtos/
├── vendas/
├── feedback/
├── desistencia/
└── webanalytics/

silver/           # Dados tratados - Star Schema
├── dim_produto/
├── dim_tempo/
├── dim_cliente/
├── fato_vendas/
├── fato_feedback/
├── fato_desistencia/
└── fato_navegacao/

gold/             # Métricas de negócio
├── performance_produtos/
├── analise_feedback/
├── comportamento_cliente/
├── analise_desistencia/
├── metricas_consolidadas/
└── metricas_gerais/
```

## 🔧 Notebooks de Processamento

### 📄 Notebook 1: `01_bronze_ingestion.py`

#### **Objetivo**
Ingestão inicial dos dados crus do volume com adição de metadados

#### **Processamento**
```python
# LEITURA DOS DADOS CRUS
produtos_raw = spark.read.option("header", "true").option("delimiter", ";").csv("/Volumes/workspace/bronze/data-feed/DIM_PRODUTOS.csv")
vendas_raw = spark.read.option("header", "true").option("delimiter", ";").csv("/Volumes/workspace/bronze/data-feed/FATO_VENDAS.csv")
# [...] demais fontes

# ADIÇÃO DE METADADOS
produtos_bronze = produtos_raw.withColumn("ingestion_timestamp", current_timestamp())
vendas_bronze = vendas_raw.withColumn("ingestion_timestamp", current_timestamp())
# [...] demais tabelas

# SALVAMENTO NO CATALOG
produtos_bronze.write.mode("overwrite").saveAsTable("bronze_produtos")
vendas_bronze.write.mode("overwrite").saveAsTable("bronze_vendas")
# [...] demais tabelas
```

#### **Saída**
- `bronze_produtos`
- `bronze_vendas` 
- `bronze_feedback`
- `bronze_desistencia`
- `bronze_webanalytics`

### 📄 Codigo Bronze
 ```python
# NOTEBOOK 1 - CAMADA BRONZE

from pyspark.sql import SparkSession
from pyspark.sql.functions import *

spark = SparkSession.builder.appName("Bronze_Ingestion").getOrCreate()

print("=== CAMADA BRONZE - INGESTÃO DE DADOS ===")

# 1. Ler dados crus do volume
print("📥 Lendo dados do volume...")
produtos_bronze = spark.read.option("header", "true").option("delimiter", ";").csv("/Volumes/workspace/bronze/data-feed/DIM_PRODUTOS.csv")
vendas_bronze = spark.read.option("header", "true").option("delimiter", ";").csv("/Volumes/workspace/bronze/data-feed/FATO_VENDAS.csv")
feedback_bronze = spark.read.option("header", "true").option("delimiter", ";").csv("/Volumes/workspace/bronze/data-feed/FATO_FEEDBACK.csv")
desistencia_bronze = spark.read.option("header", "true").option("delimiter", ";").csv("/Volumes/workspace/bronze/data-feed/FATO_DESISTENCIA.csv")
webanalytics_bronze = spark.read.option("header", "true").option("delimiter", ";").csv("/Volumes/workspace/bronze/data-feed/FATO_WEBANALYTICS.csv")

# 2. Adicionar metadados de ingestão
print("📝 Adicionando metadados...")
produtos_bronze = produtos_bronze.withColumn("ingestion_timestamp", current_timestamp()) \
                                 .withColumn("ingestion_date", current_date())

vendas_bronze = vendas_bronze.withColumn("ingestion_timestamp", current_timestamp()) \
                             .withColumn("ingestion_date", current_date())

feedback_bronze = feedback_bronze.withColumn("ingestion_timestamp", current_timestamp()) \
                                .withColumn("ingestion_date", current_date())

desistencia_bronze = desistencia_bronze.withColumn("ingestion_timestamp", current_timestamp()) \
                                      .withColumn("ingestion_date", current_date())

webanalytics_bronze = webanalytics_bronze.withColumn("ingestion_timestamp", current_timestamp()) \
                                        .withColumn("ingestion_date", current_date())

# 3. Salvar no catalog
print("💾 Salvando no catalog...")
produtos_bronze.write.mode("overwrite").saveAsTable("bronze_produtos")
vendas_bronze.write.mode("overwrite").saveAsTable("bronze_vendas")
feedback_bronze.write.mode("overwrite").saveAsTable("bronze_feedback")
desistencia_bronze.write.mode("overwrite").saveAsTable("bronze_desistencia")
webanalytics_bronze.write.mode("overwrite").saveAsTable("bronze_webanalytics")

# 4. Estatísticas
print("\n📊 ESTATÍSTICAS BRONZE:")
print(f"📦 Produtos: {produtos_bronze.count()} registros")
print(f"💰 Vendas: {vendas_bronze.count()} registros")
print(f"😊 Feedback: {feedback_bronze.count()} registros")
print(f"❌ Desistência: {desistencia_bronze.count()} registros")
print(f"🌐 Web Analytics: {webanalytics_bronze.count()} registros")

# 5. Preview dos dados
print("\n🔍 PREVIEW DOS DADOS:")
print("Produtos:")
produtos_bronze.show(2)
print("\nVendas:")
vendas_bronze.show(2)

print("\n✅ CAMADA BRONZE CONCLUÍDA!")
print("📋 Tabelas criadas: bronze_produtos, bronze_vendas, bronze_feedback, bronze_desistencia, bronze_webanalytics")
```

---

### 📄 Notebook 2: `02_silver_star_schema.py`

#### **Objetivo**
Transformação dos dados crus em modelo dimensional (Star Schema)

#### **Processamento - Dimensões**

**Dim Produto:**
```python
dim_produto = produtos_bronze.select(
    col("ID_Produto").alias("produto_id"),
    col("Nome_Produto").alias("nome_produto"),
    col("Categoria").alias("categoria"),
    col("Preco_Unitario").cast("decimal(10,2)").alias("preco_unitario"),
    col("Chamativo_Cliques").alias("nivel_chamativo"),
    when(col("Preco_Unitario").cast("decimal(10,2)") < 100, "Econômico")
    .when(col("Preco_Unitario").cast("decimal(10,2)") < 300, "Premium")
    .otherwise("Luxo").alias("segmento_preco")
).distinct()
```

**Dim Tempo:**
```python
# Geração a partir de todas as datas das fontes
datas_vendas = vendas_bronze.select("Data_Pedido").distinct()
datas_feedback = feedback_bronze.select("Data_Feedback").distinct()
# [...] união de todas as fontes

dim_tempo = todas_datas.select(
    col("data"),
    to_date(col("data"), "dd/MM/yyyy").alias("data_date"),
    year(to_date(col("data"), "dd/MM/yyyy")).alias("ano"),
    month(to_date(col("data"), "dd/MM/yyyy")).alias("mes"),
    # [...] demais campos temporais
)
```

#### **Processamento - Fatos**

**Fato Vendas:**
```python
fato_vendas = vendas_bronze.select(
    col("ID_Pedido").alias("venda_id"),
    col("Data_Pedido").alias("data_venda"),
    col("ID_Cliente").alias("cliente_id"),
    col("ID_Produto").alias("produto_id"),
    col("Valor_Total_R$").cast("decimal(10,2)").alias("valor_total"),
    col("Quantidade").cast("integer").alias("quantidade"),
    (col("Valor_Total_R$").cast("decimal(10,2)") / col("Quantidade").cast("integer")).alias("preco_unitario_venda")
)
```

**Fato Feedback:**
```python
fato_feedback = feedback_bronze.select(
    col("ID_Feedback").alias("feedback_id"),
    col("ID_Pedido").alias("venda_id"),
    col("Avaliacao_Estrelas").cast("integer").alias("nota"),
    when(col("Avaliacao_Estrelas").cast("integer") >= 4, "Positivo")
    .when(col("Avaliacao_Estrelas").cast("integer") == 3, "Neutro")
    .otherwise("Negativo").alias("sentimento")
)
```

#### **Saída**
- `silver_dim_produto`
- `silver_dim_tempo`
- `silver_dim_cliente`
- `silver_fato_vendas`
- `silver_fato_feedback`
- `silver_fato_desistencia`
- `silver_fato_navegacao`

  ### 📄 Codigo Silver
 ```python
# NOTEBOOK 2 - CAMADA SILVER

from pyspark.sql import SparkSession
from pyspark.sql.functions import *

spark = SparkSession.builder.appName("Silver_StarSchema").getOrCreate()

print("=== CAMADA SILVER - STAR SCHEMA ===")

# 1. Ler dados da Bronze
print("📥 Carregando dados da Bronze...")
produtos_bronze = spark.table("bronze_produtos")
vendas_bronze = spark.table("bronze_vendas")
feedback_bronze = spark.table("bronze_feedback")
desistencia_bronze = spark.table("bronze_desistencia")
webanalytics_bronze = spark.table("bronze_webanalytics")

### DIMENSÕES ###

print("\n🔧 PROCESSANDO DIMENSÕES...")

# Dim Produto
print("📦 Dim Produto...")
dim_produto = produtos_bronze.select(
    col("ID_Produto").alias("produto_id"),
    col("Nome_Produto").alias("nome_produto"),
    col("Categoria").alias("categoria"),
    col("Tipo").alias("tipo_produto"),
    col("Preco_Unitario").cast("decimal(10,2)").alias("preco_unitario"),
    col("Chamativo_Cliques").alias("nivel_chamativo"),
    when(col("Preco_Unitario").cast("decimal(10,2)") < 100, "Econômico")
    .when(col("Preco_Unitario").cast("decimal(10,2)") < 300, "Premium")
    .otherwise("Luxo").alias("segmento_preco"),
    current_timestamp().alias("processed_at")
).distinct()

# Dim Tempo
print("📅 Dim Tempo...")
datas_vendas = vendas_bronze.select("Data_Pedido").distinct()
datas_feedback = feedback_bronze.select("Data_Feedback").distinct()
datas_desistencia = desistencia_bronze.select("Data_Cancelamento").distinct()
datas_webanalytics = webanalytics_bronze.select("Data_Click").distinct()

todas_datas = (datas_vendas
    .union(datas_feedback)
    .union(datas_desistencia)
    .union(datas_webanalytics)
    .distinct()
    .withColumnRenamed("Data_Pedido", "data")
)

dim_tempo = todas_datas.filter(col("data").isNotNull()).select(
    col("data"),
    to_date(col("data"), "dd/MM/yyyy").alias("data_date"),
    year(to_date(col("data"), "dd/MM/yyyy")).alias("ano"),
    month(to_date(col("data"), "dd/MM/yyyy")).alias("mes"),
    dayofmonth(to_date(col("data"), "dd/MM/yyyy")).alias("dia"),
    quarter(to_date(col("data"), "dd/MM/yyyy")).alias("trimestre"),
    weekofyear(to_date(col("data"), "dd/MM/yyyy")).alias("semana_ano"),
    dayofweek(to_date(col("data"), "dd/MM/yyyy")).alias("dia_semana"),
    when(dayofweek(to_date(col("data"), "dd/MM/yyyy")).isin(1, 7), "Fim de Semana")
    .otherwise("Dia de Semana").alias("tipo_dia")
).distinct()

# Dim Cliente
print("👥 Dim Cliente...")
clientes_vendas = vendas_bronze.select("ID_Cliente").distinct().filter(col("ID_Cliente").isNotNull())
dim_cliente = clientes_vendas.select(
    col("ID_Cliente").alias("cliente_id"),
    lit("Cliente Ecommerce").alias("tipo_cliente"),
    current_timestamp().alias("processed_at")
).distinct()

### FATOS ###

print("\n📊 PROCESSANDO FATOS...")

# Fato Vendas
print("💰 Fato Vendas...")
fato_vendas = vendas_bronze.select(
    col("ID_Pedido").alias("venda_id"),
    col("Data_Pedido").alias("data_venda"),
    col("ID_Cliente").alias("cliente_id"),
    col("ID_Produto").alias("produto_id"),
    col("Tipo_Produto").alias("tipo_produto"),
    col("Valor_Total_R$").cast("decimal(10,2)").alias("valor_total"),
    col("Quantidade").cast("integer").alias("quantidade"),
    col("Regiao_Entrega").alias("regiao_entrega"),
    (col("Valor_Total_R$").cast("decimal(10,2)") / col("Quantidade").cast("integer")).alias("preco_unitario_venda"),
    current_timestamp().alias("processed_at")
)

# Fato Feedback
print("😊 Fato Feedback...")
fato_feedback = feedback_bronze.select(
    col("ID_Feedback").alias("feedback_id"),
    col("ID_Pedido").alias("venda_id"),
    col("Data_Feedback").alias("data_feedback"),
    col("Avaliacao_Estrelas").cast("integer").alias("nota"),
    col("Tipo_Avaliacao").alias("tipo_avaliacao"),
    col("Motivo_Ruim_Qualitativo").alias("motivo_feedback"),
    when(col("Avaliacao_Estrelas").cast("integer") >= 4, "Positivo")
    .when(col("Avaliacao_Estrelas").cast("integer") == 3, "Neutro")
    .otherwise("Negativo").alias("sentimento"),
    current_timestamp().alias("processed_at")
)

# Fato Desistência
print("❌ Fato Desistência...")
fato_desistencia = desistencia_bronze.select(
    col("ID_Desistencia").alias("desistencia_id"),
    col("ID_Pedido").alias("venda_id"),
    col("Data_Cancelamento").alias("data_cancelamento"),
    col("Tipo_Desistencia").alias("tipo_desistencia"),
    col("Motivo_Detalhado_Desistencia").alias("motivo_desistencia"),
    current_timestamp().alias("processed_at")
)

# Fato Navegação
print("🌐 Fato Navegação...")
fato_navegacao = webanalytics_bronze.select(
    col("ID_Sessao").alias("sessao_id"),
    col("Data_Click").alias("data_click"),
    col("ID_Produto").alias("produto_id"),
    col("Cliques_na_Pagina").cast("integer").alias("total_cliques"),
    col("Tempo_Medio_Seg").cast("integer").alias("tempo_medio_segundos"),
    col("Status_Conversao").alias("status_conversao"),
    col("ID_Pedido_Convertido").alias("venda_id_convertido"),
    when(col("Status_Conversao") == "Converteu", 1).otherwise(0).alias("converteu"),
    current_timestamp().alias("processed_at")
)

# Salvar Silver no catalog
print("\n💾 Salvando Silver no catalog...")
dim_produto.write.mode("overwrite").saveAsTable("silver_dim_produto")
dim_tempo.write.mode("overwrite").saveAsTable("silver_dim_tempo")
dim_cliente.write.mode("overwrite").saveAsTable("silver_dim_cliente")
fato_vendas.write.mode("overwrite").saveAsTable("silver_fato_vendas")
fato_feedback.write.mode("overwrite").saveAsTable("silver_fato_feedback")
fato_desistencia.write.mode("overwrite").saveAsTable("silver_fato_desistencia")
fato_navegacao.write.mode("overwrite").saveAsTable("silver_fato_navegacao")

# Estatísticas
print("\n📊 ESTATÍSTICAS SILVER:")
print(f"📦 Dim Produto: {dim_produto.count()} produtos")
print(f"📅 Dim Tempo: {dim_tempo.count()} datas")
print(f"👥 Dim Cliente: {dim_cliente.count()} clientes")
print(f"💰 Fato Vendas: {fato_vendas.count()} vendas")
print(f"😊 Fato Feedback: {fato_feedback.count()} feedbacks")
print(f"❌ Fato Desistência: {fato_desistencia.count()} desistências")
print(f"🌐 Fato Navegação: {fato_navegacao.count()} sessões")

print("\n✅ CAMADA SILVER CONCLUÍDA!")
print("📋 Tabelas criadas: silver_dim_*, silver_fato_*")
```

---

### 📄 Notebook 3: `03_gold_business_metrics.py`

#### **Objetivo**
Cálculo de métricas de negócio e agregações para análise

#### **Métricas Principais**

**Performance de Produtos:**
```python
performance_produtos = (fato_vendas
    .groupBy("produto_id")
    .agg(
        sum("valor_total").alias("faturamento_total"),
        sum("quantidade").alias("unidades_vendidas"),
        count("venda_id").alias("qtd_vendas"),
        avg("valor_total").alias("ticket_medio_venda")
    )
    .join(dim_produto, "produto_id")
    .withColumn("performance", 
                when(col("unidades_vendidas") > 50, "Alta")
                .when(col("unidades_vendidas") > 20, "Média")
                .otherwise("Baixa"))
)
```

**Análise de Feedback:**
```python
analise_feedback = (fato_feedback
    .groupBy("produto_id")
    .agg(
        count("*").alias("total_feedbacks"),
        avg("nota").alias("nota_media"),
        sum(when(col("sentimento") == "Positivo", 1).otherwise(0)).alias("feedbacks_positivos"),
        sum(when(col("sentimento") == "Negativo", 1).otherwise(0)).alias("feedbacks_negativos")
    )
    .withColumn("taxa_satisfacao", 
                when(col("total_feedbacks") > 0, 
                     round((col("feedbacks_positivos") * 100.0) / col("total_feedbacks"), 2))
                .otherwise(0))
)
```

**Métricas Consolidadas:**
```python
metricas_consolidadas = (dim_produto
    .join(performance_produtos, "produto_id", "left")
    .join(analise_feedback, "produto_id", "left")
    .join(comportamento_cliente, "produto_id", "left")
    .join(analise_desistencia, "produto_id", "left")
    .na.fill(0)
    .withColumn("score_geral", 
                round(
                    (col("taxa_satisfacao") * 0.3) +
                    (col("taxa_conversao") * 0.3) +
                    (col("faturamento_total") / 1000 * 0.2) +
                    ((100 - col("total_desistencias")) * 0.2), 2
                ))
)
```

#### **Saída**
- `gold_performance_produtos`
- `gold_analise_feedback`
- `gold_comportamento_cliente`
- `gold_analise_desistencia`
- `gold_metricas_consolidadas`
- `gold_metricas_gerais`

  ### 📄 Codigo Gold
 ```python
# NOTEBOOK 3 - CAMADA GOLD

from pyspark.sql import SparkSession
from pyspark.sql.functions import *

spark = SparkSession.builder.appName("Gold_BusinessMetrics").getOrCreate()

print("=== CAMADA GOLD - MÉTRICAS DE NEGÓCIO ===")

# 1. Ler dados da Silver
print("📥 Carregando dados da Silver...")
dim_produto = spark.table("silver_dim_produto")
fato_vendas = spark.table("silver_fato_vendas")
fato_feedback = spark.table("silver_fato_feedback")
fato_desistencia = spark.table("silver_fato_desistencia")
fato_navegacao = spark.table("silver_fato_navegacao")

### MÉTRICAS DE BUSINESS ###

print("\n📊 CALCULANDO MÉTRICAS GOLD...")

# 1. PERFORMANCE DE PRODUTOS
print("📈 Performance de Produtos...")
performance_produtos = (fato_vendas
    .groupBy("produto_id")
    .agg(
        sum("valor_total").alias("faturamento_total"),
        sum("quantidade").alias("unidades_vendidas"),
        count("venda_id").alias("qtd_vendas"),
        avg("valor_total").alias("ticket_medio_venda")
    )
    .join(dim_produto, "produto_id")
    .withColumn("performance", 
                when(col("unidades_vendidas") > 50, "Alta")
                .when(col("unidades_vendidas") > 20, "Média")
                .otherwise("Baixa"))
    .select(
        "produto_id", "nome_produto", "categoria", "segmento_preco", "nivel_chamativo",
        "unidades_vendidas", "faturamento_total", "qtd_vendas", "ticket_medio_venda", "performance"
    )
)

# 2. ANÁLISE DE FEEDBACK
print("😊 Análise de Feedback...")
analise_feedback = (fato_feedback
    .groupBy("venda_id")
    .agg(
        first("nota").alias("nota"),
        first("sentimento").alias("sentimento")
    )
    .join(fato_vendas.select("venda_id", "produto_id"), "venda_id")
    .groupBy("produto_id")
    .agg(
        count("*").alias("total_feedbacks"),
        avg("nota").alias("nota_media"),
        sum(when(col("sentimento") == "Positivo", 1).otherwise(0)).alias("feedbacks_positivos"),
        sum(when(col("sentimento") == "Negativo", 1).otherwise(0)).alias("feedbacks_negativos")
    )
    .join(dim_produto, "produto_id")
    .withColumn("taxa_satisfacao", 
                when(col("total_feedbacks") > 0, 
                     round((col("feedbacks_positivos") * 100.0) / col("total_feedbacks"), 2))
                .otherwise(0))
    .select(
        "produto_id", "nome_produto", "categoria",
        "total_feedbacks", "nota_media", "feedbacks_positivos", "feedbacks_negativos", "taxa_satisfacao"
    )
)

# 3. COMPORTAMENTO E CONVERSÃO
print("🌐 Comportamento e Conversão...")
comportamento_cliente = (fato_navegacao
    .groupBy("produto_id")
    .agg(
        count("sessao_id").alias("total_sessoes"),
        sum("converteu").alias("sessoes_convertidas"),
        avg("total_cliques").alias("cliques_medio_por_sessao"),
        avg("tempo_medio_segundos").alias("tempo_medio_sessao_seg")
    )
    .join(dim_produto, "produto_id")
    .withColumn("taxa_conversao", 
                when(col("total_sessoes") > 0,
                     round((col("sessoes_convertidas") * 100.0) / col("total_sessoes"), 2))
                .otherwise(0))
    .select(
        "produto_id", "nome_produto", "categoria", "nivel_chamativo",
        "total_sessoes", "sessoes_convertidas", "taxa_conversao",
        "cliques_medio_por_sessao", "tempo_medio_sessao_seg"
    )
)

# 4. ANÁLISE DE DESISTÊNCIAS
print("❌ Análise de Desistências...")
analise_desistencia = (fato_desistencia
    .join(fato_vendas.select("venda_id", "produto_id", "valor_total"), "venda_id")
    .groupBy("produto_id")
    .agg(
        count("desistencia_id").alias("total_desistencias"),
        sum("valor_total").alias("total_valor_perdido")
    )
    .join(dim_produto, "produto_id")
    .select(
        "produto_id", "nome_produto", "categoria",
        "total_desistencias", "total_valor_perdido"
    )
)

# 5. MÉTRICAS CONSOLIDADAS
print("🎯 Métricas Consolidadas...")
metricas_consolidadas = (dim_produto
    .join(performance_produtos.select("produto_id", "unidades_vendidas", "faturamento_total", "performance"), "produto_id", "left")
    .join(analise_feedback.select("produto_id", "nota_media", "taxa_satisfacao", "feedbacks_positivos", "feedbacks_negativos"), "produto_id", "left")
    .join(comportamento_cliente.select("produto_id", "taxa_conversao", "total_sessoes", "cliques_medio_por_sessao"), "produto_id", "left")
    .join(analise_desistencia.select("produto_id", "total_desistencias", "total_valor_perdido"), "produto_id", "left")
    .na.fill(0)
    .withColumn("score_geral", 
                round(
                    (col("taxa_satisfacao") * 0.3) +
                    (col("taxa_conversao") * 0.3) +
                    (col("faturamento_total") / 1000 * 0.2) +
                    ((100 - col("total_desistencias")) * 0.2), 2
                ))
    .withColumn("classificacao_risco",
                when(col("total_desistencias") > 10, "Alto Risco")
                .when(col("total_desistencias") > 5, "Médio Risco")
                .when(col("total_desistencias") > 0, "Baixo Risco")
                .otherwise("Sem Risco"))
    .select(
        "produto_id", "nome_produto", "categoria", "segmento_preco", "nivel_chamativo",
        "unidades_vendidas", "faturamento_total", "performance",
        "nota_media", "taxa_satisfacao", "feedbacks_positivos", "feedbacks_negativos",
        "taxa_conversao", "total_sessoes", "cliques_medio_por_sessao",
        "total_desistencias", "total_valor_perdido", "classificacao_risco", "score_geral"
    )
    .orderBy(desc("score_geral"))
)

# 6. MÉTRICAS GERAIS
print("🏆 Métricas Gerais...")
metricas_gerais = spark.createDataFrame([(
    fato_vendas.count(),
    fato_vendas.agg(sum("valor_total")).first()[0] or 0,
    fato_vendas.agg(avg("valor_total")).first()[0] or 0,
    fato_feedback.count(),
    fato_feedback.filter(col("sentimento") == "Positivo").count(),
    fato_feedback.filter(col("sentimento") == "Negativo").count(),
    fato_navegacao.count(),
    fato_navegacao.filter(col("converteu") == 1).count(),
    fato_desistencia.count(),
    dim_produto.count(),
    spark.table("silver_dim_cliente").count()
)], ["total_vendas", "faturamento_total", "ticket_medio", "total_feedbacks", 
     "feedbacks_positivos", "feedbacks_negativos", "total_sessoes", 
     "sessoes_convertidas", "total_desistencias", "total_produtos", "total_clientes"])

metricas_gerais = metricas_gerais.withColumn(
    "taxa_satisfacao_geral",
    when(col("total_feedbacks") > 0, 
         round((col("feedbacks_positivos") * 100.0) / col("total_feedbacks"), 2))
    .otherwise(0)
).withColumn(
    "taxa_conversao_geral",
    when(col("total_sessoes") > 0,
         round((col("sessoes_convertidas") * 100.0) / col("total_sessoes"), 2))
    .otherwise(0)
).withColumn(
    "taxa_desistencia_geral", 
    when(col("total_vendas") > 0,
         round((col("total_desistencias") * 100.0) / col("total_vendas"), 2))
    .otherwise(0)
)

# Salvar Gold no catalog
print("\n💾 Salvando Gold no catalog...")
performance_produtos.write.mode("overwrite").saveAsTable("gold_performance_produtos")
analise_feedback.write.mode("overwrite").saveAsTable("gold_analise_feedback")
comportamento_cliente.write.mode("overwrite").saveAsTable("gold_comportamento_cliente")
analise_desistencia.write.mode("overwrite").saveAsTable("gold_analise_desistencia")
metricas_consolidadas.write.mode("overwrite").saveAsTable("gold_metricas_consolidadas")
metricas_gerais.write.mode("overwrite").saveAsTable("gold_metricas_gerais")

# Visualização
print("\n📊 VISUALIZAÇÃO GOLD:")
print("🏆 Métricas Gerais:")
metricas_gerais.show()

print("\n📈 Top 10 Produtos:")
metricas_consolidadas.select("nome_produto", "categoria", "faturamento_total", "nota_media", "score_geral").show(10)

print("\n✅ CAMADA GOLD CONCLUÍDA!")
print("📋 Tabelas criadas: gold_*")
```

---

### 📄 Notebook 4: `04_powerbi_preparation.py`

#### **Objetivo**
Preparação de views otimizadas para consumo no Power BI

#### **Processamento**

**Dashboard Principal:**
```python
dashboard_principal = spark.sql("""
SELECT 
    p.produto_id,
    p.nome_produto,
    p.categoria,
    p.segmento_preco,
    p.unidades_vendidas,
    p.faturamento_total,
    p.performance,
    p.nota_media,
    p.taxa_satisfacao,
    p.taxa_conversao,
    p.total_desistencias,
    p.classificacao_risco,
    p.score_geral
FROM gold_metricas_consolidadas p
""")
```

**Evolução Temporal:**
```python
evolucao_vendas = spark.sql("""
SELECT 
    t.data,
    t.ano,
    t.mes,
    t.trimestre,
    COUNT(v.venda_id) as total_vendas,
    SUM(v.valor_total) as faturamento_dia
FROM silver_fato_vendas v
JOIN silver_dim_tempo t ON v.data_venda = t.data
GROUP BY t.data, t.ano, t.mes, t.trimestre
""")
```

#### **Saída**
- `powerbi_dashboard_principal`
- `powerbi_evolucao_vendas`
- `powerbi_feedback_detalhado`
- `powerbi_comportamento_detalhado`
- `powerbi_metricas_gerais`

  ### 📄 Codigo Dashboard
 ```python
# NOTEBOOK 4 - PREPARAÇÃO POWER BI

from pyspark.sql import SparkSession
from pyspark.sql.functions import *

spark = SparkSession.builder.appName("PowerBI_Preparation").getOrCreate()

print("=== PREPARAÇÃO PARA POWER BI ===")

# 1. Criar views otimizadas para Power BI
print("📊 Criando views para Power BI...")

# Dashboard Principal
dashboard_principal = spark.sql("""
SELECT 
    p.produto_id,
    p.nome_produto,
    p.categoria,
    p.segmento_preco,
    p.nivel_chamativo,
    p.unidades_vendidas,
    p.faturamento_total,
    p.performance,
    p.nota_media,
    p.taxa_satisfacao,
    p.feedbacks_positivos,
    p.feedbacks_negativos,
    p.taxa_conversao,
    p.total_sessoes,
    p.cliques_medio_por_sessao,
    p.total_desistencias,
    p.total_valor_perdido,
    p.classificacao_risco,
    p.score_geral,
    -- Classificações para filtros
    CASE 
        WHEN p.nota_media >= 4.5 THEN 'Excelente'
        WHEN p.nota_media >= 4.0 THEN 'Bom' 
        WHEN p.nota_media >= 3.0 THEN 'Regular'
        ELSE 'Ruim'
    END as classificacao_nota,
    CASE 
        WHEN p.taxa_conversao >= 50 THEN 'Alta Conversão'
        WHEN p.taxa_conversao >= 30 THEN 'Média Conversão'
        WHEN p.taxa_conversao > 0 THEN 'Baixa Conversão'
        ELSE 'Sem Conversão'
    END as classificacao_conversao
FROM gold_metricas_consolidadas p
""")

# Evolução Temporal
evolucao_vendas = spark.sql("""
SELECT 
    t.data,
    t.ano,
    t.mes,
    t.trimestre,
    t.dia_semana,
    t.tipo_dia,
    p.categoria,
    COUNT(v.venda_id) as total_vendas,
    SUM(v.valor_total) as faturamento_dia,
    SUM(v.quantidade) as unidades_vendidas,
    AVG(v.valor_total) as ticket_medio_dia
FROM silver_fato_vendas v
JOIN silver_dim_tempo t ON v.data_venda = t.data
JOIN silver_dim_produto p ON v.produto_id = p.produto_id
GROUP BY t.data, t.ano, t.mes, t.trimestre, t.dia_semana, t.tipo_dia, p.categoria
""")

# Feedback Detalhado
feedback_detalhado = spark.sql("""
SELECT 
    f.feedback_id,
    f.venda_id, 
    f.data_feedback,
    f.nota,
    f.sentimento,
    f.motivo_feedback,
    p.nome_produto,
    p.categoria,
    v.valor_total
FROM silver_fato_feedback f
JOIN silver_fato_vendas v ON f.venda_id = v.venda_id
JOIN silver_dim_produto p ON v.produto_id = p.produto_id
""")

# Comportamento Detalhado
comportamento_detalhado = spark.sql("""
SELECT 
    n.sessao_id,
    n.data_click,
    n.produto_id,
    p.nome_produto,
    p.categoria,
    p.nivel_chamativo,
    n.total_cliques,
    n.tempo_medio_segundos,
    n.status_conversao,
    n.converteu,
    CASE 
        WHEN n.tempo_medio_segundos > 120 THEN 'Alto Engajamento'
        WHEN n.tempo_medio_segundos > 60 THEN 'Médio Engajamento'
        ELSE 'Baixo Engajamento'
    END as nivel_engajamento
FROM silver_fato_navegacao n
JOIN silver_dim_produto p ON n.produto_id = p.produto_id
""")

# 2. Salvar views no catalog
print("💾 Salvando views Power BI...")
dashboard_principal.write.mode("overwrite").saveAsTable("powerbi_dashboard_principal")
evolucao_vendas.write.mode("overwrite").saveAsTable("powerbi_evolucao_vendas")
feedback_detalhado.write.mode("overwrite").saveAsTable("powerbi_feedback_detalhado")
comportamento_detalhado.write.mode("overwrite").saveAsTable("powerbi_comportamento_detalhado")
spark.table("gold_metricas_gerais").write.mode("overwrite").saveAsTable("powerbi_metricas_gerais")

# 3. Verificação final
print("\n✅ VIEWS POWER BI CRIADAS:")
views = ["powerbi_dashboard_principal", "powerbi_evolucao_vendas", 
         "powerbi_feedback_detalhado", "powerbi_comportamento_detalhado",
         "powerbi_metricas_gerais"]

for view in views:
    count = spark.table(view).count()
    print(f"   📊 {view}: {count} registros")
```
---

## ⚙️ Configuração do Ambiente

### Cluster Databricks
```python
# Configuração Recomendada:
• Runtime: 12.2 LTS
• Worker Type: Standard_DS3_v2
• Min Workers: 2
• Max Workers: 8
• Auto-scaling: Enabled
```

### Dependências
```python
# Libraries:
• PySpark 3.4.0
• Pandas (para transformações complexas)
• SQL (para queries otimizadas)
```

## 🚀 Execução

### Ordem de Processamento
```bash
1. 01_bronze_ingestion.py
2. 02_silver_star_schema.py
3. 03_gold_business_metrics.py
4. 04_powerbi_preparation.py
```

### Agendamento
```python
# Job Scheduling:
• Tipo: Scheduled Job
• Frequência: Diário
• Horário: 02:00 AM
• Notificação: Email em caso de falha
```

---

**🛠️ Mantido por:** Equipe de Engenharia de Dados  
**📊 Monitoramento:** Databricks Dashboard + Alertas  
**🔐 Segurança:** Unity Catalog + RBAC
