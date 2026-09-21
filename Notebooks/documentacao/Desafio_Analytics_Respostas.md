# Desafio de Analytics — Respostas

**Projeto:** CineData Analytics — Rocket Lab 2026.2
**Fonte:** Camada Gold (`fact_movies_performance`, `dim_movies`, `dim_genres`, `dim_people`, `dim_companies`, tabelas-ponte)

## 1. Receita total somada (em R$)

**R$ 837.771.066.819,07**

Soma de `receita_brl` em `fact_movies_performance`, desconsiderando registros nulos.

## 2. Top 5 filmes com maior popularidade

| Posição | Título | Popularidade |
|---|---|---|
| 1 | Blue Beetle | 2994.357 |
| 2 | Gran Turismo | 2680.593 |
| 3 | La Fellinette | 2020 |
| 4 | The Fear Footage 2: Curse of the Tape | 2019 |
| 5 | WWE Survivor Series 2018 | 2018 |

## 3. Quantidade de filmes por gênero (do maior para o menor)

| Gênero | Total de filmes |
|---|---|
| Drama | 32.287 |
| Documentary | 18.996 |
| Comedy | 18.625 |
| Thriller | 10.276 |
| Horror | 9.729 |
| Romance | 7.639 |
| Action | 6.049 |
| Science Fiction | 3.769 |
| Family | 3.722 |
| Mystery | 3.318 |
| Fantasy | 3.279 |
| Adventure | 2.870 |

Lista completa possui 19 gêneros, do `dim_genres` cruzado com `bridge_movie_genre`.

## 4. Top 10 filmes de maior receita (US$ e R$), com RANK()

| Posição | Título | Receita USD | Receita BRL |
|---|---|---|---|
| 1 | Avengers: Endgame | 2.800.000.000,00 | 14.439.320.000,00 |
| 2 | Avatar: The Way of Water | 2.320.250.281,00 | 11.965.298.674,09 |
| 3 | Avengers: Infinity War | 2.052.415.039,00 | 10.584.099.114,62 |
| 4 | Spider-Man: No Way Home | 1.921.847.111,00 | 9.910.773.366,72 |
| 5 | The Lion King | 1.663.075.401,00 | 8.576.313.535,42 |
| 6 | Top Gun: Maverick | 1.488.732.821,00 | 7.677.246.284,61 |
| 7 | Barbie | 1.428.545.028,00 | 7.366.863.854,89 |
| 8 | The Super Mario Bros. Movie | 1.355.725.263,00 | 6.991.339.608,76 |
| 9 | Black Panther | 1.349.926.083,00 | 6.961.433.817,42 |
| 10 | Star Wars: The Last Jedi | 1.332.698.830,00 | 6.872.594.596,43 |

O `RANK()` foi aplicado com `ORDER BY receita_usd DESC`, o que explica por que não há empates nessa faixa do ranking.

## 5. Ator com maior participação nos últimos 2 anos

**Kevin Hart, com 64 filmes.**

Considerando como data-âncora a última data de lançamento realizada na base (2026-02-19) e a janela dos 24 meses anteriores (a partir de 2024-02-19), Kevin Hart lidera com folga. Logo atrás, um grupo de atores (John Cena, David Chang, Forrest Conoly, Jay Hector, Nathalie Emmanuel, Greg Kriek, Stacy Hall, John Travolta, Alon Mckveen) aparece empatado com 59 participações cada.

A query usa `MAX(data_lancamento)` como âncora dinâmica em vez de `CURRENT_DATE()`, o que evita que a janela de 2 anos fique vazia caso a base não tenha dados recentes o suficiente em relação à data real de execução do notebook.

## 6. Produtora com maior lucro nos últimos 5 anos

**Universal Pictures**, com lucro total de USD 5.772.329.679,00 (R$ 29.767.326.921,64), somando 24 filmes lucrativos no período.

| Posição | Produtora | Lucro USD | Lucro BRL | Filmes lucrativos |
|---|---|---|---|---|
| 1 | Universal Pictures | 5.772.329.679,00 | 29.767.326.921,64 | 24 |
| 2 | Marvel Studios | 4.953.462.823,00 | 25.544.512.431,92 | 8 |
| 3 | Columbia Pictures | 3.662.050.755,00 | 18.884.829.538,45 | 12 |
| 4 | Pascal Pictures | 2.701.952.454,00 | 13.933.698.610,03 | 3 |
| 5 | Illumination | 2.431.353.473,00 | 12.538.246.724,91 | 3 |
| 6 | Paramount | 2.239.394.101,00 | 11.548.331.439,44 | 14 |
| 7 | Kevin Feige Productions | 2.138.205.367,00 | 11.026.511.257,08 | 4 |
| 8 | 20th Century Studios | 2.074.815.383,00 | 10.699.615.448,61 | 8 |
| 9 | Lightstorm Entertainment | 1.860.250.281,00 | 9.593.124.674,09 | 1 |
| 10 | Heyday Films | 1.490.495.872,00 | 7.686.338.162,31 | 2 |

Assim como na pergunta anterior, a data-âncora usada é a data de lançamento mais recente da base (2026-02-19), com a janela dos 5 anos (60 meses) contada a partir dela, e não da data real de hoje. Isso mantém o recorte estável e consistente independentemente de quando o notebook for executado.

---

*Respostas extraídas dos resultados de execução das queries no notebook `Silver_to_Gold`, projeto CineData Analytics (Rocket Lab 2026.2).*
