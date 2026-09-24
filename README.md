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
4. De que forma a presença de conquistas (achievements) impacta a taxa de aprovação e média de horas jogadas?
5. Quais são os títulos com as maiores médias históricas de tempo de jogo (average playtime)?
6. Quais gêneros concentram as maiores médias de horas jogadas por usuário?
7. Como evoluiu o preço médio dos jogos ao longo dos anos?

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
- **Gold:** agregações prontas para análise, cada uma respondendo a uma pergunta específica: `steam_gold_monetizacao`, `steam_gold_preco_genero`, `steam_gold_lancamentos_ano`, `steam_gold_achievements_engajamento`, `steam_gold_top_playtime_titles`, `steam_gold_playtime_genero` e `steam_gold_evolucao_precos`.

### 3.2 Modelo de dados

O modelo parte da tabela `steam_games` na Bronze e dá origem a duas tabelas na Silver. A primeira, `steam_games_silver`, é a tabela principal e inclui as colunas limpas e derivadas: `ano_lancamento` (extraído de `release_date`), `modelo_monetizacao` (classificação Gratuito/Pago), `total_avaliacoes` (soma de positivas e negativas), `taxa_aprovacao_pct` (percentual de avaliações positivas) e `faixa_achievements` (classificação categorizada em 0, 1–10, 11–25, 26–50, 51–100 e 100+). A segunda, `steam_generos_silver`, é uma tabela normalizada a nível de jogo-gênero (uma linha por par jogo-gênero), criada com `LATERAL VIEW EXPLODE` sobre o array JSON `genres`, com tratamento de formato string simples e objeto JSON via `get_json_object`.

Na camada Gold, cada tabela é construída a partir das Silvers, sem joins complexos entre si. Cada uma é independente e atende a uma das perguntas propostas.

## 4. Pipeline de Dados

### 4.1 Bronze

A camada Bronze consiste na tabela `steam_games`, criada a partir do upload direto do CSV do Kaggle. Os dados são armazenados no estado bruto, sem nenhuma transformação. Apenas uma contagem total de registros e uma consulta de verificação (SELECT *) são executadas para verificação inicial.

### 4.2 Silver

A camada Silver aplica limpeza, filtros de qualidade e derivação de colunas sobre a Bronze. Duas tabelas são criadas:

1. **`steam_games_silver`** - onde é aplicado um filtro para manter apenas jogos com pelo menos uma avaliação, com data de lançamento, que não sejam aplicativos de software e que não possuam o gênero Free To Play (este gênero está errado no dataset). A partir dessa base filtrada, são derivadas novas colunas: `ano_lancamento` (extraído do `release_date`), `modelo_monetizacao` (classificação entre Gratuito e Pago), `total_avaliacoes` (soma de avaliações positivas e negativas), `taxa_aprovacao_pct` (percentual de avaliações positivas) e `faixa_achievements` (classificação da quantidade de conquistas em 6 faixas: 0, 1–10, 11–25, 26–50, 51–100 e 100+).
2. **`steam_generos_silver`** - onde o array de gêneros de cada jogo é explodido em linhas individuais (uma linha por par jogo-gênero), com tratamento de dois formatos possíveis (string simples e objeto JSON) e remoção de gêneros vazios.

### 4.3 Gold

A camada Gold cria uma tabela agregada para cada pergunta, todas construídas a partir da Silver:

1. **`steam_gold_monetizacao`** - onde a taxa de aprovação é agregada por modelo de monetização (Gratuito vs Pago).
2. **`steam_gold_preco_genero`** - onde o preço médio e a taxa de aprovação são agregados por gênero, considerando apenas gêneros com pelo menos 50 jogos pagos.
3. **`steam_gold_lancamentos_ano`** - onde o total de lançamentos é contabilizado por ano, separando jogos pagos e gratuitos.
4. **`steam_gold_achievements_engajamento`** - onde as métricas de engajamento (quantidade média de horas jogadas, recomendações, peak CCU, total de avaliações e taxa de aprovação) são agregadas por faixa de conquistas.
5. **`steam_gold_top_playtime_titles`** - onde são listados os 20 jogos com maior tempo médio de jogo.
6. **`steam_gold_playtime_genero`** - onde a quantidade média de horas jogadas é agregada por gênero, com filtro de gêneros com pelo menos 50 jogos.
7. **`steam_gold_evolucao_precos`** - onde o preço médio, a mediana e o desvio-padrão são calculados por ano de lançamento, considerando apenas jogos pagos entre 2012 e 2026.

## 5. Qualidade de Dados

### 5.1 Outliers

O dataset contém outliers notáveis: jogos com quantidade média de horas jogadas extremamente altas e com preços extremamente altos (exemplo um jogo que custa 999 dólares). Esses valores não foram removidos. Nas agregações da camada Gold, o filtro de gêneros com pelo menos 50 jogos mitiga o efeito de categorias com poucos títulos.

### 5.2 Tratamentos realizados

Os seguintes tratamentos foram aplicados ao longo do pipeline:

- Remoção de jogos sem avaliações na Silver.
- Remoção de jogos sem data de lançamento na Silver.
- Remoção de aplicativos de software do catálogo na Silver, filtrando gêneros como Audio Production, Video Production, Utilities, Animation & Modeling, Photo Editing, Design & Illustration, Web Publishing, Software Training e Game Development.
- Remoção do gênero Free To Play na Silver, pois ele está errado no dataset.
- Classificação unificada de modelo de monetização (Gratuito/Pago) para resolver inconsistências entre as colunas de preço e status.
- Parsing e explosão do array JSON de gêneros com tratamento de múltiplos formatos.
- Remoção de gêneros vazios ou nulos na tabela normalizada.
- Filtro de gêneros com pelo menos 50 jogos em agregações da Gold para evitar viés de pequenas amostras.
- Exclusão de jogos gratuitos em análises de preço médio para não distorcer a média.
- Restrição temporal (2012–2026) na análise de evolução de preços.

## 6. Análise de Dados

As análises abaixo foram construídas a partir das tabelas Gold, que seguem a arquitetura Medalhão. Cada subseção responde a uma pergunta com dados agregados da camada Silver em diante.

### 6.1 Pergunta 1

**Existe diferença de aceitação entre jogos gratuitos e pagos?**

A tabela `steam_gold_monetizacao` (Gold Q1) agregou a taxa de aprovação por modelo de monetização (Gratuito vs Pago).

**Resposta:** Sim, existe diferença. Os jogos pagos apresentam uma taxa de aprovação de **87,74%**, enquanto os jogos gratuitos ficam em **83,30%**, uma diferença de aproximadamente 4,5%. Existem também muito mais jogos pagos do que gratuitos na plataforma da steam (73388 jogos pagos vs. 3640 gratuitos). A taxa de aprovação menor dos jogos gratuitos pode estar associada à maior facilidade de publicação nesse modelo, o que resulta em uma quantidade maior de títulos de baixa qualidade, enquanto que nos jogos pagos, os jogadores costumam pesquisam mais a fundo sobre o jogo antes de comprar.

### 6.2 Pergunta 2

**Qual a faixa de preço médio observada nas categorias de jogos mais bem avaliadas?**

A tabela `steam_gold_preco_genero` (Gold Q2) agregou preço médio e taxa de aprovação por gênero, filtrando apenas gêneros com pelo menos 50 jogos pagos.

**Resposta:** Os gêneros com as maiores taxas de aprovação são Indie (89,74%, US$ 7,93), Casual (88,92%, US$ 7,49), Simulation (88,60%, US$ 10,95), Adventure (87,51%, US$ 9,84) e Strategy (87,41%, US$ 10,36). No geral, os gêneros mais bem avaliados têm preços médios entre US$ 7 e US$ 11, indicando que na Steam a aprovação não está ligada a preços mais altos.

### 6.3 Pergunta 3

**Como evoluiu o volume anual de novos lançamentos ao longo do período analisado?**

A tabela `steam_gold_lancamentos_ano` (Gold Q3) agregou o total de lançamentos por ano, separando jogos pagos e gratuitos.

**Resposta:** O volume de lançamentos cresceu de forma expressiva e contínua de 2012 (296 jogos) até 2024 (11611 jogos), estabilizando em 2025. A queda em 2026 se deve aos dados do dataset estarem incompletos. A proporção de jogos gratuitos cresceu até 2020, mas voltou a cair nos anos mais recentes, indicando que os jogos pagos permanecem sendo os mais lançados na plataforma.

### 6.4 Pergunta 4

**De que forma a presença de conquistas (achievements) impacta a taxa de aprovação e média de horas jogadas?**

A tabela `steam_gold_achievements_engajamento` (Gold Q4) agregou métricas de engajamento por faixa de conquistas.

**Resposta:** Há uma relação clara e positiva entre a quantidade de conquistas e praticamente todos os indicadores de engajamento. Jogos sem conquistas têm uma quantidade média de horas jogadas de 55 minutos e taxa de aprovação de 71%. Quanto mais conquistas no jogo, maior é a quantidade média de horas jogadas, atingindo 782 minutos na faixa de 100+ conquistas. As recomendações médias passam de 821 (0 conquistas) para mais de 21700 (100 ou mais conquistas), e o total de avaliações aumenta de 753 para 21542. A taxa de aprovação também melhora, passando de 71% para 81,29% na faixa de 51 a 100 conquistas, embora diminua um pouco (80,1%) no grupo 100 ou mais (possivelmente por expectativas mais altas em jogos com muitas conquistas). Isso sugere que as conquistas funcionam como um mecanismo efetivo de aprovação e retenção.

### 6.5 Pergunta 5

**Quais são os títulos com as maiores médias de tempo de jogo (average playtime)?**

A tabela `steam_gold_top_playtime_titles` (Gold Q5) listou os 20 jogos com maior tempo médio de jogo.

**Resposta:** Os títulos com maior tempo médio de jogo são: O primeiro colocado, **Letters From a Rainy Day -Oceans and Lace-**, apresentando uma média de 359665 minutos (aproximadamente 5994 horas). O segundo colocado, **爱人 Lover**, com 322983 minutos (aproximadamente 5383 horas). O terceiro colocado, **秘密舞会**, com 189436 minutos (aproximadamente 3157 horas). Um detalhe é que o único jogo gratuito do top 20 é o **Football Manager 2024**, que aparece na 20ª posição com 29196 minutos (aproximadamente 487 horas).

### 6.6 Pergunta 6

**Quais gêneros concentram as maiores médias de horas jogadas por usuário?**

A tabela `steam_gold_playtime_genero` (Gold Q6) agregou a quantidade média de horas jogadas por gênero (gêneros com 50+ jogos).

**Resposta:** As maiores médias de horas jogadas por usuário concentram-se em **Massively Multiplayer** (271 min), **RPG** (226 min) e **Sports** (222 min) - gêneros que naturalmente favorecem sessões longas e rejogabilidade. Vale destacar que os gêneros "Massively Multiplayer" (1127 jogos) e "Sports" (3485 jogos) possuem um número total de títulos significativamente menor em comparação a gêneros como "RPG" (14144 jogos), porém não foram removidos da análise por atenderem ao filtro mínimo de 50 jogos. Além disso, esses gêneros se destacam por terem uma quantidade média de avaliações superior à maioria, com "Massively Multiplayer" registrando uma média de 5930,66 avaliações por jogo, sendo a maior média de avaliações entre todos os gêneros.

### 6.7 Pergunta 7

**Como evoluiu o preço médio dos jogos ao longo dos anos?**

A tabela `steam_gold_evolucao_precos` (Gold Q7) agregou preço médio, mediana e desvio-padrão por ano de lançamento (jogos pagos, 2012–2026).

**Resposta:** O preço médio dos jogos pagos caiu de 12,08 dólares em 2013 para 8,17 dólares em 2018. Essa queda coincide com a explosão do volume de lançamentos no mesmo período, sugerindo que a popularização das ferramentas de desenvolvimento e a democratização da publicação trouxeram muitos jogos indie de baixo preço ao catálogo. A partir de 2018, o preço médio estabilizou-se na faixa de 8,17 a 9,70 dólares, com a mediana estando entre 4,99 e 5,99 dólares. O desvio padrão cresceu ao longo dos anos, indicando maior dispersão de preços, mostrando que a plataforma da steam tornou-se mais heterogênea.

### 6.8 Discussão dos resultados

**Síntese geral:**

A análise do catálogo da Steam revela um mercado em franca expansão, com o volume de lançamentos crescendo muito entre 2012 e 2024. Esse crescimento foi acompanhado por uma redução e posterior estabilização do preço médio (aproximadamente 9 dólares), refletindo a popularização de jogos indie acessíveis.

Quanto ao engajamento, os dados mostram uma relação robusta entre conquistas (achievements) e métricas de retenção: jogos com mais conquistas têm a quantia média de horas jogadas maior, mais recomendações e mais avaliações. Isso sugere que as conquistas funcionam como um indicador indireto de jogos maiores e mais produzidos, não uma causa linear isolada.

Os gêneros com maior quantidade de horas jogadas são Massively Multiplayer, RPG e Sports, confirmando a expectativa de que jogos com progressão contínua e multiplayer engajam por mais tempo.

## 7. Autoavaliação

Consegui responder as 7 perguntas, porém com algumas dificuldades durante o desenvolvimento do projeto, como por exemplo descobrindo que os dados do dataset estão incompletos para o ano de 2026. Também houve a identificação de gêneros vazios (`LENGTH = 0`) após a explosão do array, que geravam linhas inválidas na tabela `steam_generos_silver` e precisaram ser filtrados. Além disso, foi necessária a identificação e filtragem de aplicativos de software no catálogo da Steam (Audio Production, Video Production, Utilities, etc.), que exigiu análise dos gêneros para distinguir aplicativos de jogos e adicioná-los como filtro na camada Silver. Por fim, a identificação e remoção do gênero Free To Play, que nestes casos não estava sendo tratados como um gênero de jogo, e que gerava inconsistência nos resultados dos jogos pagos.

Como continuidade deste trabalho, pretende-se cruzar os dados com a base complementar de avaliações textuais para análises mais profundas sobre avaliações, implementar visualizações de dashboards a partir das tabelas Gold e analisar de forma aprofundada o boom da Steam a partir do ano de 2012/2013 que foi observado na pergunta 3.
