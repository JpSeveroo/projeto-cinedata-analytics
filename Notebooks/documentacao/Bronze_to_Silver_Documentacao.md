# Documentação Técnica — Notebook `Bronze_to_Silver`

**Projeto:** CineData Analytics — Rocket Lab 2026.2
**Camada:** Silver (Arquitetura Medalhão)

## Objetivo

O notebook lê as tabelas Bronze (dados brutos, tal como vieram da origem) e gera as 7 tabelas Silver, aplicando padronização de nomes em português, tipagem correta, limpeza de valores e deduplicação. Nenhuma tabela Bronze é alterada, tudo é lido e recriado do zero na Silver.

Configuração inicial:

```python
schema_silver = "silver"
spark.sql(f"CREATE SCHEMA IF NOT EXISTS {catalog_name}.{schema_silver}")
```

Cria o schema de destino de forma idempotente, permitindo reexecutar o notebook sem erro.

## 1. `tb_info_filmes`

Renomeia as colunas de origem para português (`id` para `id_filme`, `title` para `titulo`, etc).

```python
initcap(trim(regexp_replace(upper(col("status_filme")), "-+", " ")))
```

O status vem sujo, com variações de caixa e hífens sobressalentes. A sequência primeiro coloca tudo em maiúsculo, remove hífens repetidos, tira espaços nas pontas e por fim aplica formato título. Só depois dessa normalização é que o `when/otherwise` traduz os valores conhecidos (`Released` para `Lançado`, etc), e qualquer status que não bater com a lista vira `"Não Informado"`, atendendo à exigência de registros corrompidos não travarem o pipeline.

```python
coalesce(
    try_to_date(col("data_lancamento"), "yyyy-MM-dd"),
    try_to_date(col("data_lancamento"), "MM-dd-yyyy"),
    try_to_date(col("data_lancamento"), "dd/MM/yyyy"),
)
```

A base traz datas em formatos diferentes. `try_to_date` tenta cada padrão sem lançar erro quando não bate, e `coalesce` fica com o primeiro resultado que não for nulo. Só quando nenhum dos três formatos funciona é que a data vira `NULL`, exatamente como pedido (conversão impossível vira ausente, não erro).

`ano_lancamento` é extraído direto da data já convertida, e `duracao_minutos` usa `try_cast` para virar inteiro sem quebrar quando o valor vier corrompido.

Para a deduplicação:

```python
janela = Window.partitionBy("id_filme").orderBy(col("ingestion_datetime").desc())
```

Aqui a duplicidade é tratada mantendo apenas a linha mais recente por `ingestion_datetime`, já que cada filme deve aparecer uma única vez na tabela e o critério de "mais recente" é o exigido pelo escopo. A coluna de auditoria é descartada ao final, pois não faz parte do modelo Silver.

## 2. `tb_financeiro_filmes`

A função `limpar_campo_monetario` centraliza a higienização de `budget` e `revenue`. Ela primeiro trata textos como "Unknown" ou vazio como ausência de dado, depois remove símbolos de moeda, vírgulas e espaços, identifica sufixos de escala (B, M, K) para aplicar o multiplicador correto, converte o restante para decimal e por fim descarta valores zerados ou negativos, tratando-os como `NULL`.

Diferente da tabela anterior, aqui a consolidação de duplicados não usa `ingestion_datetime`:

```python
.groupBy("id_filme").agg(
    max("orcamento_usd").alias("orcamento_usd"),
    max("receita_usd").alias("receita_usd"),
)
```

A justificativa registrada no próprio notebook é que os valores podem estar espalhados entre linhas diferentes do mesmo filme, então escolher só a linha mais recente poderia descartar um valor válido que só existe numa linha mais antiga. Usar `max()` por filme garante que o melhor valor disponível (não nulo) seja aproveitado, em vez de arriscar perder informação.

As colunas em Real usam a cotação mais recente obtida da Bronze (`cotacao_mais_recente`, calculada em uma célula anterior a partir de `tb_cotacao_dolar`), e lucro e margem percentual são calculados com proteção contra divisão por zero e valores nulos via `when`.

## 3. `tb_metricas_engajamento`

A coluna `popularidade` é limpa em duas etapas: primeiro troca vírgula por ponto decimal (`regexp_replace`), depois converte para `double` com `try_cast`, e por fim descarta valores negativos.

Notas (`nota_media_tmdb`, `nota_media_imdb`) passam por `try_cast` para double e depois por uma validação de faixa com `between(0, 10)`, descartando qualquer valor fora da escala de negócio esperada. Contagens de votos (`qtd_votos_tmdb`, `qtd_votos_imdb`) seguem o mesmo padrão: `try_cast` para inteiro e depois checagem de não negatividade.

A consolidação final também usa `groupBy` + `max()` por filme, pelo mesmo motivo da tabela financeira: o deslocamento de colunas na origem pode fazer com que métricas válidas apareçam em linhas diferentes, e `max()` evita perder dado bom ao escolher só a linha mais recente.

## 4. `tb_avaliacoes_usuarios`

`nota_usuario` é convertida para double e validada na faixa de 0 a 10, descartando o que estiver fora. `comentario_usuario` recebe o texto padrão `"Sem comentario"` sempre que vier nulo ou só com espaços em branco, garantindo que nenhuma avaliação fique com o campo vazio.

A deduplicação usa `dropDuplicates` sobre o conjunto `id_filme + nome_usuario + nota_usuario + comentario_usuario`, removendo apenas registros integralmente idênticos, como pede o escopo.

## 5. `tb_generos`

Antes de explodir a coluna `genres`, o notebook remove aspas residuais e padroniza os separadores (`;` e `|` viram `,`), já que a origem mistura formatos de separação. Depois disso, `split` + `explode` transforma a lista de gêneros de cada filme em uma linha por gênero.

```python
generos_validos = ["Action", "Adventure", ..., "Western"]
df_etapa3 = df_etapa2.filter(col("genero").isin(generos_validos))
```

A lista de 19 gêneros válidos foi definida a partir da frequência dos valores na base: os que apareciam de forma consistente correspondem aos gêneros reais, enquanto os demais são ruído decorrente do deslocamento de colunas do CSV de origem. Filtrar por essa lista fechada resolve, de uma vez, tanto os resíduos em branco quanto textos e números deslocados que não pertencem ao domínio de gêneros.

A deduplicação usa `id_filme + genero`, sem depender do `ingestion_datetime`, pois a relação entre filme e gênero não muda com a ordem de carga.

## 6. `tb_pessoas_empresas`

As quatro origens (`cast`, `directors`, `writers`, `production_companies`) passam pelo mesmo tratamento: limpeza de aspas, padronização de separadores, explosão em uma linha por entidade, padronização de capitalização com `initcap` e marcação do `tipo_entidade` correspondente.

Depois da união das quatro bases, duas regras de validação filtram nomes que não fazem sentido como pessoa ou empresa:

```python
regra_pessoas_validas = (
    col("tipo_entidade").isin("Ator", "Diretor", "Roteirista")
    & (length(col("nome_entidade")) <= 40)
    & (~col("nome_entidade").rlike(r"[\\/()!?]"))
    & (~col("nome_entidade").rlike(r"^\d{4}\b"))
)
```

Para pessoas, nomes muito longos, com caracteres de pontuação incomuns ou que começam com um número de 4 dígitos (indício de ano deslocado para essa coluna) são descartados. Para produtoras, a regra é parecida, mas com limite de tamanho maior (55 caracteres, já que nomes de empresas tendem a ser mais longos) e uma lista de sufixos genéricos (`Inc`, `Ltd`, `Llc` etc.) que sozinhos não identificam uma empresa real.

Antes de aplicar essas regras, o código já descarta linhas que contêm extensões de imagem (`.jpg`, `.png`), que começam com `/` (indício de path de arquivo) ou que não têm nenhuma letra, sinais claros de que o valor veio de uma coluna errada por causa do deslocamento na origem.

A deduplicação final usa `id_filme + nome_entidade + tipo_entidade`, já que cada combinação representa uma relação única entre filme e entidade, sem depender de `ingestion_datetime`.

## 7. `tb_cotacao_dolar`

```python
df_cotacoes_limpas = (
    df_bronze_dolar.select(
        to_date(col("dataHoraCotacao")).alias("data"),
        col("cotacaoCompra").alias("cotacao_compra"),
    )
    .filter(col("data").isNotNull())
    .dropDuplicates(["data"])
)
```

O timestamp da cotação é reduzido para apenas a data, e duplicidades por dia são removidas. Em seguida, a data mínima e máxima presentes na base são identificadas para servir de limites do calendário.

```python
sequence(DATE '{data_minima}', DATE '{data_maxima}', INTERVAL 1 DAY)
```

Um calendário contínuo é gerado dia a dia entre a primeira e a última cotação disponível, incluindo os dias sem cotação (fins de semana e feriados). Um `LEFT JOIN` desse calendário com as cotações reais preserva todas as datas, deixando `NULL` nos dias sem valor.

```python
last("cotacao_compra", ignorenulls=True).over(janela_temporal)
```

Por fim, uma window function ordenada por data e limitada do início até a linha atual propaga o último valor não nulo para frente, implementando o Forward Fill exigido: cada dia sem cotação recebe o valor do último dia útil disponível, garantindo uma série temporal contínua para uso na conversão de moeda.

## Gravação das tabelas

Todas as tabelas Silver são gravadas com:

```python
.write.format("delta").mode("overwrite").option("overwriteSchema", "true")
```

Diferente da Bronze, aqui o modo é `overwrite` em vez de `append`, já que cada execução deve recalcular a Silver inteira a partir do estado atual da Bronze, sem acumular versões antigas. `overwriteSchema` permite que o schema da tabela mude entre execuções, caso alguma regra de limpeza passe a gerar colunas diferentes.

---

*Documento de apoio para revisão do notebook `Bronze_to_Silver`, projeto CineData Analytics (Rocket Lab 2026.2).*
