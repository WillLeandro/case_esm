# Segmentação de Clientes  Transportadora de Cargas Fracionadas

**Case Técnico**  


---

## Sumário

1. [Contexto do Problema](#1-contexto-do-problema)
2. [Arquitetura do Projeto](#2-arquitetura-do-projeto)
3. [Estrutura de Arquivos](#3-estrutura-de-arquivos)
4. [Etapa 1 - Captação e Tratamento dos Dados](#4-etapa-1--captação-e-tratamento-dos-dados)
5. [Etapa 2 - Análise Exploratória de Dados](#5-etapa-2--análise-exploratória-de-dados)
6. [Etapa 3 - Ciclo 1 de Clusterização](#6-etapa-3--ciclo-1-de-clusterização)
7. [Etapa 4 - Ciclo 2 de Clusterização](#7-etapa-4--ciclo-2-de-clusterização)
8. [Resultados Finais](#8-resultados-finais)
9. [Decisões Técnicas e Justificativas](#9-decisões-técnicas-e-justificativas)
10. [Limitações e Próximos Ciclos](#10-limitações-e-próximos-ciclos)
11. [Como Reproduzir](#11-como-reproduzir)

---

## 1. Contexto do Problema

A empresa opera no segmento de transporte de cargas fracionadas, modelo em que diferentes clientes compartilham o mesmo veículo em uma mesma rota. Nesse contexto, entender o perfil de uso de cada cliente é fundamental tanto para a gestão da capacidade operacional quanto para a definição de estratégias comerciais diferenciadas.

O objetivo deste projeto é fazer uma análise e segmentar a base de clientes com base em seu comportamento operacional,  frequência de uso, volume médio por envio, regularidade e perfil de carga, de modo a identificar grupos com características homogêneas que possam ser tratados de forma distinta pela área comercial e pela operação.

Os dados cobrem 26 dias de operação em janeiro de 2026, contemplando três fontes: cadastro de clientes, histórico de viagens com métricas de peso e volume, e classificação econômica por CNAE de cada cliente cadastrado.

---

## 2. Arquitetura do Projeto

```
Arquivos CSV (origem)
        │
        ▼
┌───────────────────────┐
│  01_tratamento_insert │   Leitura, validação, tratamento de tipos
│       .ipynb          │   e carga no banco PostgreSQL (Render)
└───────────────────────┘
        │
        ▼
┌─────────────────────────────────────┐
│         PostgreSQL - Render         │
│                                     │
│  fato_volumetria   (1.048.575 reg)  │
│  dim_cliente       (131.728 reg)    │
│  dim_cliente_cnae  (764.658 reg)    │
│  clientes_segmentados (resultado)   │
└─────────────────────────────────────┘
        │
        ▼
┌───────────────────────────┐
│  02_analise_clientes_     │   EDA + Ciclo 1 + Ciclo 2 de
│       final.ipynb         │   clusterização KMeans
└───────────────────────────┘
        │
        ▼
Tabela clientes_segmentados
(84.246 clientes com cluster atribuído)
```

**Por que PostgreSQL no Render?**  
A escolha por um banco de dados relacional hospedado permite que os dados tratados fiquem acessíveis de qualquer ambiente sem necessidade de reenvio de arquivos. O Render oferece instância gratuita de PostgreSQL com acesso via connection string, o que foi suficiente para o volume deste projeto. A separação entre a etapa de carga e a etapa de análise também garante que qualquer reexecução do notebook de análise parte sempre dos mesmos dados persistidos.

---

## 3. Estrutura de Arquivos

```
├── config/
│   └── database_config.yaml        # Credenciais do banco (não versionado)
├── data/
│   ├── cliente.csv                 # Cadastro de clientes (origem)
│   ├── volumetria.csv              # Histórico de viagens (origem)
│   ├── cliente_cnae.csv            # CNAEs por cliente (origem)
│   └── clientes_segmentados.csv    # Output: clientes com cluster
├── 01_tratamento_insert.ipynb      # Tratamento e carga no banco
├── 02_analise_clientes_final.ipynb # EDA e clusterização
└── README.md
```

> O arquivo `database_config.yaml` não deve ser versionado. Usado `.gitignore` para excluí-lo do repositório.

---

## 4. Etapa 1 - Captação e Tratamento dos Dados

**Notebook:** `01_tratamento_insert.ipynb`

### 4.1 Fontes recebidas

Os dados foram fornecidos em três arquivos CSV separados por ponto e vírgula:

| Arquivo | Registros | Colunas | Descrição |
|---------|-----------|---------|-----------|
| `cliente.csv` | 131.728 | 6 | Cadastro de clientes com UF, município e tipo |
| `volumetria.csv` | 1.048.575 | 10 | Registros de viagem com peso, volume e CTes por cliente |
| `cliente_cnae.csv` | 764.658 | 4 | CNAEs associados a cada cliente, com flag de CNAE principal |

### 4.2 Diagnóstico de qualidade

Antes de qualquer transformação, foram verificados:

- **Nulos:** nenhuma das três tabelas apresentou valores nulos nas colunas operacionais. A única exceção é o campo `cnae_descr`, que fica nulo para clientes sem CNAE cadastrado, comportamento esperado e tratado nas análises posteriores.
- **Duplicatas:** zero registros duplicados em todas as tabelas.
- **Valores categóricos:** os campos `cli_fisjur` (F/J), `tipopesocubico` (M/P) e `cne_cnae_principal` (0/1) apresentaram apenas os valores esperados, sem inconsistências.

### 4.3 Tratamentos realizados

A tabela `volumetria` foi a única que exigiu transformações de tipo:

| Coluna | Tipo original | Tipo final | Motivo |
|--------|--------------|------------|--------|
| `dt_viagem` | `object` (string) | `datetime64` | Necessário para cálculos temporais |
| `peso` | `object` | `float64` | Valores com vírgula decimal no CSV |
| `m3` | `object` | `float64` | Mesma causa |
| `perc_descm3` | `object` | `float64` | Mesma causa |

A conversão dos campos numéricos exigiu substituição de vírgula por ponto antes do cast, indicando que o arquivo foi gerado em ambiente com localização pt-BR.

Os nomes das colunas foram padronizados em todas as tabelas: minúsculas, sem espaços, remoção de caracteres BOM (`ï»¿`) presentes no início de alguns arquivos CSV.

### 4.4 Modelagem no banco

As três tabelas foram carregadas no PostgreSQL com tipagem explícita via SQLAlchemy:

- `dim_cliente` - dimensão de clientes com chave primária em `cli_codigo`
- `dim_cliente_cnae` - dimensão de CNAEs com referência a `cli_codigo`
- `fato_volumetria` - tabela fato com os registros de viagem

Após a carga, foram criados índice em `cli_codigo` na tabela fato para otimizar os JOINs realizados no notebook de análise.

---

## 5. Etapa 2 - Análise Exploratória de Dados

**Notebook:** `02_analise_clientes_final.ipynb` - Parte 1

A EDA foi conduzida sobre a visão consolidada das três tabelas, extraída via query com JOIN no banco. O dataset resultante contém 1.093.129 registros e 84.491 clientes únicos ativos no período.

### 5.1 Visão geral do período

| Métrica | Valor |
|---------|-------|
| Período | 01/01/2026 a 26/01/2026 |
| Viagens únicas | 18.289 |
| Conhecimentos de Transporte (CTes) | 1.499.313 |
| Clientes ativos | 84.491 |
| Peso total transportado | 77.700.598 kg |
| Volume total | 159.673 m³ |

### 5.2 Comportamento temporal

A operação é concentrada nos dias úteis, com queda expressiva nos finais de semana. O padrão intradiário revela dois picos de saída de carga: um no início da manhã e outro no final do dia, compatível com transportadoras que operam coletas programadas e entregas em janela.

### 5.3 Perfil da base de clientes

- **Tipo de pessoa:** a grande maioria da base é composta por Pessoas Jurídicas. Clientes PF existem, mas representam parcela menor e com volumes operacionais menores.
- **Distribuição geográfica:** SC, PR, RS e SP concentram a maior parte dos clientes ativos e das viagens, refletindo a área de atuação principal da transportadora.
- **Segmento econômico:** os CNAEs mais frequentes envolvem comércio varejista e atacadista de peças automotivas, confecções e equipamentos agrícolas, setores com alta demanda por transporte fracionado regular.

### 5.4 Métricas operacionais

As distribuições de peso, volume e CTes são fortemente assimétricas (skewed right). A mediana de peso por registro é de 10 kg, mas o máximo ultrapassa 16.000 kg. Essa assimetria é estrutural do negócio: a maioria das notas é de pequenos envios, mas alguns clientes movimentam volumes excepcionalmente grandes.

O campo `tipopesocubico` revelou que 88% dos registros são do tipo **P** (peso real > peso cubado), indicando predominância de cargas densas. Os 12% do tipo **M** (cubado predomina) correspondem a mercadorias leves e volumosas, para as quais o frete é calculado pelo volume, dado relevante para a área de precificação.

### 5.5 Concentração por cliente, Curva de Pareto

**5,7% dos clientes concentram 80% do peso total transportado.**

Essa concentração é característica do mercado B2B de carga fracionada: indústrias e atacadistas respondem pelo grosso do volume enquanto a base de pequenos clientes é pulverizada. A implicação direta para a segmentação é que qualquer abordagem baseada apenas em totais absolutos irá isolar esses clientes grandes em um grupo próprio sem produzir insights acionáveis sobre o restante da base.

### 5.6 Hipóteses levantadas

Com base na EDA, quatro hipóteses guiaram a construção do modelo de segmentação:

**H1 Concentração na cauda longa:** segmentação por totais absolutos separa gigantes do restante, sem gerar grupos úteis.

**H2 Sazonalidade semanal:** a operação segue ciclo semanal claro; a regularidade do cliente dentro desse ciclo é uma dimensão comportamental relevante.

**H3 Cubagem como proxy de tipo de mercadoria:** a proporção de cargas tipo M por cliente captura indiretamente o setor econômico e o perfil logístico da mercadoria.

**H4 Grupos comportamentais latentes:** a variabilidade observada em frequência, peso por envio e recência sugere subgrupos com padrões de uso distintos que não são capturados por segmentações simples.

---

## 6. Etapa 3 Ciclo 1 de Clusterização

**Notebook:** `02_analise_clientes_final.ipynb` — Parte 2

### 6.1 Abordagem

O primeiro ciclo utilizou KMeans com as features de totais agregados por cliente - `total_viagens`, `total_ctes`, `total_volumes`, `total_peso`, `total_m3` e `media_perc_desc` - normalizadas via `StandardScaler`. O objetivo desta etapa foi estabelecer uma linha de base e identificar os problemas antes de partir para uma abordagem mais refinada.

### 6.2 Avaliação

| k | Silhouette Score |
|---|-----------------|
| 2 | **0.979** |
| 3 | 0.621 |
| 4 | 0.587 |

O Silhouette de 0.979 com k=2 sinalizou imediatamente um problema. Scores próximos de 1 em dados de negócio quase sempre indicam que o algoritmo encontrou outliers extremos separados de toda a base, e não grupos comportamentais reais.

### 6.3 Resultado e diagnóstico

Com k=2:

| Cluster | Clientes | % da Base |
|---------|----------|-----------|
| 0 | 84.361 | 99,8% |
| 1 | 130 | 0,2% |

Praticamente toda a base ficou em um único grupo. Dois problemas estruturais causaram esse resultado:

**Problema 1 - Distribuição assimétrica não tratada**

O `StandardScaler` centraliza e escala os dados, mas não corrige a assimetria. Com mediana de `total_viagens` em 3 e máximo em 1.703, os outliers continuam dominando o espaço vetorial. Como o KMeans minimiza distâncias euclidianas, os centroides são puxados em direção aos valores extremos, e o restante da base fica agrupado em torno de um único centroide distante.

**Problema 2 - Features apenas de volume absoluto**

Usar totais absolutos faz com que a clusterização separe clientes pelo tamanho, não pelo comportamento. Dois clientes com volumes similares podem ter perfis operacionais completamente diferentes: um com muitas viagens pequenas e diárias, outro com poucas viagens mas cargas esporádicas e pesadas.

---

## 7. Etapa 4 - Ciclo 2 de Clusterização

**Notebook:** `02_analise_clientes_final.ipynb` - Parte 3

### 7.1 Decisões de melhoria

Cada problema identificado no Ciclo 1 foi endereçado com uma decisão técnica específica:

#### Recorte geográfico

SC, PR, RS e SP concentram 93% das viagens totais. Manter clientes de estados com densidade operacional muito baixa introduziria ruído: o comportamento de um cliente em um estado com poucas rotas reflete limitação de malha, não perfil de uso. O recorte garante que os clientes comparados compartilham o mesmo contexto logístico.

| Impacto do filtro | Antes | Depois |
|-------------------|-------|--------|
| Clientes únicos | 84.491 | 84.246 |
| Viagens únicas | 18.289 | 18.281 |
| Registros | 1.093.129 | 1.088.403 |

O impacto foi mínimo em número de registros (0,4%), confirmando que os estados excluídos tinham presença marginal na operação.

#### Engenharia de features comportamentais

Foram criadas sete variáveis derivadas que capturam o *padrão de uso* do cliente independentemente do seu volume absoluto:

| Feature | Descrição | Captura |
|---------|-----------|---------|
| `ctes_por_viagem` | CTes / viagens | Consolidação: cliente que manda muitas notas por visita vs. um por vez |
| `peso_por_cte` | Peso total / CTes | Tamanho médio de cada envio |
| `volumes_por_cte` | Volumes / CTes | Fragmentação física das cargas |
| `dias_desde_ultimo` | Dias desde o último envio | Recência |
| `spread_dias` | Dias entre primeiro e último envio | Janela de atividade no período |
| `freq_diaria` | Viagens / (spread + 1) | Regularidade normalizada pelo período de atividade |
| `tipo_M_pct` | % de registros tipo M | Perfil do tipo de mercadoria transportada |

#### Transformação logarítmica

`log1p(x) = log(1 + x)` foi aplicada nas features de escala contínua antes do `StandardScaler`. A função é válida para x=0 e inversível via `expm1`. O efeito é comprimir a cauda direita das distribuições, fazendo o KMeans operar sobre ordens de grandeza em vez de valores absolutos. Features de percentual (`tipo_M_pct`, `media_perc_desc`) foram mantidas na escala original.

#### Tratamento de outliers extremos

Os 422 clientes acima do percentil 99,5 em peso total foram separados antes do `fit`. Isso preserva a qualidade dos centroides, que são calculados sobre a distribuição principal da base. Após o fit, esses clientes foram classificados com `predict` usando o modelo treinado - eles recebem um cluster, mas não influenciam sua definição.

### 7.2 Avaliação com três métricas

| k | Inércia | Silhouette | Davies-Bouldin |
|---|---------|-----------|----------------|
| 2 | 601.067 | 0.239 | 1.718 |
| **3** | **512.683** | **0.265** | **1.467** |
| 4 | 446.341 | 0.233 | 1.439 |
| 5 | 394.124 | 0.247 | 1.273 |

**Escolha de k=3:** melhor Silhouette entre os candidatos naturais do cotovelo (k=2 e k=3), com Davies-Bouldin ainda aceitável. O ganho de inércia ao passar de k=3 para k=4 não se traduz em melhora equivalente de separação, e clusters adicionais tendem a ser fragmentações de grupos maiores sem diferença comportamental clara. Do ponto de vista de negócio, três segmentos são suficientes para orientar ações distintas sem criar complexidade operacional.

---

## 8. Resultados Finais

### 8.1 Distribuição dos clusters

| Cluster | Nome | Clientes | % da Base |
|---------|------|----------|-----------|
| 0 | Clientes Regulares de Médio Porte | 31.399 | 37,3% |
| 1 | Clientes Estratégicos de Alto Volume | 5.214 | 6,2% |
| 2 | Clientes Esporádicos de Baixo Volume | 47.633 | 56,5% |

### 8.2 Perfil mediano por cluster

| Feature | Cluster 0 | Cluster 1 | Cluster 2 |
|---------|-----------|-----------|-----------|
| Viagens | 9 | 3 | 2 |
| Peso total (kg) | 404 | 2 | 21 |
| CTes por viagem | 1,0 | 1,0 | — |
| Peso por CTe (kg) | 42,5 | 0,72 | — |

**Cluster 0 - Clientes Regulares de Médio Porte (37,3%)**  
Frequência e peso intermediários, presença consistente ao longo do período. São o grupo com maior previsibilidade operacional. Representam a espinha dorsal da receita recorrente da transportadora. CNAEs predominantes: comércio varejista e atacadista de peças automotivas e confecções.

**Cluster 1 - Clientes Estratégicos de Alto Volume (6,2%)**  
Frequência diária elevada, peso total alto, muitos CTes por viagem. Apesar de pequeno em número, este grupo concentra o grosso do volume transportado. Exigem atenção comercial dedicada e SLA diferenciado.

**Cluster 2 - Clientes Esporádicos de Baixo Volume (56,5%)**  
Maior parcela da base em número, mas com baixo peso por envio e recência alta - boa parte enviou no início do período e não retornou. Representam potencial de crescimento via programas de fidelização ou reativação.

### 8.3 Output salvo no banco

A tabela `clientes_segmentados` foi criada no PostgreSQL com `cli_codigo`, `cluster` e todas as features utilizadas no modelo, permitindo que a segmentação seja consumida por dashboards ou outras análises.

---

## 9. Decisões Técnicas e Justificativas

| Decisão | Alternativa considerada | Justificativa da escolha |
|---------|------------------------|--------------------------|
| KMeans | DBSCAN, Hierárquico | KMeans é interpretável, escalável e produz clusters de tamanho comparável. DBSCAN foi descartado por sensibilidade a epsilon em dados de alta dimensão; hierárquico por custo computacional com 84k clientes |
| log1p antes do StandardScaler | Apenas StandardScaler (Ciclo 1) | StandardScaler não corrige assimetria. log1p comprime a cauda e permite que o algoritmo separe por padrão de uso e não por magnitude absoluta |
| Separação de outliers antes do fit | Remover outliers / manter todos | Remover perde informação; manter no fit distorce os centroides. Separar antes e classificar depois é o equilíbrio correto |
| Recorte para SC/PR/RS/SP | Usar toda a base | Clientes em estados com cobertura marginal têm comportamento incomparável com os demais. O recorte reduz ruído sem perda de representatividade (0,4% dos registros excluídos) |
| k=3 | k=2 ou k=4 | Melhor Silhouette no intervalo relevante, e três segmentos têm interpretabilidade direta para uso comercial |
| Mediana como estatística de perfil | Média | Com distribuições assimétricas, a mediana é mais representativa do comportamento típico do cluster |
| Três métricas de avaliação (Elbow + Silhouette + Davies-Bouldin) | Apenas Silhouette | O Ciclo 1 demonstrou que Silhouette alto pode ser enganoso. Davies-Bouldin penaliza sobreposição e complementa a análise |

---

## 10. Limitações e Próximos Ciclos

### Limitações do ciclo atual

**Período curto:** 26 dias de dados limitam a resolução das métricas de recência e regularidade. Um cliente que enviou apenas uma vez pode ser esporádico ou pode ter iniciado o relacionamento no final do período - não é possível distinguir com uma janela tão pequena.

**Ausência do valor do frete:** sem informação de receita por viagem, a segmentação é inteiramente baseada em volume operacional. Um cliente de alto volume pode ter margens baixas, e um cliente de baixo volume pode ser altamente rentável.

### Próximos ciclos sugeridos

**Ciclo 3 - Incorporar valor do frete**  
Com dados de receita por viagem, seria possível construir uma segmentação por rentabilidade. Features como receita por kg, margem por viagem e ticket médio por CTe permitiriam identificar clientes estratégicos por valor financeiro, não apenas por volume.

**Ciclo 4 - Ampliar janela temporal**  
Com três meses ou mais de dados, métricas de sazonalidade mensal, churn potencial (clientes que pararam de enviar) e variação de frequência entre períodos tornam-se calculáveis e mais robustas.

**Ciclo 5 - Modelo de propensão ao churn**  
O Cluster 2 (esporádicos com alta recência) é o candidato natural para um modelo de risco de abandono. Com dados históricos de clientes que de fato deixaram de operar, seria possível treinar um classificador supervisionado.

**Ciclo 6 - Segmentação hierárquica**  
Dentro do Cluster 1 (estratégicos), uma segunda rodada de clusterização pode revelar subtipos com necessidades operacionais diferentes - por exemplo, indústrias com fluxo diário versus distribuidores com picos semanais.

---

## 11. Como Reproduzir

### Pré-requisitos

```
Python 3.10+
pandas
numpy
scikit-learn
sqlalchemy
psycopg2-binary
matplotlib
seaborn
pyyaml
```

Instalar dependências:

```bash
pip install pandas numpy scikit-learn sqlalchemy psycopg2-binary matplotlib seaborn pyyaml
```

### Configuração do banco

Criar o arquivo `config/database_config.yaml`:

```yaml
host: <host_render>
port: 5432
database: <nome_do_banco>
user: <usuario>
password: <senha>
```

### Execução

```bash
# 1. Tratamento e carga no banco
jupyter nbconvert --to notebook --execute 01_tratamento_insert.ipynb

# 2. EDA e clusterização
jupyter nbconvert --to notebook --execute 02_analise_clientes_final.ipynb
```

Os notebooks também podem ser executados célula a célula via Jupyter Lab ou VS Code.

O notebook `02_analise_clientes_final.ipynb` tem uma variável `conect_db` no início. Quando `True`, conecta ao banco e executa a query. Quando `False`, carrega os dados de um CSV local previamente salvo - útil para reexecuções sem depender da conexão.

---

