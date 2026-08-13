# RetailRocket TOPSIS

Experimento de ranqueamento multicritério de produtos em um grafo bipartido visitante–produto, utilizando o dataset RetailRocket. O projeto compara um baseline hierárquico, que prioriza a coocorrência, com três configurações do método TOPSIS.

## Objetivo

O experimento investiga se a combinação de critérios comportamentais, estruturais e temporais pode melhorar a recuperação de produtos transacionados em um período posterior ao treinamento.

Os critérios utilizados são:

- coocorrência entre produtos associados aos mesmos visitantes;
- PageRank dos produtos no grafo bipartido;
- grau ponderado das relações visitante–produto;
- recência da última interação observada.

O ranking é avaliado em relação às transações posteriores à data de corte temporal.

## Dataset

O experimento utiliza o [RetailRocket Recommender System Dataset](https://www.kaggle.com/datasets/retailrocket/ecommerce-dataset), que contém eventos de interação em um ambiente de comércio eletrônico.

São utilizados os seguintes arquivos e eventos:

- `events.csv`;
- visualização de produto (`view`);
- adição ao carrinho (`addtocart`);
- transação (`transaction`).

A base utilizada no experimento completo contém:

- 2.756.101 eventos;
- 1.407.580 visitantes;
- 235.061 produtos ativos;
- período de 3 de maio a 18 de setembro de 2015.

O dataset não é distribuído neste repositório. Faça o download diretamente da fonte e disponibilize os arquivos no local esperado pelo notebook.

## Divisão temporal

Para evitar vazamento de informação, os dados são divididos por uma data de corte em 18 de agosto de 2015.

- O conjunto de treinamento contém eventos anteriores à data de corte.
- O conjunto de teste contém eventos posteriores à data de corte.
- O grafo e os rankings são construídos exclusivamente com os eventos de treinamento.
- As transações do conjunto de teste são utilizadas somente para avaliar os rankings.

No experimento completo:

- treinamento: 2.204.881 eventos e 17.864 transações;
- teste: 551.220 eventos e 4.593 transações.

## Grafo bipartido

O grafo é definido por:

```text
G = (U, I, E)
```

em que:

- `U` é o conjunto de visitantes;
- `I` é o conjunto de produtos;
- `E` é o conjunto de relações visitante–produto.

Cada relação conecta um visitante a um produto. Não são criadas relações entre dois visitantes ou entre dois produtos.

Antes da construção do grafo, os eventos são agregados por par visitante–produto. Assim, cada par corresponde a uma única aresta ponderada.

Os pesos das interações são definidos por tipo de evento:

| Evento | Peso |
|---|---:|
| `view` | 1 |
| `addtocart` | 3 |
| `transaction` | 5 |

Esses valores são uma escolha metodológica do experimento. Eles permanecem constantes em todas as configurações avaliadas e representam uma escala crescente de intensidade comportamental.

O grafo de treinamento contém:

- 1.123.767 visitantes;
- 212.916 produtos;
- 1.713.175 relações únicas visitante–produto.

## Critérios de ranqueamento

Para cada produto presente no grafo de treinamento, são calculados quatro critérios:

| Critério | Interpretação |
|---|---|
| `cooccurrence_score` | Associação ponderada entre produtos acessados pelos mesmos visitantes. |
| `pagerank` | Importância estrutural do produto no grafo. |
| `weighted_degree` | Intensidade acumulada das relações do produto com os visitantes. |
| `recency_score` | Atualidade da última interação observada. |

Todos os critérios são considerados critérios de benefício: valores maiores indicam maior relevância para o ranking.

### Coocorrência

Para cada produto, a coocorrência soma as associações ponderadas com os demais produtos presentes nos históricos dos mesmos visitantes. A contribuição de cada visitante considera o produto dos pesos das suas interações com os dois produtos.

### PageRank

O PageRank é calculado no grafo bipartido visitante–produto considerando os pesos das arestas. A configuração utiliza fator de amortecimento `d = 0,85`, limite máximo de 100 iterações e tolerância de $10^{-10}$.

### Grau ponderado

O grau ponderado corresponde à soma dos pesos das arestas incidentes em cada produto.

### Recência

A recência é calculada a partir do intervalo, em dias, entre a última interação do produto no treinamento e o último evento observado no treinamento. O escore utilizado é:

```text
RecencyScore(i) = 1 / (1 + Δi)
```

## Baseline

O baseline é uma ordenação hierárquica que prioriza a coocorrência. Os produtos são ordenados da seguinte forma:

1. `cooccurrence_score`, em ordem decrescente;
2. `weighted_degree`, em caso de empate;
3. `pagerank`, em caso de novo empate;
4. `recency_score`, em caso de novo empate.

O baseline utiliza os quatro critérios, mas apenas a coocorrência atua como critério principal. Os demais são aplicados sucessivamente para resolver empates.

## TOPSIS

O TOPSIS considera simultaneamente os quatro critérios. Os valores são normalizados, multiplicados pelos pesos dos critérios e comparados com as soluções ideal positiva e anti-ideal.

A pontuação de proximidade utilizada no ranking é:

$$
C_i = \frac{D_i^-}{D_i^+ + D_i^-}
$$

Valores maiores de $C_i$ indicam maior proximidade da solução ideal.

### Configurações de pesos

O experimento avalia três configurações:

| Configuração | Coocorrência | PageRank | Grau ponderado | Recência |
|---|---:|---:|---:|---:|
| `equal` | 25% | 25% | 25% | 25% |
| `cooccurrence_50` | 50% | 16,67% | 16,67% | 16,66% |
| `cooccurrence_70` | 70% | 10% | 10% | 10% |

A análise dos pesos é exploratória. O objetivo é verificar como a ênfase na coocorrência modifica o ranking, e não definir um vetor de pesos universalmente ótimo.

## Avaliação

São avaliados os cortes:

```text
K = 5, 10, 20, 50, 100
```

Os produtos relevantes são aqueles que aparecem em transações no conjunto de teste.

No experimento completo, foram identificados 3.292 produtos relevantes:

- 3.091 já estavam presentes no treinamento e são utilizados no cálculo das métricas;
- 201 foram observados pela primeira vez no teste e são contabilizados separadamente, pois não estavam disponíveis no grafo de treinamento.

As métricas utilizadas são:

- `Hits@K`: quantidade de produtos relevantes nos `K` primeiros itens;
- `Precision@K`: proporção de produtos relevantes entre os `K` itens recuperados;
- `Recall@K`: proporção dos produtos relevantes conhecidos recuperados pelo ranking.

## Resultados principais

A tabela apresenta o número de acertos no experimento completo:

| Corte | Baseline | TOPSIS `equal` | TOPSIS `cooccurrence_50` | TOPSIS `cooccurrence_70` |
|---:|---:|---:|---:|---:|
| Top-5 | 5 | 1 | 3 | 4 |
| Top-10 | 9 | 2 | 7 | 8 |
| Top-20 | 15 | 10 | 16 | 16 |
| Top-50 | 33 | 25 | 34 | 34 |
| Top-100 | 56 | 51 | 51 | 57 |

O baseline apresenta o melhor desempenho nos cortes Top-5 e Top-10. A configuração `cooccurrence_50` iguala ou supera o baseline nos cortes Top-20 e Top-50. A configuração `cooccurrence_70` apresenta o melhor resultado no Top-100 e supera o baseline nos cortes Top-20, Top-50 e Top-100.

A configuração com pesos iguais apresenta o menor número de acertos em todos os cortes avaliados. No conjunto analisado, a coocorrência foi o critério mais alinhado à recuperação de produtos transacionados posteriormente.

## Execução

O notebook pode ser executado em modo reduzido para testar o pipeline ou em modo completo para reproduzir os resultados principais:

```python
DATA_MODE = 'sample'
```

ou:

```python
DATA_MODE = 'full'
```

Para executar o experimento:

1. faça o download do dataset RetailRocket;
2. disponibilize os arquivos no local esperado pelo notebook;
3. abra o notebook no Google Colab ou em um ambiente Jupyter;
4. selecione `DATA_MODE` no início do notebook;
5. execute as células em ordem;
6. consulte as tabelas e os arquivos de resultados gerados.

O processamento foi desenvolvido em Python. O artigo utiliza Google Colaboratory, Python 3.12.13, Pandas 2.2.2 e NumPy 2.0.2.

## Limitações

- A avaliação principal utiliza uma única divisão temporal.
- Os pesos dos critérios são definidos manualmente.
- Produtos novos no conjunto de teste não podem ser recuperados a partir do grafo de treinamento.
- A avaliação considera produtos associados a transações e não inclui outras métricas de ranking, como NDCG ou MRR.
- O experimento não utiliza atributos semânticos dos produtos.

Possíveis extensões incluem validação em múltiplas janelas temporais, definição de pesos orientada pelos dados, inclusão de atributos semânticos e avaliação em outros grafos de interação.

## Estrutura esperada

A estrutura exata dos arquivos pode variar conforme a organização do repositório. Recomenda-se manter no projeto:

- o notebook principal do experimento;
- arquivos de configuração;
- scripts auxiliares;
- tabelas ou relatórios de resultados;
- este `README.md`.

Os dados brutos do RetailRocket não devem ser incluídos no repositório sem verificar as condições de distribuição da fonte original.
