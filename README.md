# Bike Store | Arquitetura Medallion com Databricks

## Sobre o Projeto

Este projeto implementa um pipeline de Engenharia de Dados para a base **Bike Store**, utilizando **Databricks, PySpark, Spark SQL, Delta Lake, Unity Catalog e Azure Data Lake Storage (ADLS)**. Os dados CSV são ingeridos na camada Bronze, refinados e integrados na Silver e transformados em datasets de negócio na Gold.

## Por que este trabalho foi realizado

Os dados da Bike Store estão distribuídos em diferentes arquivos transacionais, cada um representando uma parte da operação: clientes, pedidos, itens vendidos, produtos, marcas, categorias, lojas, funcionários e estoques. Embora esses arquivos sejam suficientes para registrar as operações, isoladamente eles não entregam uma visão integrada do negócio e exigem relacionamentos, tratamentos e regras para responder perguntas gerenciais.

O trabalho foi realizado para construir uma **base de dados analítica, organizada e rastreável**, separando os dados brutos das informações tratadas e dos datasets destinados ao consumo de negócio. A arquitetura Medallion foi adotada para que cada etapa do pipeline tenha uma responsabilidade clara e para evitar que análises dependam diretamente dos arquivos CSV de origem.

Do ponto de vista de negócio, o objetivo é permitir que os dados apoiem atividades como **análise de vendas, acompanhamento de pedidos pendentes, análise regional e gestão de estoque**. Do ponto de vista de Engenharia de Dados, o projeto demonstra como transformar múltiplas fontes transacionais em uma estrutura Lakehouse utilizando Databricks e Delta Lake.

## O que foi feito

O projeto implementou um fluxo completo de dados da origem até a camada de consumo:

1. **Ingestão dos dados:** os nove arquivos CSV foram carregados no Databricks e persistidos na camada Bronze em formato Delta, mantendo uma representação próxima dos dados de origem.
2. **Integração e tratamento:** na Silver, as tabelas relacionadas foram combinadas para formar entidades mais adequadas à análise, como produtos enriquecidos, pedidos e clientes.
3. **Aplicação de regras de negócio:** foram criadas regras como tradução do status dos pedidos, cálculo do valor líquido da venda considerando quantidade e desconto, consolidação do estoque dos produtos e seleção de informações de contato utilizáveis.
4. **Criação de datasets de negócio:** na Gold, os dados foram preparados para necessidades específicas, incluindo análise de vendas das lojas de New York e acompanhamento de pedidos pendentes.
5. **Persistência e governança:** os dados foram armazenados em Delta Lake no ADLS e disponibilizados por meio do catálogo do Databricks, mantendo separação entre armazenamento, processamento e consumo.
6. **Documentação:** foram registrados o modelo de dados, a linhagem entre as camadas, as regras de negócio e o dicionário de dados de Bronze, Silver e Gold.

O resultado é o seguinte fluxo:

```text
Arquivos CSV transacionais
          ↓
      BRONZE
Dados brutos e rastreáveis
          ↓
       SILVER
Dados tratados, integrados e enriquecidos
          ↓
        GOLD
Datasets orientados às necessidades de negócio
          ↓
   BI / Analytics / Operação
```

## Cenário de Negócio

A **Bike Store** representa uma operação varejista de bicicletas organizada em torno de clientes, pedidos, produtos, lojas, funcionários e estoque. Na origem, essas informações estão distribuídas em nove arquivos CSV relacionados entre si. Essa estrutura atende ao registro transacional, mas exige integração e regras de negócio antes de ser utilizada em análises ou processos operacionais.

O pipeline foi construído para transformar esses dados operacionais em informações que permitam responder perguntas como:

- Qual é o valor efetivamente vendido depois dos descontos?
- Quanto foi vendido e entregue por dia nas lojas localizadas no estado de New York (NY)?
- Quais pedidos ainda estão pendentes?
- Quantos itens estão associados aos pedidos pendentes?
- Quais clientes de pedidos pendentes possuem telefone e e-mail para contato?
- Qual é o estoque total disponível de cada produto considerando todas as lojas?
- A qual marca e categoria cada produto pertence?

A arquitetura Medallion separa essas responsabilidades. A **Bronze** preserva os dados próximos da origem; a **Silver** integra as entidades, melhora a qualidade e cria atributos de negócio; e a **Gold** materializa conjuntos de dados voltados a necessidades específicas de análise e operação.

### Fluxo do cenário de negócio

```text
Clientes ──> Pedidos ──> Itens do pedido ──> Produtos
               │                 │              ├──> Marcas
               │                 │              └──> Categorias
               │                 │
               │                 └──> Valor líquido da venda
               │
               ├──> Loja ──> Localização da venda
               └──> Funcionário responsável

Produtos + Estoques por loja ──> Estoque total por produto

Silver Orders ──> Vendas entregues em NY ──> Gold Sales NY
       │
       └──> Pedidos pendentes + clientes contactáveis ──> Gold Orders Pending
```

### Necessidades de negócio atendidas

**Análise comercial:** mensurar o valor líquido vendido, já considerando quantidade e desconto aplicados em cada item do pedido.

**Análise regional:** acompanhar o valor diário das vendas entregues pelas lojas do estado de NY. No projeto, `state` vem da tabela de lojas; portanto, o recorte geográfico representa o **estado da loja responsável pelo pedido**, e não o endereço do cliente.

**Acompanhamento operacional:** identificar pedidos com status `Pending`, consolidar sua quantidade de itens e associá-los aos dados de contato disponíveis do cliente.

**Gestão de produtos e estoque:** enriquecer os produtos com marca, categoria e estoque total somado entre as lojas.

## Arquitetura da Solução

```text
CSV (origem / Volume)
        ↓
Bronze - dados brutos em Delta
        ↓
Silver - dados integrados e tratados
        ↓
Gold - datasets orientados ao negócio
        ↓
BI / Analytics / Alertas
```

O armazenamento físico é realizado no ADLS via `abfss://.../bikestore/{bronze|silver|gold}/`, enquanto tabelas externas são registradas no catálogo `bikestore.logistics`.

## Tecnologias Utilizadas

- Databricks
- Apache Spark / PySpark
- Spark SQL
- Delta Lake
- Unity Catalog
- Databricks Volumes
- Azure Data Lake Storage (ADLS / ABFS)
- CSV como fonte de dados
- Arquitetura Medallion

## Por que utilizar Databricks e Cloud neste projeto

A utilização de **Databricks em conjunto com serviços de cloud** torna a solução mais adequada para um cenário em que os dados podem crescer em volume, variedade e frequência de atualização. Neste projeto, o Databricks concentra processamento, transformação e governança, enquanto o **Azure Data Lake Storage (ADLS)** fornece a camada de armazenamento dos dados.

### Benefícios do Databricks

| Benefício | Como se aplica ao projeto | Valor gerado |
|---|---|---|
| **Processamento distribuído com Apache Spark** | As transformações Bronze → Silver → Gold são executadas com PySpark e Spark SQL. | Permite que a mesma abordagem seja utilizada quando o volume de dados aumentar. |
| **Lakehouse** | O projeto combina arquivos armazenados no Data Lake com tabelas e recursos analíticos do Databricks. | Aproxima a flexibilidade de um Data Lake das funcionalidades normalmente esperadas de uma plataforma analítica. |
| **Delta Lake** | Bronze, Silver e Gold são persistidas em formato Delta. | Adiciona recursos transacionais e facilita a manutenção de dados estruturados no Data Lake. |
| **Arquitetura Medallion** | Os dados evoluem de Bronze para Silver e Gold. | Separa ingestão, tratamento e consumo, facilitando manutenção, rastreabilidade e reutilização. |
| **PySpark e Spark SQL no mesmo ambiente** | O pipeline utiliza APIs de DataFrame e consultas SQL. | Permite que engenheiros de dados e usuários com perfil SQL trabalhem sobre a mesma plataforma. |
| **Unity Catalog** | As tabelas do projeto são disponibilizadas no catálogo `bikestore.logistics`. | Centraliza a organização e cria uma base para governança, controle de acesso e descoberta dos dados. |
| **Escalabilidade** | O processamento não fica limitado a uma execução local em uma única máquina. | A arquitetura pode acompanhar o crescimento da quantidade de clientes, pedidos, produtos e histórico de vendas. |
| **Integração analítica** | A Gold produz datasets como `gold_sales_ny` e `gold_orders_pending`. | Facilita o consumo posterior por BI, análises e processos operacionais. |

### Benefícios da utilização de Cloud

Neste projeto, a cloud permite separar **armazenamento** e **processamento**. Os dados são persistidos no ADLS, enquanto o Databricks é utilizado para processá-los. Essa separação é importante porque o armazenamento não precisa ficar preso à máquina que executa o processamento.

Os principais benefícios são:

- **Escalabilidade:** a infraestrutura pode ser ampliada conforme o crescimento dos dados e das cargas de processamento.
- **Elasticidade:** recursos computacionais podem ser ajustados de acordo com a necessidade do workload, evitando depender de uma capacidade física fixa.
- **Armazenamento centralizado:** Bronze, Silver e Gold ficam armazenadas no Data Lake e podem ser reutilizadas por diferentes processos e consumidores autorizados.
- **Maior disponibilidade dos dados:** os dados deixam de depender de arquivos mantidos exclusivamente em uma máquina local.
- **Integração entre serviços:** o Data Lake pode servir como base para Databricks, ferramentas de BI e outros serviços analíticos.
- **Governança e segurança:** a combinação de armazenamento cloud e catálogo cria uma base para políticas de acesso, organização e rastreabilidade.
- **Automação:** a arquitetura facilita a evolução de notebooks executados manualmente para pipelines agendados, monitorados e integrados a práticas de CI/CD.
- **Crescimento sem redesenho completo:** caso a empresa passe a receber mais pedidos, clientes, lojas ou novas fontes de dados, a arquitetura pode evoluir preservando o padrão Bronze → Silver → Gold.
- **Otimização e controle de custos:** a cloud permite acompanhar e controlar os gastos de armazenamento e processamento de acordo com o consumo dos recursos. Diferentemente de uma infraestrutura dimensionada permanentemente para atender aos picos de demanda, recursos computacionais podem ser iniciados, dimensionados e encerrados conforme a necessidade. Entretanto, essa flexibilidade exige práticas de FinOps, como monitoramento de custos, definição de budgets e alertas, desligamento de clusters ociosos, escolha adequada do tamanho dos recursos, otimização dos jobs e adoção de políticas de retenção e ciclo de vida dos dados. Dessa forma, busca-se equilibrar desempenho, disponibilidade e custo, evitando desperdício de recursos financeiros.

### Benefício para o cenário de negócio

No contexto da Bike Store, esses recursos permitem transformar dados operacionais dispersos em uma base centralizada e preparada para análise. Em vez de cada área trabalhar diretamente com arquivos CSV e reconstruir relacionamentos e cálculos, o pipeline passa a entregar dados tratados e orientados ao negócio.

Por exemplo:

```text
Arquivos operacionais
        ↓
ADLS + Bronze
Centralização e preservação dos dados de origem
        ↓
Databricks + Silver
Limpeza, integração, qualidade e regras de negócio
        ↓
Databricks + Gold
Métricas e datasets preparados para consumo
        ↓
BI / Analytics / Operação
```

Isso permite que perguntas como **“quanto foi efetivamente vendido?”**, **“quais pedidos estão pendentes?”**, **“qual é o estoque disponível?”** e **“qual o volume de vendas associado às lojas de NY?”** sejam respondidas a partir de conjuntos de dados preparados para essa finalidade, sem exigir que cada consumidor refaça todo o tratamento dos dados brutos.

## Fonte de Dados e Modelo Relacional

A origem é composta por nove arquivos CSV: `brands`, `categories`, `customers`, `order_items`, `orders`, `products`, `staffs`, `stocks` e `stores`. O modelo relaciona clientes, pedidos, itens, produtos, marcas, categorias, lojas, funcionários e estoque.

Principais relacionamentos: `customers.customer_id → orders.customer_id`; `orders.order_id → order_items.order_id`; `products.product_id → order_items.product_id`; `brands.brand_id → products.brand_id`; `categories.category_id → products.category_id`; `stores.store_id → orders.store_id/staffs.store_id/stocks.store_id`; e `staffs.manager_id → staffs.staff_id`.

## Camada Bronze - Raw Data

A Bronze lê os CSVs com `header=True` e `inferSchema=True` e grava cada entidade em **Delta** com `mode("overwrite")` e `mergeSchema=true`. Nesta etapa não há transformação de negócio: a finalidade é preservar uma representação próxima da origem para rastreabilidade e reprocessamento.

### `bronze_brands`

| Coluna | Tipo | Chave | Descrição |
|---|---|---|---|
| `brand_id` | INT | PK | Identificador único da marca. |
| `brand_name` | STRING | - | Nome da marca. |

### `bronze_categories`

| Coluna | Tipo | Chave | Descrição |
|---|---|---|---|
| `category_id` | INT | PK | Identificador único da categoria. |
| `category_name` | STRING | - | Nome da categoria do produto. |

### `bronze_customers`

| Coluna | Tipo | Chave | Descrição |
|---|---|---|---|
| `customer_id` | INT | PK | Identificador único do cliente. |
| `first_name` | STRING | - | Primeiro nome do cliente. |
| `last_name` | STRING | - | Sobrenome do cliente. |
| `phone` | STRING | - | Telefone do cliente; pode ser nulo na origem. |
| `email` | STRING | - | E-mail do cliente. |
| `street` | STRING | - | Logradouro do endereço. |
| `city` | STRING | - | Cidade do cliente. |
| `state` | STRING | - | Sigla do estado. |
| `zip_code` | INT | - | Código postal. |

### `bronze_order_items`

| Coluna | Tipo | Chave | Descrição |
|---|---|---|---|
| `order_id` | INT | PK/FK | Pedido ao qual o item pertence. |
| `item_id` | INT | PK | Sequência/identificador do item dentro do pedido. |
| `product_id` | INT | FK | Produto vendido. |
| `quantity` | INT | - | Quantidade vendida. |
| `list_price` | DOUBLE | - | Preço unitário de lista no pedido. |
| `discount` | DOUBLE | - | Percentual de desconto em formato decimal. |

### `bronze_orders`

| Coluna | Tipo | Chave | Descrição |
|---|---|---|---|
| `order_id` | INT | PK | Identificador único do pedido. |
| `customer_id` | INT | FK | Cliente associado ao pedido. |
| `order_status` | INT | - | Código do status do pedido (1 a 4). |
| `order_date` | DATE | - | Data de criação do pedido. |
| `required_date` | DATE | - | Data requerida para atendimento. |
| `shipped_date` | DATE | - | Data de envio; pode ser nula. |
| `store_id` | INT | FK | Loja responsável pelo pedido. |
| `staff_id` | INT | FK | Funcionário responsável pelo pedido. |

### `bronze_products`

| Coluna | Tipo | Chave | Descrição |
|---|---|---|---|
| `product_id` | INT | PK | Identificador único do produto. |
| `product_name` | STRING | - | Nome do produto. |
| `brand_id` | INT | FK | Marca do produto. |
| `category_id` | INT | FK | Categoria do produto. |
| `model_year` | INT | - | Ano/modelo do produto. |
| `list_price` | DOUBLE | - | Preço de lista do produto. |

### `bronze_staffs`

| Coluna | Tipo | Chave | Descrição |
|---|---|---|---|
| `staff_id` | INT | PK | Identificador único do funcionário. |
| `first_name` | STRING | - | Primeiro nome do funcionário. |
| `last_name` | STRING | - | Sobrenome do funcionário. |
| `email` | STRING | - | E-mail corporativo. |
| `phone` | STRING | - | Telefone. |
| `active` | INT | - | Indicador de funcionário ativo. |
| `store_id` | INT | FK | Loja à qual o funcionário pertence. |
| `manager_id` | INT | FK | Funcionário gestor; autorrelacionamento e pode ser nulo. |

### `bronze_stocks`

| Coluna | Tipo | Chave | Descrição |
|---|---|---|---|
| `store_id` | INT | PK/FK | Loja onde o estoque está localizado. |
| `product_id` | INT | PK/FK | Produto armazenado. |
| `quantity` | INT | - | Quantidade disponível na loja. |

### `bronze_stores`

| Coluna | Tipo | Chave | Descrição |
|---|---|---|---|
| `store_id` | INT | PK | Identificador único da loja. |
| `store_name` | STRING | - | Nome da loja. |
| `phone` | STRING | - | Telefone da loja. |
| `email` | STRING | - | E-mail da loja. |
| `street` | STRING | - | Logradouro da loja. |
| `city` | STRING | - | Cidade da loja. |
| `state` | STRING | - | Sigla do estado. |
| `zip_code` | INT | - | Código postal da loja. |

## Camada Silver - Trusted Data

A Silver lê os arquivos Delta da Bronze por meio de views temporárias, integra entidades relacionadas e aplica regras de qualidade e negócio. Os resultados são novamente persistidos em Delta.

### `silver_product`

Consolida produto, marca, categoria e estoque total disponível entre todas as lojas.

| Coluna | Tipo | Origem | Transformação / Regra | Descrição |
|---|---|---|---|---|
| `product_id` | INT | `products.product_id` | Direto | Identificador do produto. |
| `product_name` | STRING | `products.product_name` | Direto | Nome do produto. |
| `brand_name` | STRING | `brands.brand_name` | LEFT JOIN por brand_id | Nome da marca. |
| `category_name` | STRING | `categories.category_name` | LEFT JOIN por category_id | Nome da categoria. |
| `model_year` | INT | `products.model_year` | Direto | Ano/modelo. |
| `list_price` | DOUBLE | `products.list_price` | Direto | Preço de lista. |
| `total_stock` | BIGINT | `stocks.quantity` | SUM(quantity) por product_id | Estoque total do produto somado entre todas as lojas. |

### `silver_orders`

Consolida pedido, loja, funcionário e itens, traduz o status e calcula o valor líquido de cada item.

| Coluna | Tipo | Origem | Transformação / Regra | Descrição |
|---|---|---|---|---|
| `order_id` | INT | `orders.order_id` | Direto | Identificador do pedido. |
| `customer_id` | INT | `orders.customer_id` | Direto | Identificador do cliente. |
| `status` | STRING | `orders.order_status` | CASE 1=Pending, 2=Processing, 3=Shipped, 4=Delivered; demais=Unknown | Descrição textual do status. |
| `order_status` | INT | `orders.order_status` | Direto | Código original do status. |
| `order_date` | DATE | `orders.order_date` | Direto | Data do pedido. |
| `required_date` | DATE | `orders.required_date` | Direto | Data requerida. |
| `shipped_date` | DATE | `orders.shipped_date` | Direto | Data de envio. |
| `store_name` | STRING | `stores.store_name` | LEFT JOIN por store_id | Nome da loja. |
| `state` | STRING | `stores.state` | LEFT JOIN por store_id | Estado da loja. |
| `city` | STRING | `stores.city` | LEFT JOIN por store_id | Cidade da loja. |
| `first_name_staff` | STRING | `staffs.first_name` | LEFT JOIN por staff_id | Primeiro nome do vendedor/funcionário. |
| `active_staff` | INT | `staffs.active` | LEFT JOIN por staff_id | Indicador de funcionário ativo. |
| `email` | STRING | `staffs.email` | LEFT JOIN por staff_id | E-mail do funcionário. |
| `product_id` | INT | `order_items.product_id` | LEFT JOIN por order_id | Produto do item do pedido. |
| `quantity` | INT | `order_items.quantity` | Direto após join | Quantidade do item. |
| `total_sale` | DOUBLE | `order_items` | ROUND((list_price * quantity) * (1-discount), 2) | Valor líquido do item após desconto. |
| `list_price` | DOUBLE | `order_items.list_price` | Direto | Preço unitário de lista. |
| `discount` | DOUBLE | `order_items.discount` | Direto | Desconto aplicado. |

### `silver_customer`

Cria uma base de clientes aptos a contato, mantendo somente registros com telefone e e-mail preenchidos.

| Coluna | Tipo | Origem | Transformação / Regra | Descrição |
|---|---|---|---|---|
| `customer_id` | INT | `customers.customer_id` | Direto + filtro | Identificador do cliente. |
| `first_name` | STRING | `customers.first_name` | Direto + filtro | Primeiro nome. |
| `last_name` | STRING | `customers.last_name` | Direto + filtro | Sobrenome. |
| `phone` | STRING | `customers.phone` | Mantém somente valores não nulos e diferentes de texto NULL | Telefone válido para contato. |
| `email` | STRING | `customers.email` | Mantém somente valores não nulos e diferentes de texto NULL | E-mail válido para contato. |
| `street` | STRING | `customers.street` | Direto | Logradouro. |
| `city` | STRING | `customers.city` | Direto | Cidade. |
| `state` | STRING | `customers.state` | Direto | Estado. |
| `zip_code` | INT | `customers.zip_code` | Direto | Código postal. |

### Regras de negócio da Silver

**Valor líquido da venda:** `ROUND((list_price * quantity) * (1 - discount), 2)`.

**Mapeamento de status:** 1 = Pending; 2 = Processing; 3 = Shipped; 4 = Delivered; demais valores = Unknown.

**Qualidade de clientes:** `phone` e `email` devem ser diferentes de `NULL` real e também dos textos `NULL` e `NULL `.

## Camada Gold - Business Data

A Gold utiliza as entidades confiáveis da Silver para produzir datasets diretamente relacionados a necessidades de negócio.

### `gold_sales_ny`

KPI diário de vendas entregues no estado de Nova York (NY).

| Coluna | Tipo | Origem | Transformação / Regra | Descrição |
|---|---|---|---|---|
| `shipped_date` | DATE | `silver_orders.shipped_date` | Filtro state=NY, status=Delivered e shipped_date não nula; agrupamento por data | Data de envio usada como granularidade diária. |
| `total_sale` | DOUBLE | `silver_orders.total_sale` | ROUND(SUM(total_sale),2) | Total de vendas entregues por dia em NY. |

### `gold_orders_pending`

Dataset para acompanhamento/alerta de pedidos pendentes, incluindo quantidade de itens e canais de contato do cliente.

| Coluna | Tipo | Origem | Transformação / Regra | Descrição |
|---|---|---|---|---|
| `customer_id` | INT | `silver_orders.customer_id` | Pedidos com status Pending | Cliente do pedido pendente. |
| `order_date` | DATE | `silver_orders.order_date` | Agrupamento | Data do pedido. |
| `quantity` | BIGINT | `silver_orders.quantity` | SUM(quantity) | Quantidade total de itens pendentes no agrupamento. |
| `store_name` | STRING | `silver_orders.store_name` | Agrupamento | Loja do pedido. |
| `first_name_customer` | STRING | `silver_customer.first_name` | LEFT JOIN por customer_id | Primeiro nome do cliente. |
| `email` | STRING | `silver_customer.email` | Filtro IS NOT NULL | E-mail para contato. |
| `phone` | STRING | `silver_customer.phone` | Filtro IS NOT NULL | Telefone para contato. |

## Métricas e Indicadores de Negócio

O projeto implementa métricas em diferentes níveis do pipeline. Algumas são métricas intermediárias criadas na Silver para reutilização; outras são agregações da Gold voltadas diretamente ao consumo de negócio.

### 1. `total_sale` - Valor líquido do item vendido

**O que retorna:** o valor monetário efetivamente associado a cada item do pedido depois de considerar a quantidade comprada e o desconto aplicado. A granularidade é **item do pedido**.

**Para que serve:** fornece uma base monetária consistente para análises de vendas. Em vez de utilizar somente o preço de lista, a métrica considera quantas unidades foram vendidas e o desconto concedido. Ela também é a base utilizada posteriormente pela Gold `sales_ny`.

**Com o que é feita:** utiliza três campos da Bronze `order_items`:

- `list_price`: preço unitário de lista registrado no item do pedido;
- `quantity`: quantidade de unidades do item;
- `discount`: desconto em formato decimal.

**Fórmula implementada:**

```text
total_sale = ROUND((list_price * quantity) * (1 - discount), 2)
```

**Exemplo:** para 2 unidades com preço de 1.000,00 e desconto de 10% (`0.10`):

```text
(1.000,00 * 2) * (1 - 0,10) = 1.800,00
```

A métrica retorna **1.800,00** para esse item do pedido.

> Observação: no código do projeto, `total_sale` representa o valor líquido comercial calculado a partir de preço, quantidade e desconto. Não há campos de impostos, frete, devoluções ou custo no cálculo; portanto, a métrica não deve ser interpretada como lucro ou margem.

### 2. `total_stock` - Estoque total por produto

**O que retorna:** a soma das unidades disponíveis de cada produto considerando os registros de estoque de todas as lojas. A granularidade é **produto**.

**Para que serve:** oferece uma visão consolidada da disponibilidade do produto na rede, sem exigir que o consumidor analítico some manualmente o estoque loja a loja.

**Com o que é feita:** utiliza `stocks.product_id` para identificar o produto e `stocks.quantity` como quantidade disponível em cada loja.

**Fórmula implementada:**

```text
total_stock = SUM(stocks.quantity) GROUP BY product_id
```

**Interpretação:** se um produto possui 5 unidades na Loja A, 8 na Loja B e 2 na Loja C, `total_stock` retorna **15 unidades**.

> A métrica representa estoque consolidado da rede. Ela não informa, isoladamente, em qual loja as unidades estão disponíveis.

### 3. `gold_sales_ny.total_sale` - Vendas diárias entregues em NY

**O que retorna:** o valor total dos itens de pedidos **entregues**, agrupado pela `shipped_date`, considerando apenas pedidos associados a lojas cujo `state = 'NY'`. A granularidade final é **dia de envio**.

**Para que serve:** permite acompanhar a evolução diária do valor das vendas entregues no recorte de NY e pode alimentar gráficos, acompanhamento comercial e análises temporais.

**Com o que é feita:** parte de `silver_orders` e utiliza:

- `total_sale`: valor líquido de cada item;
- `state`: estado da loja responsável pelo pedido;
- `status`: descrição do status do pedido;
- `shipped_date`: data de envio.

**Regras aplicadas:**

```text
state = 'NY'
status = 'Delivered'
shipped_date IS NOT NULL
```

Depois dos filtros:

```text
ROUND(SUM(total_sale), 2) GROUP BY shipped_date
```

**Interpretação:** se, em uma determinada `shipped_date`, os itens elegíveis totalizarem 25.430,75, a Gold retorna esse valor como o total diário das vendas entregues de NY para aquela data.

> Importante: o notebook usa a `shipped_date` como eixo temporal, embora filtre pedidos cujo status é `Delivered`. Assim, a métrica é agrupada pela **data de envio**, não por uma data de entrega ao cliente. O dataset não contém um campo específico de data de entrega.

### 4. `gold_orders_pending.quantity` - Quantidade de itens em pedidos pendentes

**O que retorna:** a soma das quantidades dos itens associados aos pedidos com status `Pending`, agrupada por `customer_id`, `store_name` e `order_date`.

**Para que serve:** dimensiona o volume de itens que permanece associado a pedidos pendentes e prepara uma base operacional que pode ser utilizada para acompanhamento e contato com clientes.

**Com o que é feita:** utiliza:

- `silver_orders.status` para selecionar somente `Pending`;
- `silver_orders.quantity` para somar as unidades;
- `customer_id`, `store_name` e `order_date` como chaves do agrupamento;
- `silver_customer` para acrescentar `first_name`, `email` e `phone`.

**Fórmula/regra implementada:**

```text
SUM(quantity)
GROUP BY customer_id, store_name, order_date
WHERE lower(status) = 'pending'
```

Após a agregação, o resultado é relacionado à Silver de clientes e somente permanecem registros com `email` e `phone` não nulos.

**Interpretação:** um resultado com `quantity = 4` significa que existem **4 unidades de itens** no agrupamento pendente daquele cliente, loja e data. Não significa necessariamente quatro pedidos distintos.

### Resumo das métricas

| Métrica | Camada | Granularidade | Cálculo principal | Uso de negócio |
|---|---|---|---|---|
| `total_sale` | Silver | Item do pedido | `(list_price × quantity) × (1 - discount)` | Valor líquido comercial do item |
| `total_stock` | Silver | Produto | `SUM(quantity)` por produto | Estoque consolidado na rede |
| `gold_sales_ny.total_sale` | Gold | `shipped_date` | `SUM(total_sale)` após filtros NY + Delivered | Acompanhamento diário das vendas entregues de NY |
| `gold_orders_pending.quantity` | Gold | Cliente + loja + data do pedido | `SUM(quantity)` para status Pending | Volume de itens associado a pedidos pendentes e contactáveis |

### Indicadores categóricos utilizados nas regras

Além das métricas numéricas, o projeto cria o indicador textual `status` a partir de `order_status`. Ele não é uma métrica quantitativa, mas é fundamental para segmentar o processo do pedido:

| `order_status` | `status` | Interpretação no pipeline |
|---:|---|---|
| 1 | Pending | Pedido pendente |
| 2 | Processing | Pedido em processamento |
| 3 | Shipped | Pedido enviado |
| 4 | Delivered | Pedido entregue |
| Outros | Unknown | Código fora do mapeamento definido |

Esse indicador é utilizado diretamente nas duas regras Gold: `Delivered` para `gold_sales_ny` e `Pending` para `gold_orders_pending`.

## Linhagem dos Dados

```text
brands ─────┐
categories ─┼─> silver_product
products ───┤
stocks ─────┘

orders ─────┐
stores ─────┼─> silver_orders ─────> gold_sales_ny
staffs ─────┤          │
order_items ┘          └────────────> gold_orders_pending
                                      ↑
customers ─────> silver_customer ─────┘
```

## Regras de Qualidade e Observações

- A camada Bronze preserva o conteúdo de origem e usa inferência automática de schema.
- `customers.phone` possui valores ausentes na origem; a Silver remove clientes sem telefone/e-mail válidos.
- `orders.shipped_date` pode ser nula para pedidos ainda não enviados.
- `staffs.manager_id` pode ser nulo para o nível superior da hierarquia.
- As cargas atuais utilizam `overwrite`, portanto representam recarga completa e não carga incremental.
- `mergeSchema=true` permite evolução de schema durante a escrita Delta.

## Estrutura do Pipeline

1. Preparação do catálogo, schema, external volume e caminhos no ADLS.
2. Ingestão dos nove CSVs na Bronze.
3. Construção de `silver_product`.
4. Construção de `silver_orders`.
5. Construção de `silver_customer`.
6. Construção de `gold_sales_ny`.
7. Construção de `gold_orders_pending`.

## Entrega de Valor

A arquitetura separa responsabilidades entre ingestão, tratamento e consumo. A Silver oferece entidades reutilizáveis e confiáveis, enquanto a Gold materializa casos de negócio objetivos: acompanhamento diário de vendas entregues em NY e identificação de pedidos pendentes com dados de contato para ações operacionais.

## Melhorias Futuras

- Substituir cargas completas por estratégia incremental quando aplicável.
- Implementar validações automatizadas de qualidade de dados.
- Padronizar nomes de tabelas e notebooks.
- Adicionar monitoramento e tratamento de falhas do pipeline.
- Implementar CI/CD para notebooks e configurações.
- Explorar lineage e governança no Unity Catalog.
- Avaliar Auto Loader para ingestão incremental de novos arquivos.

## Conclusão

O projeto demonstra um fluxo Lakehouse completo em Databricks, partindo de arquivos CSV e evoluindo os dados pelas camadas Bronze, Silver e Gold. Além da persistência em Delta Lake, são aplicadas integrações, regras de qualidade, cálculos de negócio e agregações voltadas ao consumo analítico e operacional.
