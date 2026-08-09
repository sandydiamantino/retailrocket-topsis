# RetailRocket TOPSIS

Avaliação de ranqueamento de produtos em um grafo de interações visitante–produto, comparando um baseline baseado em coocorrência com o método multicritério TOPSIS.

O projeto investiga se métricas estruturais e temporais extraídas do grafo conseguem melhorar a priorização de produtos associados a transações futuras. O ranking produzido também é interpretado como uma etapa preliminar de seleção de nós para uma futura arquitetura de recuperação baseada em grafos, como GraphRAG.

> **Escopo:** este projeto não implementa um GraphRAG completo, consulta semântica ou geração de respostas. O foco é avaliar o ranqueamento de nós/produtos antes de uma possível etapa de recuperação e geração.

## Objetivos

Os objetivos do experimento são:

1. carregar e padronizar os eventos do dataset RetailRocket;
2. construir um grafo bipartido entre visitantes e produtos;
3. calcular métricas de relevância para os produtos;
4. construir um baseline baseado exclusivamente em coocorrência;
5. construir rankings multicritério utilizando TOPSIS;
6. analisar a influência dos pesos dos critérios;
7. avaliar se os nós priorizados aparecem em transações futuras;
8. estabelecer uma base experimental para futura recuperação semântica em grafos.

## Dados

O experimento utiliza o dataset [RetailRocket](https://www.kaggle.com/datasets/retailrocket/ecommerce-dataset), especialmente:

- `events.csv`: visualizações, adições ao carrinho e transações;
- `item_properties.csv`: propriedades associadas aos produtos.

Os dados de eventos contêm, entre outras informações, identificador do visitante, identificador do produto, tipo de evento e instante da ocorrência.

## Modos de execução

O notebook possui dois modos de execução:

```python
DATA_MODE = 'sample'
```

ou:

```python
DATA_MODE = 'full'
```

| Modo | Finalidade |
|---|---|
| `sample` | Testar e validar o pipeline com uma quantidade menor de dados. |
| `full` | Executar a avaliação em maior escala com o conjunto completo disponível. |

A alteração do modo deve ser feita no início do notebook. As etapas posteriores utilizam a variável `events_active` para processar o conjunto selecionado.

## Pipeline

O processamento segue as etapas abaixo:

1. Carregamento e padronização dos dados.
2. Seleção do modo `sample` ou `full`.
3. Divisão temporal em treino e teste.
4. Construção do grafo com eventos de treino.
5. Cálculo das métricas dos produtos.
6. Construção do baseline.
7. Aplicação do TOPSIS.
8. Avaliação dos rankings em eventos futuros.
9. Geração de relatórios e tabelas comparativas.

## Divisão temporal

Para evitar vazamento de informação, o grafo e os rankings são construídos exclusivamente com eventos anteriores à data de corte:

```text
events_train: eventos anteriores à data de corte
events_test: eventos posteriores à data de corte
```

Os eventos de teste não participam do cálculo das métricas nem da escolha dos rankings. Eles são utilizados somente para verificar se os produtos priorizados aparecem posteriormente em transações.

## 6. Grafo de interação

O grafo é bipartido e possui dois tipos de nós:

- visitantes;
- produtos.

É definido por:

```text
G = (U, I, E)
```

em que:

- `U` é o conjunto de visitantes;
- `I` é o conjunto de produtos;
- `E` é o conjunto de relações visitante–produto.

Uma relação é criada quando um visitante interage com um produto. Relações repetidas podem ser consolidadas em uma única aresta e receber pesos de acordo com a intensidade ou o tipo dos eventos.

O grafo é armazenado de forma esparsa para reduzir o consumo de memória e permitir o processamento dos modos `sample` e `full`.

Os produtos são tratados como nós candidatos ao ranqueamento. O objetivo é atribuir uma pontuação de relevância a cada produto e avaliar se os produtos mais bem posicionados aparecem nas transações futuras.

## Critérios de relevância

A configuração final do TOPSIS utiliza quatro critérios, todos considerados critérios de benefício:

| Critério | Interpretação |
|---|---|
| `cooccurrence_score` | Intensidade da coocorrência entre produtos associados aos mesmos visitantes. |
| `pagerank` | Importância estrutural do produto no grafo. |
| `weighted_degree` | Intensidade agregada das interações do produto. |
| `recency_score` | Atualidade da última interação observada. |

A taxa de conversão é calculada e apresentada de forma descritiva, mas não faz parte da configuração principal do TOPSIS. Essa decisão foi tomada porque produtos com poucas interações poderiam apresentar taxa igual a 1 e receber influência excessiva no ranking.

## Baseline

A pontuação de coocorrência é utilizada como principal critério de ordenação, em ordem decrescente. Para produtos com valores iguais de coocorrência, são utilizados, sucessivamente, o grau ponderado, o PageRank e a recência como critérios de desempate.

A ordenação do baseline segue a seguinte prioridade:

```text
1. cooccurrence_score, em ordem decrescente;
2. weighted_degree, em ordem decrescente, em caso de empate;
3. pagerank, em ordem decrescente, em caso de novo empate;
4. recency_score, em ordem decrescente, em caso de novo empate.
```

O baseline funciona como uma referência simples, priorizando principalmente a coocorrência e utilizando métricas estruturais e temporais apenas para resolver empates. Ele é comparado ao TOPSIS, que combina os mesmos quatro critérios por meio de normalização, pesos e distância em relação às soluções ideal positiva e ideal negativa.

## Configuração TOPSIS

O TOPSIS compara cada produto com uma solução ideal positiva e uma solução ideal negativa. Primeiro, os critérios são normalizados; em seguida, os pesos são aplicados e são calculadas as distâncias até as duas soluções.

A pontuação de proximidade é dada por:

$$
S_i =
\frac{D_i^-}
{D_i^+ + D_i^-}
$$

Quanto maior $\mathcal{S_i}$, mais próximo o produto está da solução ideal e mais distante está da solução anti-ideal.

### Perfis de pesos

O perfil é selecionado por:

```python
WEIGHT_PROFILE = 'equal'
```

Perfis disponíveis:

| Perfil | Coocorrência | PageRank | Grau ponderado | Recência |
|---|---:|---:|---:|---:|
| `equal` | 25% | 25% | 25% | 25% |
| `cooccurrence_focus` | 40% | 20% | 20% | 20% |

O perfil `equal` é uma referência neutra. O perfil `cooccurrence_focus` é utilizado para análise de sensibilidade e aumenta a influência do critério que apresentou maior alinhamento com as transações futuras.

## Comparação entre os rankings

Os rankings podem ser comparados por meio de:

- sobreposição dos produtos nos Top 5, Top 10, Top 20, Top 50 e Top 100;
- identificação dos produtos que subiram ou caíram de posição;
- análise dos produtos exclusivos de cada ranking;
- comparação das pontuações e dos critérios associados.

Essas comparações mostram como os pesos e as métricas modificam a ordenação. Entretanto, a eficácia dos métodos é determinada principalmente pela avaliação temporal contra as transações futuras.

## Avaliação

A avaliação utiliza os eventos de `events_test` e considera os produtos associados a transações futuras.

Para cada valor de `K`, são calculados:

- `Hits@K`: quantidade de produtos relevantes no Top-K;
- `Precision@K`: proporção de itens relevantes entre os K recomendados;
- `Recall@K`: proporção dos produtos relevantes conhecidos recuperados pelo ranking.

As métricas utilizadas são:

$$
\mathrm{Hits}@K =
\left|
\mathcal{R} \cap \mathcal{L}_{K}
\right|
$$

$$
\mathrm{Precision}@K =
\frac{\mathrm{Hits}@K}{K}
$$

$$
\mathrm{Recall}@K =
\frac{\mathrm{Hits}@K}
{\left|\mathcal{R}_{\mathrm{known}}\right|}
$$

em que:

- $\mathcal{R}$ representa o conjunto de produtos relevantes;
- $\mathcal{L}_{K}$ representa os $K$ primeiros produtos do ranking;
- $\mathcal{R}_{\mathrm{known}}$ representa os produtos relevantes conhecidos no conjunto de treino.

Os valores avaliados são:

```text
K = 5, 10, 20, 50, 100
```

Produtos que aparecem pela primeira vez somente no teste são contabilizados separadamente como casos de *cold start*, pois não poderiam ser recomendados por um modelo construído apenas com o grafo de treino.

## Interpretação dos resultados no modo full

Na execução completa, foram processados:

- 2.756.101 eventos;
- 1.407.580 visitantes;
- 235.061 produtos;
- 1.123.767 visitantes no grafo de treino;
- 212.916 produtos no grafo de treino;
- 1.713.175 relações únicas visitante–produto.

Foram identificados 3.292 produtos associados a transações futuras. Desses, 3.091 eram conhecidos no treino e 201 eram novos.

O baseline apresentou o maior número de acertos em todos os cortes. Entretanto, o TOPSIS com foco em coocorrência aproximou-se do baseline nos rankings mais amplos:

| Top-K | Baseline | TOPSIS `equal` | TOPSIS `cooccurrence_focus` |
|---:|---:|---:|---:|
| 5 | 5 | 1 | 1 |
| 10 | 9 | 2 | 5 |
| 20 | 15 | 10 | 12 |
| 50 | 33 | 25 | 32 |
| 100 | 56 | 51 | 54 |

Os resultados indicam que a coocorrência foi o critério mais alinhado às transações futuras no conjunto analisado. O TOPSIS não superou o baseline, mas aproximou-se dele nos rankings mais amplos.

## Reprodutibilidade

Para executar o notebook:

1. disponibilize os arquivos do RetailRocket;
2. abra o notebook no Google Colab ou Jupyter;
3. defina `DATA_MODE` no início;
4. defina `WEIGHT_PROFILE` para selecionar os pesos;
5. execute as células em ordem;
6. examine os relatórios de métricas e avaliação.

A execução no modo `sample` é recomendada para testar o pipeline. A execução no modo `full` deve ser utilizada para os resultados principais.

## Limitações

- Foi utilizada uma única divisão temporal na avaliação principal.
- Os pesos foram definidos manualmente.
- Produtos novos no teste não podem ser recuperados pelo grafo de treino.
- A taxa de conversão não foi utilizada como critério devido à instabilidade em produtos com poucas interações.
- O experimento não inclui atributos semânticos.

Essas limitações definem oportunidades para trabalhos futuros, incluindo múltiplas divisões temporais, seleção de pesos com validação, suavização da conversão, inclusão de métricas de ranking e enriquecimento semântico do grafo.
