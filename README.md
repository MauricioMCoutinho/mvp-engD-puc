# mvp-engD-puc

## Análise de Mercado e Padrões de Consumo na Plataforma Steam

## 1. Contextualização e objetivos gerais

### 1.1 Problema

O mercado global de distribuição digital de jogos eletrônicos para computadores é amplamente liderado pela plataforma Steam, caracterizando-se por um catálogo massivo em constante expansão, grande pulverização de gêneros e volatilidade de preços. Em virtude do elevado volume de títulos publicados anualmente, produtores e analistas enfrentam dificuldades para identificar quais atributos de catálogo (precificação, engajamento e recursos técnicos) realmente favorecem a retenção de público e a aprovação crítica dos usuários.

### 1.2 Objetivo

Conduzir um processo estruturado de análise de dados sobre o catálogo de jogos da plataforma Steam, identificando tendências de precificação, recepção dos consumidores, retenção de jogadores e impacto de recursos adicionais. O estudo busca responder à seguinte pergunta central, quais os padrões de consumo da plataforma steam ao longo dos anos?

### 1.3 Perguntas do negócio

1. Existe diferença de aceitação entre jogos gratuitos e pagos?
2. Qual a faixa de preço médio observada nas categorias de jogos mais bem avaliadas?
3. Como evoluiu o volume anual de novos lançamentos ao longo do período analisado?
4. De que forma a presença de conquistas (achievements) impacta a interação e fidelização da base de jogadores?
5. Existe correlação positiva entre a quantidade de conquistas disponibilizadas por um jogo e o total de avaliações registradas pelos usuários?
6. Quais são os títulos com as maiores médias históricas de tempo de jogo (average playtime)?
7. Quais gêneros concentram as maiores médias de horas jogadas por usuário?
8. Como evoluiu o preço médio dos jogos ao longo dos anos?

### 1.4 Fonte de dados e licença

Os dados foram extraídos do repositório público do Kaggle por meio do dataset Steam Games Dataset All Games, que possui os dados de mais de 136 mil jogos, totalizando 910MB de dados, além de possuir uma base de dados complementar não utilizada no trabalho que possui 5MB de dados contendo avaliações textuais. O conjunto de dados utiliza a licença MIT.

## 2. Carga dos Dados

### 2.1 Processo de ingestão

O dataset **Steam Games Dataset All Games** foi extraído do repositório público do Kaggle em formato CSV e importado para o Databricks por meio de upload manual de arquivo. Após isso foi criado a tabela `steam_games` que serve como camada Bronze do projeto.

### 2.2 Armazenamento no Databricks

Os dados brutos foram armazenados na tabela `steam_games`, no catálogo `workspace` e schema `default`. A tabela contém mais de 136 mil registros com colunas como `app_id`, `name`, `release_date`, `price`, `genres`, `positive`, `negative`, `average_playtime_forever`, `achievements`, entre outras. Esta é a tabela bronze do projeto.

## 3. Modelagem e Catálogo de Dados

### 3.1 Arquitetura Medalhão

A modelagem segue a arquitetura Medalhão, que organiza os dados em três camadas progressivas de qualidade e granularidade:

- **Bronze:** dados brutos, Tabela: `steam_games`.
- **Silver:** dados limpos, filtrados e normalizados. Tabela `steam_games_silver` (uma linha por jogo, com colunas derivadas) e `steam_generos_silver` (uma linha por jogo-gênero, com parsing e EXPLODE do array de gêneros).
- **Gold:** agregações prontas para análise, cada uma respondendo a uma pergunta de negócio específica: `steam_gold_monetizacao`, `steam_gold_preco_genero`, `steam_gold_lancamentos_ano`, `steam_gold_achievements_engajamento`, `steam_gold_correlacao_achievements`, `steam_gold_avaliacoes_por_faixa`, `steam_gold_top_playtime_titles`, `steam_gold_playtime_genero` e `steam_gold_evolucao_precos`.

### 3.2 Modelo de dados

O modelo parte da tabela `steam_games` na Bronze e dá origem a duas tabelas na Silver. A primeira, `steam_games_silver`, é a tabela principal e inclui as colunas limpas e derivadas: `ano_lancamento` (extraído de `release_date`), `modelo_monetizacao` (classificação Gratuito/Pago), `total_avaliacoes` (soma de positivas e negativas), `taxa_aprovacao_pct` (percentual de avaliações positivas) e `faixa_achievements` (classificação categorizada em 0, 1–10, 11–25, 26–50, 51–100 e 100+). A segunda, `steam_generos_silver`, é uma tabela normalizada a nível de jogo-gênero (uma linha por par jogo-gênero), criada com `LATERAL VIEW EXPLODE` sobre o array JSON `genres`, com tratamento de formato string simples e objeto JSON via `get_json_object`.

Na camada Gold, cada tabela é construída a partir das Silvers, sem joins complexos entre si. Cada uma é independente e atende a uma das perguntas propostas.

## 4. Pipeline de Dados

### 4.1 Bronze

A camada Bronze consiste na tabela `steam_games`, criada a partir do upload direto do CSV do Kaggle. Os dados são armazenados no estado bruto, sem nenhuma transformação. Apenas uma contagem total de registros e uma consulta de verificação (SELECT *) são executadas para verificação inicial.

### 4.2 Silver

A camada Silver aplica limpeza, filtros de qualidade e derivação de colunas sobre a Bronze. Duas tabelas são criadas:

1. **`steam_games_silver`** - onde é aplicado um filtro para manter apenas jogos com pelo menos uma avaliação e com data de lançamento. A partir dessa base filtrada, são derivadas novas colunas: `ano_lancamento` (extraído do `release_date`), `modelo_monetizacao` (classificação entre Gratuito e Pago), `total_avaliacoes` (soma de avaliações positivas e negativas), `taxa_aprovacao_pct` (percentual de avaliações positivas) e `faixa_achievements` (classificação da quantidade de conquistas em 6 faixas: 0, 1–10, 11–25, 26–50, 51–100 e 100+).
2. **`steam_generos_silver`** - onde o array de gêneros de cada jogo é explodido em linhas individuais (uma linha por par jogo-gênero), com tratamento de dois formatos possíveis (string simples e objeto JSON) e remoção de gêneros vazios.

### 4.3 Gold

A camada Gold cria uma tabela agregada para cada pergunta de negócio, todas construídas a partir da Silver:

1. **`steam_gold_monetizacao`** - onde a taxa de aprovação é agregada por modelo de monetização (Gratuito vs Pago).
2. **`steam_gold_preco_genero`** - onde o preço médio e a taxa de aprovação são agregados por gênero, considerando apenas gêneros com pelo menos 50 jogos pagos.
3. **`steam_gold_lancamentos_ano`** - onde o total de lançamentos é contabilizado por ano, separando jogos pagos e gratuitos.
4. **`steam_gold_achievements_engajamento`** - onde as métricas de engajamento (quantidade média de horas jogadas, recomendações, peak CCU, total de avaliações e taxa de aprovação) são agregadas por faixa de conquistas.
5. **`steam_gold_correlacao_achievements`** e **`steam_gold_avaliacoes_por_faixa`** - a primeira calcula o coeficiente de correlação de Pearson entre o número de conquistas e o total de avaliações; a segunda mostra a média de avaliações por faixa de conquistas.
6. **`steam_gold_top_playtime_titles`** - onde são listados os 20 jogos com maior tempo médio de jogo.
7. **`steam_gold_playtime_genero`** - onde a quantidade média de horas jogadas é agregada por gênero, com filtro de gêneros com pelo menos 50 jogos.
8. **`steam_gold_evolucao_precos`** - onde o preço médio, a mediana e o desvio-padrão são calculados por ano de lançamento, considerando apenas jogos pagos entre 2012 e 2026.

## 5. Qualidade de Dados

### 5.1 Outliers

O dataset contém outliers notáveis: jogos com quantidade média de horas jogadas extremamente altas e com preços extremamente altos (exemplo um jogo que custa 999 dólares). Esses valores não foram removidos. A presença de software não-jogo no catálogo (VEGAS Pro, Boom 3D) também distorce métricas de quantidade de horas jogadas. Nas agregações da camada Gold, o filtro de gêneros com pelo menos 50 jogos mitiga o efeito de categorias com poucos títulos.

### 5.2 Tratamentos realizados

Os seguintes tratamentos foram aplicados ao longo do pipeline:

- Remoção de jogos sem avaliações na Silver.
- Remoção de jogos sem data de lançamento na Silver.
- Classificação unificada de modelo de monetização (Gratuito/Pago) para resolver inconsistências entre as colunas de preço e status.
- Parsing e explosão do array JSON de gêneros com tratamento de múltiplos formatos.
- Remoção de gêneros vazios ou nulos na tabela normalizada.
- Filtro de gêneros com pelo menos 50 jogos em agregações da Gold para evitar viés de pequenas amostras.
- Exclusão de jogos gratuitos em análises de preço médio para não distorcer a média.
- Restrição temporal (2012–2026) na análise de evolução de preços.

## 6. Análise de Dados

As análises abaixo foram construídas a partir das tabelas Gold do MVP, que seguem a arquitetura Medalhão (Bronze → Silver → Gold). Cada subseção responde a uma pergunta de negócio com dados agregados da camada Silver em diante.

### 6.1 Pergunta 1

**Existe diferença de aceitação entre jogos gratuitos e pagos?**

A tabela `steam_gold_monetizacao` (Gold Q1) agregou a taxa de aprovação por modelo de monetização (Gratuito vs Pago).

**Resposta:** Sim, existe diferença. Os jogos pagos apresentam uma taxa de aprovação de **87,82%**, enquanto os jogos gratuitos ficam em **80,98%** - uma diferença de aproximadamente 7 pontos percentuais. Os jogos pagos também dominam numericamente o catálogo (74.420 títulos vs. 8.629 gratuitos). A menor aceitação dos jogos gratuitos pode estar associada à maior exposição a títulos de baixa qualidade (barreira de entrada menor), enquanto os jogos pagos tendem a passar por maior escrutínio dos consumidores antes da compra.

### 6.2 Pergunta 2

**Qual a faixa de preço médio observada nas categorias de jogos mais bem avaliadas?**

A tabela `steam_gold_preco_genero` (Gold Q2) agregou preço médio e taxa de aprovação por gênero, filtrando apenas gêneros com pelo menos 50 jogos pagos.

**Resposta:** Os gêneros com as maiores taxas de aprovação (acima de 91%) - Photo Editing, Animation & Modeling, Design & Illustration, Audio Production e Game Development - apresentam preços médios na faixa de **US$ 14 a US$ 27**, com medianas tipicamente entre US$ 20 e US$ 30. Já os gêneros de jogos mais tradicionais (Indie, Casual, RPG, Strategy), com taxas de aprovação entre 87% e 90%, têm preços médios significativamente menores (**US$ 7 a US$ 11**). Observa-se que os gêneros melhor avaliados tendem a ser categorias de software criativo/profissional com ticket mais alto, enquanto os jogos convencionais ocupam uma faixa de preço mais acessível.

### 6.3 Pergunta 3

**Como evoluiu o volume anual de novos lançamentos ao longo do período analisado?**

A tabela `steam_gold_lancamentos_ano` (Gold Q3) agregou o total de lançamentos por ano, separando jogos pagos e gratuitos.

**Resposta:** O volume de lançamentos cresceu de forma expressiva e contínua de 2012 (322 títulos) até 2024 (12.322 títulos), representando um aumento de aproximadamente **38x** no período. Observam-se três fases: (1) crescimento inicial moderado (2012–2015), (2) aceleração (2015–2018, impulsionada por ferramentas de publicação acessíveis), e (3) maturação em patamar elevado (2018–2024). A queda em 2026 reflete dados parciais (ano não completo). A proporção de jogos gratuitos cresceu até 2020 (~17%), mas voltou a cair nos anos mais recentes (~6% em 2024), indicando que o modelo pago permanece dominante.

### 6.4 Pergunta 4

**De que forma a presença de conquistas (achievements) impacta a interação e fidelização da base de jogadores?**

A tabela `steam_gold_achievements_engajamento` (Gold Q4) agregou métricas de engajamento por faixa de conquistas.

**Resposta:** Há uma relação clara e positiva entre a quantidade de conquistas e praticamente todos os indicadores de engajamento. Jogos sem conquistas têm uma quantidade média de horas jogadas de 77 minutos e taxa de aprovação de 71,2%. À medida que o número de achievements aumenta, a quantidade média de horas jogadas cresce de forma escalonada, atingindo **818 minutos** na faixa de 100+ conquistas - mais de 10x a quantidade de horas jogadas dos jogos sem conquistas. As recomendações médias passam de 766 (sem conquistas) para mais de 20.500 (100+), e o total de avaliações salta de 868 para 23.352. A taxa de aprovação também melhora, passando de 71% para ~81% na faixa 51–100, embora recue levemente para 79,7% no grupo 100+ (possivelmente por expectativas mais altas em jogos com muitas conquistas). Isso sugere que as conquistas funcionam como um mecanismo efetivo de retenção e fidelização.

### 6.5 Pergunta 5

**Existe correlação positiva entre a quantidade de conquistas disponibilizadas por um jogo e o total de avaliações registradas pelos usuários?**

A tabela `steam_gold_correlacao_achievements` (Gold Q5) calculou o coeficiente de correlação de Pearson entre o número de conquistas e o total de avaliações. A tabela `steam_gold_avaliacoes_por_faixa` mostra a média de avaliações por faixa.

**Resposta:** A correlação de Pearson entre o número absoluto de conquistas e o total de avaliações é **0,0103**, ou seja, praticamente nula. Embora a tabela por faixas mostre que jogos com mais conquistas tendem a ter mais avaliações em média (de 868 para 23.352), a correlação linear ponto-a-ponto é desprezível. Isso indica que a relação não é linear: o efeito observado nas faixas é provavelmente um reflexo indireto - jogos maiores, com maior produção e maior base de jogadores, tendem a ter tanto mais conquistas quanto mais avaliações, mas o número de conquistas por si só não é um preditor linear do volume de avaliações.

### 6.6 Pergunta 6

**Quais são os títulos com as maiores médias históricas de tempo de jogo (average playtime)?**

A tabela `steam_gold_top_playtime_titles` (Gold Q6) listou os 20 jogos com maior tempo médio de jogo.

**Resposta:** Os títulos com maior tempo médio de jogo são predominantemente jogos de nicho (visual novels asiáticos, idle games e simuladores). O primeiro colocado, *Letters From a Rainy Day*, apresenta uma média de quase **360 horas** de jogo. Observa-se a presença de software não-jogo no topo da lista (VEGAS Pro 18, com 185 horas e apenas 20,45% de aprovação), o que sugere que parte do catálogo da Steam inclui aplicações que não são jogos, e que a quantidade de horas jogadas dessas aplicações pode inflar os rankings. A maioria dos títulos é paga, com taxas de aprovação variadas (20% a 98%).

### 6.7 Pergunta 7

**Quais gêneros concentram as maiores médias de horas jogadas por usuário?**

A tabela `steam_gold_playtime_genero` (Gold Q7) agregou a quantidade média de horas jogadas por gênero (gêneros com 50+ jogos).

**Resposta:** As maiores médias de horas jogadas por usuário concentram-se em categorias de **software criativo/profissional** (Audio Production com 1.640 min, Video Production com 1.104 min, Utilities com 955 min). Entre os gêneros de jogos tradicionais, destacam-se **Massively Multiplayer** (327 min), **RPG** (246 min) e **Free To Play** (224 min) - gêneros que naturalmente favorecem sessões longas e replayabilidade. Gêneros casuais e de alta rotatividade como Indie (81 min) e Action (65 min) ficam nas últimas posições. A presença de software não-jogo no topo do ranking merece atenção para interpretação do resultado.

### 6.8 Pergunta 8

**Como evoluiu o preço médio dos jogos ao longo dos anos?**

A tabela `steam_gold_evolucao_precos` (Gold Q8) agregou preço médio, mediana e desvio-padrão por ano de lançamento (jogos pagos, 2012–2026).

**Resposta:** O preço médio dos jogos pagos caiu de **US$ 12,47 em 2013** para **US$ 8,32 em 2018** - uma redução de ~33%. Essa queda coincide com a explosão do volume de lançamentos no mesmo período, sugerindo que a popularização das ferramentas de desenvolvimento e a democratização da publicação trouxeram muitos jogos indie de baixo preço ao catálogo. A partir de 2018, o preço médio estabilizou-se na faixa de **US$ 8,30 a US$ 9,90**, com a mediana fixada em US$ 4,99–5,99. O desvio-padrão cresceu ao longo dos anos (de 8,39 para ~13), indicando maior dispersão de preços - o catálogo tornou-se mais heterogêneo, com coexistência de jogos muito baratos e títulos premium caros.

### 6.9 Discussão dos resultados

**Síntese geral:**

A análise do catálogo da Steam revela um mercado em franca expansão, com o volume de lançamentos crescendo +-38x entre 2012 e 2024. Esse crescimento foi acompanhado por uma redução e posterior estabilização do preço médio (+-US$ 9), refletindo a popularização de jogos indie acessíveis.

Quanto ao engajamento, os dados mostram uma relação robusta entre conquistas (achievements) e métricas de retenção: jogos com mais conquistas têm a quantidade média de horas jogadas até 10x maior, mais recomendações e mais avaliações. No entanto, a correlação linear entre o número absoluto de conquistas e o total de avaliações é praticamente nula (Pearson = 0,01), o que indica que o efeito não é direto - conquistas são um proxy para jogos maiores e mais produzidos, não uma causa linear isolada.

Os gêneros com maior quantidade de horas jogadas são categorias de software criativo (Audio/Video Production), que não são jogos no sentido tradicional. Entre os gêneros de jogos, Massively Multiplayer, RPG e Free To Play lideram, confirmando a expectativa de que jogos com progressão contínua e multiplayer engajam por mais tempo.

## 7. Autoavaliação

### 7.1 Objetivos atingidos

O estudo conseguiu responder as 8 perguntas com clareza, utilizando uma arquitetura de medalhão. A principal descoberta foi a relação positiva entre conquistas e engajamento: jogos com mais conquistas têm uma hora de jogo médio até 10 vezes maior, mais recomendações e mais avaliações, sugerindo que as conquistas funcionam como um mecanismo efetivo de retenção. Além disso, observou-se um boom da Steam a partir de 2012/2013, com o volume de lançamentos crescendo aproxidamente 38 vezes até 2024, impulsionado pela popularização das ferramentas de desenvolvimento e da publicação acessível na plataforma.

### 7.2 Dificuldades

- Tratamento do array JSON de gêneros, que exigiu `LATERAL VIEW EXPLODE` com `from_json` e `COALESCE` para lidar com dois formatos diferentes (string simples e objeto JSON com `description`).
- Identificação de gêneros vazios (`LENGTH = 0`) após a explosão do array, que geravam linhas inválidas na tabela `steam_generos_silver` e precisaram ser filtrados.
- Interpretação da correlação de Pearson próxima de zero (0,01) entre achievements e avaliações, que exigiu análise por faixas para entender que a relação é indireta, não linear.
- Presença de softwares (sem serem jogos) no catálogo da Steam, que altera métricas de quantidade de horas e gêneros.
- Dados de 2026 como ano parcial, que exige cuidado na interpretação das tendências.

### 7.3 Trabalhos futuros

- Cruzar os dados com a base complementar de avaliações textuais para análises sobre avaliações
- Implementar visualizações de dashboards a partir das tabelas Gold.
- Analisar de forma aprofundada sobre o boom da steam a partir do ano de 2012/2013
