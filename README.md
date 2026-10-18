# Sprint_1_e_2_Dynamic_Programming

# Challenge Motiva: Dynamic Programming

**Escadas rolantes: manutenção orientada por dados** · FIAP, Engenharia de Software, Challenge 2026/2027

## Integrantes

| Nome | RM |
|---|---|
| Camile Vitoria Silva | RM566649 |
| Gustavo Almeida Ferreira | RM566980 |
| Lucas de Oliveira Miranda Caetano | RM568036 |
| Marco Túlio Longo Conte | RM568373 |
| Sofia Souza Rodrigues | RM566708 |

## Sobre o projeto

Este repositório traz a entrega de Dynamic Programming do Challenge Motiva. Partimos do histórico real de manutenção das escadas rolantes da Linha 4 e de leituras de sensores simuladas para:

1. calcular a **degradação acumulada** de cada escada com três estratégias (recursão pura, memoização e tabulação), medir as três e gerar o **ranking de criticidade**;
2. modelar a **rede de estações como grafo** e encontrar rotas com **BFS**, **Dijkstra** e **A\***;
3. montar o **plano de manutenção do turno** com a função `planejar_turno(base, horas)`.

Todo o código, as saídas e os gráficos estão no Notebook realizado no colab, já executado.

## O que é real e o que é simulado

| Elemento | Origem |
|---|---|
| Notas de manutenção, falhas, indisponibilidades e durações (planilha `ecr_09-26.XLSX`, aba `ZMM_ANALISE_NOTAS`) | **Real**, base fornecida pela Motiva |
| Leituras de vibração, temperatura, corrente, nível de óleo e velocidade | **Simulado**, com semente fixa (`np.random.default_rng(42)`) |
| Posições das estações, ligações e tempos de deslocamento do grafo | **Simulado** |
| Duração de 30 minutos por manutenção | **Hipótese** do projeto |

Os índices de criticidade são uma proposta acadêmica do grupo e não uma classificação oficial da Motiva. Os resultados de degradação e de rotas demonstram os algoritmos e não descrevem a condição física real das escadas.

## Estrutura do repositório

```
.
├── Sprint1e2.ipynb                        # notebook executado do início ao fim
├── README.md
└── dados/                                # gerada pela célula "Exportação dos dados utilizados"
    ├── sensores_simulados.csv            # 240 leituras simuladas (8 escadas x 30 leituras) + incremento
    ├── ranking_criticidade_historica.csv # ranking das 201 escadas com os dados reais
    ├── ranking_criticidade_final.csv     # ranking final: degradação simulada + histórico
    └── rede_estacoes.json                # vértices, coordenadas, arestas (minutos) e escadas por estação
```

## Como executar

1. Abra o `Sprint1e2.ipynb` no Google Colab.
2. Envie o arquivo `DataSetMotiva.zip` (dados fornecidos pela Motiva) para `/content`.
3. Use **Ambiente de execução → Executar tudo**.

O notebook extrai o ZIP para `/content/datasets_motiva`, lê `ecr_09-26.XLSX` e, depois da Etapa 8, grava os arquivos da pasta `dados/`. Bibliotecas usadas: `pandas`, `numpy`, `matplotlib` e a biblioteca padrão do Python (`heapq`, `collections`, `timeit`, `statistics`). Como as leituras usam semente fixa, toda execução gera os mesmos dados. Só os tempos medidos mudam de uma máquina para outra.

---

## Etapas, funções e interpretação dos resultados

### Etapa 1: carregamento e preparação dos dados históricos

A aba `ZMM_ANALISE_NOTAS` é carregada com Pandas. Combinamos data e hora de início e fim de cada avaria para calcular a duração em horas, e descartamos intervalos inválidos (fim antes do início ou datas implausíveis) em vez de estimar um valor.

**Interpretação.** A base tem 104.119 registros: 11.734 com falha marcada, 39.234 com indisponibilidade marcada e 76.778 com duração válida. Os 214 locais distintos incluem locais de nível estação, que não são escadas e são separados na etapa seguinte.

### Etapa 2: criticidade histórica (dados reais)

Antes de agrupar por escada, tratamos três problemas da base:

- mantivemos só locais no padrão `SP04-6-ECR-XXX-ERnn`, que identifica uma escada (685 linhas removidas);
- mantivemos uma linha por `Nota`, porque a mesma nota aparece em várias linhas quando tem mais de uma causa (2.580 linhas repetidas removidas);
- limitamos cada registro a 720 h (30 dias), para que apontamentos implausíveis não dominem o índice (103 registros afetados).

O índice combina os três indicadores normalizados pelo maior valor de cada um:

```
criticidade = 100 × (0,50 × falhas_norm + 0,30 × indisponibilidades_norm + 0,20 × horas_norm)
```

**Interpretação.** Analisamos 100.854 notas de 201 escadas. BUT-ER12 lidera (91,68), com 176 falhas e 561 indisponibilidades, seguida por PTA-ER16 (86,13) e PTA-ER17 (84,98). A ordem não segue um único indicador: PIN-ER24 tem o maior total de horas entre as cinco primeiras (5.280 h), mas fica em 5º por ter menos falhas (122).

| Posição | Escada | Falhas | Indisponibilidades | Horas | Criticidade |
|---|---|---|---|---|---|
| 1 | SP04-6-ECR-BUT-ER12 | 176 | 561 | 3.996,9 | 91,68 |
| 2 | SP04-6-ECR-PTA-ER16 | 179 | 472 | 3.493,2 | 86,13 |
| 3 | SP04-6-ECR-PTA-ER17 | 168 | 465 | 4.079,7 | 84,98 |
| 4 | SP04-6-ECR-BUT-ER11 | 182 | 316 | 4.175,8 | 81,70 |
| 5 | SP04-6-ECR-PIN-ER24 | 122 | 545 | 5.279,7 | 80,90 |
| 6 | SP04-6-ECR-REP-ER61 | 159 | 373 | 4.461,6 | 79,33 |
| 7 | SP04-6-ECR-LUZ-ER02 | 150 | 597 | 1.807,1 | 78,05 |
| 8 | SP04-6-ECR-REP-ER59 | 143 | 368 | 4.814,4 | 76,02 |

### Etapa 3: leituras simuladas dos sensores

A base histórica não traz leituras de sensores, então simulamos 30 leituras para cada uma das 8 escadas mais críticas, com distribuição uniforme nestas faixas: vibração de 1 a 8, temperatura de 25 a 85, corrente de 2 a 12, nível de óleo de 40 a 100 e velocidade de 0,5 a 1,0. Cada leitura vira um incremento de degradação:

```
incremento = 0,015 × (vibração / 8) + 0,010 × (temperatura / 85) + 0,010 × (corrente / 12)
           + 0,010 × (1 − óleo / 100) + 0,005 × (1 − velocidade)
```

**Interpretação.** São 240 leituras sintéticas. Pelas faixas, cada incremento fica entre cerca de 0,0065 e 0,0435, e em 30 leituras a degradação esperada fica perto de 0,75. Por isso o teto de 100% não chega a ser atingido nesta simulação. A fórmula é uma hipótese acadêmica, não um modelo industrial validado.

### Etapa 4: as três estratégias de programação dinâmica

Todas implementam a mesma recorrência, em que `D(n)` é a degradação acumulada (0 a 1) após a leitura `n`:

```
D(-1) = 0
D(n)  = min(1, D(n-1) + incremento[n])
```

| Função | Estratégia | Tempo | Espaço |
|---|---|---|---|
| `degradacao_recursiva(incrementos, n)` | Recursão pura | O(n) | O(n) de pilha |
| `degradacao_memoizada(incrementos, n, memo)` | Recursão com dicionário | O(n) | O(n) dicionário + pilha |
| `degradacao_tabular(incrementos)` | Iterativa, tabulação | O(n) | O(n) tabela |

Cada estado depende só do anterior. Por isso a recursão pura já é linear, e não exponencial, e a memoização só evita recomputação quando várias consultas compartilham o mesmo dicionário. As versões recursivas usam uma chamada por leitura e esbarram no limite de recursão do Python (cerca de 1.000 chamadas).

### Etapa 5: equivalência das três implementações

Comparamos as três versões nas 8 escadas e em todos os 30 prefixos de leituras de cada uma, e contamos as chamadas da recursão pura.

**Interpretação.** As três produziram o mesmo valor em todos os 240 casos, por exemplo 0,766808 para BUT-ER12 e 0,819960 para PTA-ER17. A contagem confirma a análise de complexidade: 5, 10, 20 e 30 leituras geram 6, 11, 21 e 31 chamadas, ou seja, uma por leitura mais o caso-base.

### Etapa 6: desempenho das implementações

Cada medição executa a função 50 vezes. Repetimos a medição 30 vezes e usamos a mediana, com séries de 5 a 900 leituras.

| Leituras | Recursão pura (µs) | Memoização (µs) | Tabulação (µs) |
|---|---|---|---|
| 5 | 0,82 | 1,47 | 0,69 |
| 30 | 4,11 | 8,31 | 3,03 |
| 100 | 13,43 | 26,91 | 9,88 |
| 300 | 55,03 | 90,60 | 28,12 |
| 900 | 219,22 | 331,22 | 81,98 |

Num segundo experimento, consultamos a degradação após cada uma das 600 primeiras leituras:

| Consultas | Recursão pura (ms) | Memoização compartilhada (ms) | Tabulação (ms) |
|---|---|---|---|
| 600 | 36,78 | 0,292 | 0,057 |

**Interpretação.** Na consulta única, os três tempos crescem de forma aproximadamente linear, como previsto. A tabulação é a mais rápida porque não tem o custo das chamadas recursivas, e a memoização é a mais lenta porque paga o dicionário sem evitar nenhum recálculo. Com várias consultas o quadro muda: a recursão pura refaz os estados anteriores a cada consulta (cerca de N²/2 chamadas), enquanto a memoização calcula cada estado uma vez e fica 126 vezes mais rápida. Concluímos que a tabulação é a melhor escolha para este problema, e que a memoização só compensa quando há consultas repetidas. Os tempos absolutos variam com a máquina, mas a ordem entre as estratégias se mantém.

### Etapa 7: ranking final de criticidade

Usamos a tabulação para calcular a degradação final de cada escada e a combinamos com o índice histórico real, com pesos de 50% cada (hipótese do projeto):

```
criticidade_final = 50 × degradação_final + 0,5 × criticidade_histórica
```

| Posição | Escada | Degradação simulada | Criticidade histórica | Criticidade final | Posição só pela degradação |
|---|---|---|---|---|---|
| 1 | SP04-6-ECR-BUT-ER12 | 0,7668 | 91,68 | 84,18 | 2 |
| 2 | SP04-6-ECR-PTA-ER17 | 0,8200 | 84,98 | 83,49 | 1 |
| 3 | SP04-6-ECR-PTA-ER16 | 0,7541 | 86,13 | 80,77 | 4 |
| 4 | SP04-6-ECR-BUT-ER11 | 0,7542 | 81,70 | 78,56 | 3 |
| 5 | SP04-6-ECR-PIN-ER24 | 0,7384 | 80,90 | 77,37 | 6 |
| 6 | SP04-6-ECR-REP-ER61 | 0,7352 | 79,33 | 76,42 | 7 |
| 7 | SP04-6-ECR-LUZ-ER02 | 0,7329 | 78,05 | 75,67 | 8 |
| 8 | SP04-6-ECR-REP-ER59 | 0,7476 | 76,02 | 75,39 | 5 |

**Interpretação.** BUT-ER12 continua em 1º, mas PTA-ER17 sobe de 3º para 2º por ter a maior degradação simulada (0,8200), à frente de PTA-ER16, que tem histórico maior. REP-ER59 seria a 5ª só pela degradação, mas fica em 8º pelo histórico mais baixo. As degradações simuladas variam pouco (0,73 a 0,82), então o histórico ainda pesa bastante na ordem final. Com leituras reais, a diferença entre as escadas tenderia a ser maior.

### Etapa 8: rede de estações como grafo, com BFS e Dijkstra

O grafo tem **13 vértices**: a base de manutenção e as **12 estações que têm escadas no dataset**. Cada escada se liga à estação indicada no seu código (`SP04-6-ECR-BUT-ER12` → `BUT`). As estações ocupam uma grade de 4 colunas, e as arestas ligam posições vizinhas, com peso de 4 a 7 minutos. O grafo é guardado como **lista de adjacência** (`grafo[no] = [(vizinho, minutos), ...]`) e exportado em [`dados/rede_estacoes.json`](dados/rede_estacoes.json).

```
BASE ── PIN ── LUZ ── PTA
 │       │      │      │
BUT ── HIG ── FAL ── FRE
 │       │      │      │
MBI ── VSO ── FRA ── REP
 │
TVS
```

| Função | O que faz |
|---|---|
| `bfs(grafo, origem, destino)` | Caminho com **menos trechos**, ignorando os pesos |
| `dijkstra(grafo, origem)` | **Menores tempos** da origem a todos os vértices, com fila de prioridade (`heapq`), e predecessores |
| `reconstruir_caminho(anteriores, origem, destino)` | Refaz a rota pelos predecessores; devolve `None` se o destino for inalcançável |

**Interpretação.** Nenhuma das 201 escadas fica sem estação. BFS e Dijkstra coincidem em parte dos destinos, mas não em todos: para HIG, o BFS escolhe BASE → PIN → HIG (2 trechos, 11 min), enquanto Dijkstra acha BASE → BUT → HIG (2 trechos, 9 min). Para FRE, VSO e REP, o caminho do BFS custa 23, 17 e 29 minutos, contra 21, 13 e 25 do Dijkstra. Isso mostra que o BFS minimiza a quantidade de trechos, e não o tempo.

| Estação | Trechos BFS | Tempo BFS (min) | Tempo Dijkstra (min) |
|---|---|---|---|
| PIN | 1 | 5 | 5 |
| BUT | 1 | 4 | 4 |
| HIG | 2 | 11 | 9 |
| VSO | 3 | 17 | 13 |
| FRE | 4 | 23 | 21 |
| REP | 5 | 29 | 25 |

### Etapa 9: A\* e validação contra Dijkstra

`astar(grafo, origem, destino, coordenadas)` busca o menor tempo até **um** destino, guiado pela heurística

```
h(n) = PESO_MINIMO × (|x_n − x_destino| + |y_n − y_destino|),  com PESO_MINIMO = 4 minutos
```

**Justificativa da heurística.** Cada aresta avança exatamente uma unidade na grade e custa pelo menos 4 minutos. Logo, chegar ao destino exige pelo menos a distância de Manhattan em arestas, e cada uma custa no mínimo 4 minutos. A heurística nunca superestima o custo restante (**admissível**), e em toda aresta vale `h(u) ≤ w(u,v) + h(v)` (**consistente**). Com isso o A\* devolve o caminho ótimo.

**Interpretação.** O A\* encontrou o mesmo custo mínimo de Dijkstra nas 12 estações a partir da base e nos **169 de 169 pares** origem-destino do grafo. As rotas também coincidem com as de Dijkstra a partir da base.

### Etapa 10: desempenho de Dijkstra × A\*

Para comparar entradas equivalentes, usamos um Dijkstra que **para ao fechar o destino**, como o A\*. Cada medição executa 2.000 vezes, com 30 medições e mediana. Também contamos os **nós expandidos**, medida que não depende da máquina.

| Métrica | Dijkstra | A\* |
|---|---|---|
| Nós expandidos (soma dos 12 destinos) | 90 | 63 |
| Mediana do tempo entre destinos (µs) | 8,75 | 10,30 |
| Destinos em que foi mais rápido | 10 | 2 |

**Interpretação.** A heurística reduz o trabalho de busca: o A\* expande cerca de 30% menos nós, e o ganho é maior nos destinos em que ela orienta bem a busca, como PTA (4 contra 10 nós). No tempo de relógio, porém, o A\* foi cerca de 18% mais lento nesta execução, porque calcular a heurística custa mais do que a economia de nós num grafo de 13 vértices. Concluímos que o A\* faz menos trabalho, mas não podemos afirmar que seja mais rápido em geral. A vantagem dele tende a aparecer em grafos maiores.

### Etapa 11: plano de manutenção do turno, `planejar_turno(base, horas)`

A função integra o ranking de criticidade e as rotas de Dijkstra:

1. percorre as escadas da mais para a menos crítica;
2. para cada uma, calcula com Dijkstra o menor caminho **a partir da posição atual da equipe**;
3. inclui a escada se o deslocamento mais os 30 minutos de manutenção ainda couberem no turno;
4. devolve quais escadas atender, em que ordem e por qual caminho.

Escadas da mesma estação têm deslocamento zero entre si. O parâmetro `retornar_base=True` também contabiliza a volta à base. Estações fora do grafo e destinos sem caminho são ignorados, e um `assert` garante que o plano nunca ultrapassa o turno.

Resultado para `planejar_turno(BASE_MANUTENCAO, 4)`:

| Ordem | Escada | Deslocamento (min) | Total (min) | Rota |
|---|---|---|---|---|
| 1 | BUT-ER12 | 4 | 34 | BASE → BUT |
| 2 | PTA-ER16 | 21 | 51 | BUT → BASE → PIN → LUZ → PTA |
| 3 | PTA-ER17 | 0 | 30 | (mesma estação) |
| 4 | BUT-ER11 | 21 | 51 | PTA → LUZ → FAL → HIG → BUT |
| 5 | PIN-ER24 | 9 | 39 | BUT → BASE → PIN |
| 6 | PIN-ER22 | 0 | 30 | (mesma estação) |

**Interpretação.** Num turno de 4 horas, o plano atende 6 escadas e usa 235 dos 240 minutos. As cinco primeiras são as mais críticas do ranking, e PIN-ER22 (58,08) entra porque está na mesma estação de PIN-ER24 e não exige deslocamento. Com o retorno à base contabilizado, o plano é o mesmo e usa os 240 minutos inteiros. A estratégia é **gulosa**: a equipe vai de BUT a PTA e volta a BUT (21 minutos em cada trecho) porque segue a ordem de criticidade sem agrupar escadas da mesma estação. O plano é sempre viável, mas não garante o ótimo global.

---

## Requisitos da entrega × onde estão atendidos

| Critério (pontos) | Onde |
|---|---|
| Função de criticidade com recursão, memoização e tabulação, com medição (20) | Etapas 4 a 7 |
| Modelagem do grafo e caminhos: lista de adjacência, BFS, Dijkstra, reconstrução de rota (25) | Etapa 8 |
| Otimização de rotas: A\* com heurística justificada, comparado ao Dijkstra (25) | Etapas 9 e 10 |
| Plano de manutenção integrado, `planejar_turno` (15) | Etapa 11 |
| Análise de desempenho, documentação e interpretação (15) | Etapas 6 e 10, docstrings das funções e este README |
| Dataset e rede em CSV, Excel ou JSON | `ecr_09-26.XLSX` (Motiva) e pasta [`dados/`](dados/), gravada pela célula de exportação |
| Interpretação de cada experimento | Célula **Interpretação** logo após cada resultado no notebook, e as seções acima |

## Conclusão

As três estratégias de programação dinâmica produzem os mesmos resultados. A contagem de chamadas mostrou que a recursão pura é linear nesta recorrência, e que a memoização só ganha quando várias consultas compartilham o dicionário. Combinamos a degradação simulada com o histórico real para gerar o ranking final de criticidade. Na rede, confirmamos que o BFS minimiza trechos, que o Dijkstra minimiza tempo e que o A\*, com heurística admissível e consistente, encontra os mesmos custos ótimos expandindo menos nós. Por fim, `planejar_turno` integrou o ranking e as rotas num plano viável para o turno.

## Limitações

- As leituras de sensores, a rede de estações, os tempos de deslocamento e a duração de 30 minutos por manutenção são simulados ou hipóteses.
- A fórmula de incremento e os pesos dos índices (50/30/20 no histórico e 50/50 no ranking final) foram definidos pelo grupo, sem validação industrial.
- O ranking histórico depende da qualidade dos apontamentos. Removemos locais que não são escadas e notas repetidas e limitamos durações a 720 h, mas notas automáticas e registros incompletos ainda podem influenciar o resultado.
- O planejamento do turno é guloso e não garante a solução ótima.
- Os tempos medidos variam com o ambiente de execução. Por isso apoiamos as conclusões também na contagem de chamadas e de nós expandidos.
