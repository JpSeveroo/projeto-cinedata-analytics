# Documentação Técnica — Notebook `Landing_to_Bronze`

**Projeto:** CineData Analytics — Rocket Lab 2026.2
**Camada:** Bronze (Arquitetura Medalhão)

## 1. Objetivo

O notebook `Landing_to_Bronze` é o primeiro estágio do pipeline de ETL. Ele ingere os dados brutos sem nenhuma alteração estrutural ou de conteúdo, apenas adicionando metadados de controle. É a fonte da verdade bruta e rastreável que sustenta as camadas Silver e Gold.

Duas frentes de ingestão:
1. Os 5 arquivos CSV de filmes (TMDB/IMDb), convertidos em tabelas Delta Bronze.
2. A cotação do dólar via API do Banco Central, usada depois na Silver para converter valores de USD para BRL.

## 2. Catálogo, Schema e Volume

```python
spark.sql(f"CREATE CATALOG IF NOT EXISTS {catalog_name}")
spark.sql(f"CREATE SCHEMA IF NOT EXISTS {catalog_name}.{schema_name}")
spark.sql(f"CREATE VOLUME IF NOT EXISTS {catalog_name}.{schema_name}.{volume_name}")
```

Cria a estrutura de governança no Unity Catalog. O Volume é onde os CSVs brutos ficam armazenados de forma governada dentro do Lakehouse. O `IF NOT EXISTS` torna a etapa idempotente, permitindo reexecutar o notebook sem erro.

## 3. Ingestão dos CSVs

```python
for arquivo, nome_tabela in mapeamento_tabelas.items():
    df = spark.read.csv(base_path + arquivo, header=True, inferSchema=True)
    df = df.withColumn("ingestion_datetime", current_timestamp())
    df.write.mode("append").format("delta").saveAsTable(f"{catalog_name}.{schema_bronze}.{nome_tabela}")
```

Os arquivos são primeiro copiados do Workspace para o Volume. O dicionário `mapeamento_tabelas` relaciona cada CSV à sua tabela Bronze, permitindo percorrer os 5 arquivos em um único laço em vez de repetir o código cinco vezes.

Antes da ingestão, cada tabela é removida com `DROP TABLE IF EXISTS`, garantindo que cada execução comece de um estado conhecido.

`inferSchema=True` deixa o Spark inferir os tipos automaticamente. Isso é adequado porque a Bronze não deve corrigir ou padronizar nada, esse tratamento é papel da Silver.

A coluna `ingestion_datetime` registra o timestamp exato do processamento. Ela sustenta a rastreabilidade da camada e, mais à frente, a lógica de deduplicação da Silver.

A gravação usa formato Delta e modo `append`, registrando a tabela no Unity Catalog no caminho `catalog.schema.tabela`.

## 4. Widgets de Data para a API

```python
data_fim = datetime.today()
data_inicio = data_fim - timedelta(days=7)

dbutils.widgets.text("data_inicio", data_inicio_formatada)
dbutils.widgets.text("data_fim", data_fim_formatada)
```

Calcula uma janela de 7 dias corridos até a data de execução. Essa margem garante que sempre exista ao menos um dia útil com cotação disponível, já que a API do Bacen não publica valores em finais de semana e feriados.

As datas são registradas como widgets do notebook, já pré-preenchidas. Isso permite parametrizar o período externamente (por exemplo via Job agendado) sem alterar o código.

## 5. Consulta à API do Banco Central

```python
url = (
    "https://olinda.bcb.gov.br/olinda/servico/PTAX/versao/v1/odata/CotacaoDolarPeriodo("
    f"dataInicial=@dataInicial,dataFinalCotacao=@dataFinalCotacao)"
    f"?@dataInicial='{data_inicio_param}'&@dataFinalCotacao='{data_fim_param}'"
    "&$select=dataHoraCotacao,cotacaoCompra&$format=json"
)
response = requests.get(url)
```

Monta a URL do serviço PTAX no padrão OData, selecionando apenas os campos necessários (`dataHoraCotacao` e `cotacaoCompra`). O código confere o `status_code` da resposta antes de seguir, evitando que uma falha da API passe despercebida.

## 6. Persistência da Cotação

```python
df_cotacao = spark.createDataFrame(dados_json["value"])
df_cotacao.write.mode("append").format("delta").saveAsTable(f"{catalog_name}.{schema_bronze}.tb_cotacao_dolar")
```

O JSON retornado é convertido em DataFrame Spark e gravado como `bronze.tb_cotacao_dolar`, no mesmo padrão Delta/append usado nas demais tabelas.

## 7. Aderência ao Escopo

O notebook cobre os requisitos definidos para a camada Bronze: infraestrutura de catálogo/schema/volume, ingestão dos 5 CSVs mapeados nominalmente, coluna `ingestion_datetime` em cada registro, gravação em Delta com modo Append, e ingestão obrigatória da cotação do dólar via API do Bacen, com formato de data correto, uso de widgets e janela de 7 dias.

---

*Documento de apoio para revisão do notebook `Landing_to_Bronze`, projeto CineData Analytics (Rocket Lab 2026.2).*
