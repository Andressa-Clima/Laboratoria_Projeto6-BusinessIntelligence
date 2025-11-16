# 🧠 Projeto de Business Intelligence - Airbnb Data Analysis Project

Este documento descreve o processo completo de limpeza, transformação e modelagem dos dados do dataset Airbnb, incluindo etapas de inspeção, tratamento e criação de tabelas para uso analítico no Power BI.

## 🔵 1. Conectar e Importar Dados para Ferramentas

Carregamento dos dados para o Data Lake
Observação: O carregamento ocorreu normalmente no datalake, porém
alguns dados vieram com problemas desde a origem.

**Problema Identificado:**\
Alguns dados foram incluídos de forma incorreta já na origem do arquivo.

**Solução:**\
Registros afetados foram desconsiderados. Impacto total: **147 registros
(0.3%)**

## 🔵 2. Identificar e Tratar Valores Nulos

### **Tabela Hosts**

``` sql
SELECT
COUNT(*) AS total_linhas,
COUNT(host_id) AS host_id_preenchidos,
COUNT(host_name) AS host_name_preenchidos
FROM airbnbpowerbi.airbnb.hosts;
```

18 valores nulos em `host_name`.

**Tratamento:**

``` sql
SELECT
host_id,
IFNULL(host_name, 'desconhecido') AS host_name
FROM airbnbpowerbi.airbnb.hosts;
```

### **Tabela Reviews**

-   `number_of_reviews`: substituído por 0\
-   `last_review`: mantidos nulos\
-   `reviews_per_month`: substituído por 0\
-   `availability_365`: substituído por 0

``` sql
SELECT
id,
host_id,
price,
number_of_reviews,
IFNULL(last_review, '0') AS last_review,
IFNULL(reviews_per_month, 0) AS reviews_per_month,
calculated_host_listings_count,
availability_365
FROM airbnbpowerbi.airbnb.reviews;
```

### **Tabela Rooms**

Nenhum valor nulo encontrado.

## 🔵 3. Identificar e Tratar Valores Duplicados

### **Tabela Hosts**

Nenhum duplicado encontrado.

### **Tabela Reviews**

18 registros duplicados foram removidos:

``` sql
CREATE OR REPLACE TABLE airbnbpowerbi.airbnb.reviews AS
SELECT *
FROM airbnbpowerbi.airbnb.reviews
WHERE (id, host_id) NOT IN (...);
```

### **Tabela Rooms**

18 registros duplicados removidos:

``` sql
CREATE OR REPLACE TABLE airbnbpowerbi.airbnb.rooms AS
SELECT *
FROM airbnbpowerbi.airbnb.rooms
WHERE (id, name) NOT IN (...);
```

## 🔵 4. Tratar Dados Discrepantes --- Variáveis Categóricas

### **Limpeza de nomes de hosts**

``` sql
CREATE OR REPLACE TABLE airbnbpowerbi.airbnb.hosts_clean AS
SELECT 
    host_id,
    CASE
        WHEN REGEXP_CONTAINS(host_name, r'^[0-9]+$') THEN 'desconhecido'
        ELSE REGEXP_REPLACE(host_name, r'[@'"\(\)\+\*\&\%\$\#\!\?]', '')
    END as host_name
FROM airbnbpowerbi.airbnb.hosts;
```

## 🔵 5. Tratar Dados Discrepantes --- Variáveis Numéricas

### **Tabela Hosts**

Remoção de 123 registros inválidos:

``` sql
CREATE OR REPLACE TABLE airbnbpowerbi.airbnb.hosts_limpo AS
SELECT *
FROM airbnbpowerbi.airbnb.hosts
WHERE REGEXP_CONTAINS(CAST(host_id AS STRING), r'^[0-9]+$');
```

### **Tabela Rooms**

147 registros removidos:

``` sql
CREATE OR REPLACE TABLE airbnbpowerbi.airbnb.rooms_numericos AS
SELECT *
FROM airbnbpowerbi.airbnb.rooms
WHERE REGEXP_CONTAINS(id, r'^[0-9]+$');
```

## 🔵 6. Alterar Tipo de Dados

### **Tabela Hosts**

``` sql
CREATE OR REPLACE TABLE airbnbpowerbi.airbnb.hosts_corrigido AS
SELECT
CASE WHEN REGEXP_CONTAINS(host_id, r'^[0-9]+$') THEN CAST(host_id AS INT64) END as host_id,
host_name
FROM airbnbpowerbi.airbnb.hosts
WHERE REGEXP_CONTAINS(host_id, r'^[0-9]+$');
```

### **Tabela Rooms**

``` sql
CREATE OR REPLACE TABLE airbnbpowerbi.airbnb.rooms_corrigido AS 
SELECT  
    CAST(id AS INT64) as id,  
    name,  
    neighbourhood,  
    neighbourhood_group,  
    latitude,  
    CAST(longitude AS FLOAT64) as longitude,  
    room_type,  
    minimum_nights  
FROM airbnbpowerbi.airbnb.rooms_numericos;
```

### **Tabela Reviews**

``` sql
CREATE OR REPLACE TABLE airbnbpowerbi.airbnb.reviews_numericos AS
SELECT
CAST(id AS INT64),
CAST(host_id AS INT64),
price,
CAST(number_of_reviews AS INT64),
CASE WHEN last_review IS NOT NULL AND last_review != '' THEN PARSE_DATE('%Y-%m-%d', last_review) END,
reviews_per_month,
calculated_host_listings_count,
availability_365
FROM airbnbpowerbi.airbnb.reviews_numericos;
```

## 🔵 7. Relacionar Tabelas

-   **reviews_corrigido ↔ hosts_corrigido**\
    Chave: `host_id`\
    Cardinalidade: **N:1**

-   **rooms_corrigido ↔ reviews_corrigido**\
    Chave: `id`\
    Cardinalidade: **1:1**

## 🔵 8. Criar Novas Variáveis

### **Potential Revenue**

``` dax
Potential Annual Revenue =
RELATED(reviews_corrigido[price]) * RELATED(reviews_corrigido[availability_365])
```

## 🔵 9. Fórmulas DAX

### **Capacidade anual de hóspedes**

``` dax
Total Annual Guests =
IF(
'rooms_corrigido csv'[minimum_nights] > 0,
DIVIDE(365, 'rooms_corrigido csv'[minimum_nights], 0),
365
)
```

### **% de quartos disponíveis**

``` dax
% Available Rooms =
DIVIDE(
CALCULATE(
COUNTROWS('rooms_corrigido csv'),
reviews_corrigido[availability_365] > 0
),
COUNTROWS('rooms_corrigido csv'),
0
)
```

## 🔵 10. Preparação para Visualização

### **Correção de coordenadas**

``` dax
longitude_corrigida = 'rooms_corrigido csv'[longitude] / 100000
latitude_corrigida  = 'rooms_corrigido csv'[latitude]  / 100000
```

# 📊 Resultados e Conclusões

## Principais Insights

-   Manhattan e Brooklyn concentram maior número de anúncios
-   Imóveis inteiros são os mais rentáveis
-   Sazonalidade forte em fevereiro, junho e dezembro
-   Queens e Bronx são oportunidades de expansão

## Recomendações

-   Ajustar preços sazonalmente
-   Focar em imóveis inteiros
-   Expandir presença em bairros menos saturados
-   Otimizar disponibilidade anual

## 🎯 Dashboard

Inclui: - mapa geográfico com coordenadas corrigidas
- métricas de revenue potential
- sazonalidade
- indicadores por tipo e região

**ETL finalizado. Dashboard pronto para uso estratégico.**
