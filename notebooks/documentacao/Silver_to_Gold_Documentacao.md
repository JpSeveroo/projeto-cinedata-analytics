# Documentação Técnica — Notebook `Silver_to_Gold`

**Projeto:** CineData Analytics — Rocket Lab 2026.2
**Camada:** Gold (Arquitetura Medalhão)

## Objetivo

O notebook lê as tabelas Silver (já limpas e tipadas) e constrói o modelo dimensional (Star Schema) da camada Gold: dimensões, tabela fato, tabelas-ponte e a tabela de contexto para IA.

```python
schema_gold = "gold"
spark.sql(f"CREATE SCHEMA IF NOT EXISTS {catalog_name}.{schema_gold}")
```

## 1. `dim_movies`

```python
janela_sk = Window.orderBy("id_filme")
row_number().over(janela_sk).cast("bigint").alias("sk_movie_id")
```

A chave substituta é gerada com `row_number()` sobre uma janela ordenada por `id_filme`. Como não há `partitionBy`, a numeração é sequencial e única em toda a tabela, exatamente o comportamento esperado de uma Surrogate Key. Ordenar por `id_filme` também torna a geração determinística: rodar o notebook de novo produz as mesmas chaves, desde que os dados de origem não mudem.

### Por que `row_number()` e não `monotonically_increasing_id()` ou `sha2`

O escopo permite gerar as Surrogate Keys por qualquer uma das três opções (`monotonically_increasing_id()`, `row_number()` ou uma função de hash como `sha2`), e o notebook usa `row_number()` em todas as dimensões. Esse padrão se repete em `dim_genres`, `dim_people`, `dim_companies` e `dim_reviews`.

`monotonically_increasing_id()` gera valores únicos e crescentes, mas não sequenciais nem determinísticos: o valor depende da partição em que a linha está e do número de partições do cluster no momento da execução, então a mesma linha pode receber um ID diferente entre duas execuções, e os IDs podem ter saltos grandes entre si. Isso dificultaria tanto a leitura manual das tabelas quanto a reprodutibilidade dos resultados em reprocessamentos.

Uma função de hash como `sha2` produziria uma chave sempre igual para a mesma entrada (nome, id de origem etc.), o que garante estabilidade entre execuções sem depender de ordenação, mas gera uma chave longa (string hexadecimal) em vez de um número inteiro compacto, sendo mais indicado quando se quer evitar reprocessar toda a dimensão a cada carga (chaves incrementais/upsert), o que não é o caso aqui, já que o notebook recria toda a camada Gold em modo `overwrite` a cada execução.

`row_number()` sobre uma janela ordenada entrega exatamente o que as duas alternativas não garantem ao mesmo tempo: um inteiro sequencial, compacto e determinístico, mesmo sob reexecução, desde que a ordenação usada na janela seja estável. Como o notebook recalcula a Gold inteira do zero a cada rodada, a característica de "IDs sempre entre 1 e N, sem lacunas" facilita tanto a leitura das tabelas quanto eventuais validações manuais, sem trazer o custo de manter uma chave incremental entre cargas.

## 2. `dim_genres`

```python
.filter(col("nome_genero").isNotNull() & (col("nome_genero") != ""))
.distinct()
```

Antes de gerar a chave, o código remove nulos e strings vazias e aplica `distinct()`, garantindo que o catálogo de gêneros seja único e limpo, como pede o escopo. A janela de geração da chave (`Window.orderBy("nome_genero")`) segue o mesmo padrão determinístico da dimensão de filmes.

## 3. `dim_people`

```python
.filter(col("tipo_entidade").isin("Ator", "Diretor", "Roteirista"))
```

Filtra a tabela `tb_pessoas_empresas` para manter só pessoas físicas, excluindo produtoras (que viram uma dimensão à parte). Além de checar nulo e vazio, o filtro também descarta o valor literal `"None"`, cobrindo o caso em que um nome ausente foi convertido para essa string durante a limpeza da Silver, e não para um `NULL` real.

A janela de geração da Surrogate Key usa `orderBy("tipo_pessoa", "nome_pessoa")`, o comentário no notebook justifica que não há `partitionBy` justamente para garantir que a chave `sk_person_id` seja única em toda a tabela, e a ordenação por tipo e nome serve apenas para tornar a atribuição das chaves determinística e alfabética.

## 4. `dim_companies`

Segue a mesma lógica de `dim_people`, mas filtrando `tipo_entidade == "Produtora"` e usando `nome_produtora` como nome de saída, conforme o contrato de colunas definido no escopo.

## 5. `dim_reviews`

```python
df_avaliacoes_agrupadas = (
    df_silver_avaliacoes.filter(col("id_filme").isNotNull() & col("nota_usuario").isNotNull())
    .groupBy(col("id_filme"))
    .agg(count("nota_usuario").alias("qtd_avaliacoes_usuarios"),
         round(avg("nota_usuario"), 2).alias("nota_media_usuarios"))
)
```

Diferente das outras dimensões, `dim_reviews` já nasce como uma agregação: cada avaliação individual da Silver é resumida por filme, contando o total de avaliações e calculando a média arredondada em duas casas, exatamente como pede o contrato da tabela.

O comentário no notebook justifica agrupar antes de cruzar com `dim_movies`: reduzir o volume de linhas antes do join evita processar o join sobre a base bruta de avaliações, que é bem maior que a base já agregada por filme.

## 6. Tabelas-ponte (`bridge_movie_genre`, `bridge_movie_person`, `bridge_movie_company`)

As três seguem o mesmo padrão: partem da tabela Silver de origem, cruzam com a dimensão de filmes para resgatar `sk_movie_id` e com a dimensão periférica correspondente para resgatar a outra chave substituta, selecionam só as duas colunas de chave e aplicam `distinct()` no final.

```python
.join(df_gold_genres.select("nome_genero", "sk_genre_id"),
      df_silver_generos["genero"] == df_gold_genres["nome_genero"], how="inner")
```

Em `bridge_movie_genre`, o segundo join compara o texto do gênero (`genero`) com o nome já cadastrado na dimensão (`nome_genero`), já que a Silver de gêneros não carrega a chave substituta diretamente. Em `bridge_movie_person`, o segundo join usa o par `nome_pessoa + tipo_pessoa`, pois um mesmo nome pode existir como Ator e como Diretor, e é essa combinação que identifica a pessoa de forma única em `dim_people`.

O `distinct()` ao final de cada tabela-ponte garante que a mesma relação filme/entidade não apareça duplicada, o que poderia acontecer se a base de origem tivesse mais de uma linha equivalente após os joins.

## 7. `fact_movies_performance`

```python
df_filmes_lancados = df_gold_movies.filter(col("status_filme") == "Lançado").select("sk_movie_id", "id_filme")
```

A tabela fato parte apenas dos filmes com status `"Lançado"`, já traduzido pela Silver. Isso garante que a fato só contenha filmes efetivamente lançados, como pede o grão da tabela ("um registro único por filme").

Os dois `join` seguintes (com as métricas financeiras e de engajamento) usam `how="left"`, a partir da lista de filmes lançados. Isso preserva o grão de um registro por filme mesmo quando um filme não tem métrica financeira ou de engajamento associada, ao invés de excluí-lo da fato (o que aconteceria com um `inner join`).

Ao final, cada coluna é explicitamente convertida para o tipo exigido pelo contrato da Entrega 1 do escopo (`decimal(18,2)` para valores financeiros, `double` para notas e popularidade, `int` para contagens de votos), reforçando a tipagem correta independentemente do tipo que a coluna tinha na Silver.

## 8. `gold_genai_movies_context`

```python
concat_ws(", ", slice(collect_list("nome_pessoa"), 1, 5)).alias("atores_principais")
```

Os atores de cada filme são agregados em uma única string, limitados aos 5 primeiros nomes coletados via `slice`, o suficiente para compor um contexto textual sem tornar o documento excessivamente longo. Diretores seguem a mesma lógica de agregação, sem limite de quantidade, já que normalmente há só um ou poucos diretores por filme.

```python
concat(
    lit("O filme "), coalesce(col("titulo"), lit("Sem Título")),
    lit(", lançado no ano de "), coalesce(col("ano_lancamento").cast("string"), lit("ano não informado")),
    ...
)
```

Este é o ponto central da tabela: a construção do `llm_context_document` como frase corrida. O escopo alerta explicitamente que `concat()` retorna `NULL` para a string inteira se qualquer campo envolvido for nulo, então cada campo variável (título, ano, receita, orçamento, atores, diretor, sinopse) é protegido individualmente com `coalesce`, recebendo um texto de fallback específico (`"ano não informado"`, `"um valor não divulgado"`, `"elenco não informado"`, `"diretor não informado"`, `"Sinopse não disponível."`) quando o dado de origem vier ausente. Assim, um único campo faltante nunca faz o filme inteiro desaparecer da tabela de contexto, o problema descrito no escopo como a "casca de banana" dos nulos.

Receita e orçamento são formatados com `format_number` e prefixados com `$` antes de entrar no `coalesce`, para que o valor apareça no texto já no formato legível esperado para um documento em linguagem natural.

## Gravação

Todas as tabelas Gold usam `mode("overwrite")` com `overwriteSchema=true`, recalculando a camada inteira a cada execução a partir da Silver, sem acumular histórico, o mesmo padrão já adotado na camada Silver.

---

*Documento de apoio para revisão do notebook `Silver_to_Gold`, projeto CineData Analytics (Rocket Lab 2026.2).*
