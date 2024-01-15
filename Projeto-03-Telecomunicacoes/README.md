# O Desafio

Como analista de uma empresa de telecomunicações, que anunciou a aquisição da TELCO, uma empresa jovem e com muito potencial, cujas filiais estão localizadas em 1.106 cidades do estado da Califórnia, Estados Unidos, avaliar os desafios em termos de perdas de cliente da Telco. Atualmente a principal hipótese é a concorrência acirradas, principalmente na região de San Diego e a solução proposta é investir em contratos de longo prazo.

O desafio consiste em validar ou não essa hipótese, através da exploração do banco de dados e contrato atuais e passados.
Obter  o Cluster de clientes de “Alto risco de abandono e de alto retorno financeiro".

## Skills/Tools:
  - Power BI
  - Google Cloud
  - SQL
  - Análise de Dados
  - Decisões de Negócios apoiadas por dados
  
 # Resolução

- [Dashboard no Power BI](https://app.powerbi.com/view?r=eyJrIjoiYmFiYTcwMzUtNDFiNS00NDAwLThlYzUtOTg5ZWI4ODNhYzdjIiwidCI6IjYzYTNiY2VhLTk1ZWEtNDVlZC05YWE4LTA1Yjk3ZDkwODM2MCJ9)

## 1. Entendendo os dados
As seguintes tabelas foram fornecidas:
- **Services**: Informações dos serviços contratatos pelo cliente, assim como os valores pagos mensalmentes, cobranças extras, e indicações de clientes.
- **Churn_demographics**: Informações sobre o cliente como gênero, idade, estado civil e número de filhos.
- **location**: Informações geograficos sobre o cliente.
- **population**: Informações sobre a população de cada cidade
- **status**: Informações sobre a perda de clientes, status atual do cliente e motivo de cancelamento.

## 2. Preparando os dados
### Unificando em uma única tabela
```
CREATE TABLE  `perda-de-clientes.churn.master_view` as (
SELECT l.* EXCEPT(Zip_Code),
d.Gender,d.Age, d.Under_30, d.Senior_Citizen,d.Married,d.Number_of_Dependents, s.*EXCEPT (Customer_ID, Count),
st.Churn_Label,st.Churn_Category, st.Churn_Value, st.Churn_Reason
FROM perda-de-clientes.churn.location as l
LEFT JOIN perda-de-clientes.churn.churn_demographics as d
ON l.Customer_ID=d.Customer_ID
LEFT JOIN perda-de-clientes.churn.Services as s
ON d.Customer_ID=s.Customer_ID
LEFT JOIN perda-de-clientes.churn.status as st
ON s.Customer_ID=st.Customer_ID)
```

### Explorando os dados
- Número de Registros: 7043

```
SELECT COUNT(Customer_ID) FROM `perda-de-clientes.churn.master_view` ;

```
- Explorar idade dos usuários:

```
SELECT 
AVG(Age) AS idade_media,  #Resultado: 47,5
MIN(Age) as idade_minima, #Resultado: 19
MAX(Age) as idade_maxima  #Resultado: 119
from `perda-de-clientes.churn.master_view` ;
```
- Número de Clientes por sexo:
F - 638	
Male - 2918
Female - 2850
M - 637
```
SELECT DISTINCT Gender,
COUNT(Customer_ID) AS clientes_totais
FROM `perda-de-clientes.churn.master_view`
GROUP BY
1;
```
- Explorar pagamentos mensais
```
SELECT 
AVG(Monthly_Charge) AS pagamento_medio,   #Resultado: $63,6
MIN(Monthly_Charge) AS pagamento_minimo,  #Resultado: -$10.00
MAX(Monthly_Charge) AS pagamento_maximo   #Resultado: $118,75
FROM `perda-de-clientes.churn.master_view`;
```
- Explorar Receita por trimestre por cliente
```
SELECT 
AVG(Total_Revenue) AS receita_media,    #Resultado: $3034,40
MIN(Total_Revenue) AS receita_minima,   #Resultado: $21,36
MAX(Total_Revenue) AS receita_maxima    #Resultado: 11979,34
FROM `perda-de-clientes.churn.master_view`;
```
- Explorar estado civil		
Solteiros: 3641
Casados: 3402
```
SELECT DISTINCT Married,
COUNT(Customer_ID) AS clientes_totais
FROM `perda-de-clientes.churn.master_view`
GROUP BY
1;
```
- Explorar cidades com maior número de clientes
Los Angeles: 293
San Diego: 285
San Jose: 112	
Sacramento: 108
San Francisco: 104
```
SELECT DISTINCT City,
COUNT(Customer_ID) AS clientes_totais
FROM `perda-de-clientes.churn.master_view`
GROUP BY
City
ORDER BY clientes_totais DESC
LIMIT 5
```
### Limpeza da tabela
Apagar registros com idade maior que 95, com monthly charges menor que 0 e unificar sexo em Male e Female
```
DELETE  FROM `perda-de-clientes.churn.master_view_limpa` WHERE Age>95;
DELETE  FROM `perda-de-clientes.churn.master_view_limpa` WHERE Monthly_Charge<0;

UPDATE `perda-de-clientes.churn.master_view_limpa`
SET Gender = 'Male'
WHERE Gender = "M";

UPDATE `perda-de-clientes.churn.master_view_limpa`
SET Gender = 'Female'
WHERE Gender = "F";
```
## 3. Segmentação de Clientes:
Segmentação dos clientes por idade, por número de indicações, por número de dependentes, e por tempo de serviço.
- Idade
```
WITH
  tabela AS (
  SELECT
    *,
    CASE
	 WHEN Age < 41 THEN '19 a 40 anos'
    WHEN Age BETWEEN 41 AND 60 THEN '41 a 60' 
    ELSE 'Mais de 60 anos'
  END
    AS range_age
  FROM
    `perda-de-clientes.churn.master_view_limpa` )
SELECT
Customer_ID,
  City,
  Gender,
  Married,
  Number_of_Dependents,
  range_age,
  
FROM
  tabela
```
- Número de indicações:
```
WITH
  tabela AS (
  SELECT
    *,
    CASE
    WHEN Number_of_Referrals=0 THEN '0 referências'
	  WHEN Number_of_Referrals BETWEeN 1 and 4 THEN '1 a 4 referências'
    WHEN Number_of_Referrals BETWEeN 5 and 8 THEN '5 a 8 referências'
    ELSE 'Mais de 8 referências'
  END
    AS range_referrals
  FROM
    `perda-de-clientes.churn.master_view_limpa` )
SELECT
Customer_ID,
range_referrals
   
FROM
  tabela
```
- Número de Dependentes
```
WITH
  tabela AS (
  SELECT
    *,
    CASE
    WHEN Number_of_Dependents=0 THEN '0 dependentes'
	  WHEN Number_of_Dependents BETWEeN 1 and 4 THEN '1 a 4 dependentes'
    WHEN Number_of_Dependents BETWEeN 5 and 8 THEN '5 a 8 dependentes'
    ELSE 'Mais de 8 referências'
  END
    AS range_dependents
  FROM
    `perda-de-clientes.churn.master_view_limpa` )
SELECT
Customer_ID,
range_dependents

FROM
  tabela
```
- Tempo de Serviço
```
WITH
  tabela AS (
  SELECT
    *,
    CASE
    WHEN Tenure_in_Months BETWEEN 0 AND 12 THEN 'Até 1 ano'
	  WHEN Tenure_in_Months BETWEEN 13 AND 24 THEN '1 a 2'
    WHEN Tenure_in_Months BETWEEN 25 AND 36 THEN '2 a 3'
    WHEN Tenure_in_Months BETWEEN 37 AND 48 THEN '3 a 4'
    WHEN Tenure_in_Months BETWEEN 49 AND 62 THEN '4 a 5'
    WHEN Tenure_in_Months >62 THEN '5 ou mais'
  END
    AS range_tenure_year
  FROM
    `perda-de-clientes.churn.master_view_limpa` )
SELECT
Customer_ID,
range_tenure_year,
Tenure_in_Months
  
  
FROM
  tabela
```
## 4. Segmentar Grupos de Risco
O churn rate total é 26,7% e pela exploração dos dados chegamos a seguinte segmentação de risco:
- Grupo 1: Contrato mes-a-mes e idade > 64 -                                         churn: 82,2%
- Grupo 2: Contrato mes-a-mes e Idade < 64 e número de referências < 2 -             churn: 45,6%
- Grupo 3: Contrato diferente de mes-a-mes, Idade > 64 y número de referidos < 2 -   churn: 8,7%
- Grupo 4: Contrato diferente a mes-a-mes e antiguidade em meses < 40 -               churn: 1,6%
- Grupo 5: Cliente sem Suporte técnico Premium -                                      churn: 8,3%
 
```
CREATE OR REPLACE TABLE perda-de-clientes.churn.segmento_risco AS(
WITH
  tabela AS (
  SELECT
    *,
    CASE
    WHEN Age>64 and Contract ='Month-to-Month' THEN 'G1'
    WHEN Age>64 and Contract <>"Month-to-Month" and Number_of_Referrals<2 Then "G3"
    WHEN Age<64 and Number_of_Referrals<2 THEN 'G2'
    WHEN Contract <> "Month-to-Month" and Tenure_in_Months<40 THEN "G4"
    WHEN  Premium_Tech_Support = false Then "G5"
    ELSE "Sem Grupo"
    END

    AS range_riskGroups
  FROM
    `perda-de-clientes.churn.master_view_limpa` )
SELECT
Customer_ID,
range_riskGroups
Churn_Label
From tabela
)
```  
## 5. Segmentar Clientes de alto valor
Primeiro calcula-se o Tenure Médio para cada tipo de contrato:
```
CREATE OR REPLACE  TABLE `perda-de-clientes.churn.master_view_limpa` AS(
    WITH base_tenure_prom AS (
        SELECT Contract,AVG(Tenure_in_Months) AS media_tenure
        FROM `perda-de-clientes.churn.master_view_limpa`
        GROUP BY 1
)
SELECT a.*,
        b.media_tenure
FROM `perda-de-clientes.churn.master_view_limpa` as a
LEFT JOIN base_tenure_prom as b
ON a.Contract = b.Contract
)
```
Calculando a receita total estimada para cada cliente (Estimated_Revenue) com base nos pagamentos mensais e no Tenure médio calculado no passo anterior
```
CREATE OR REPLACE TABLE perda-de-clientes.churn.master_view_limpa AS(
Select * ,
        media_tenure*Monthly_Charge as Estimated_Revenue
FROM perda-de-clientes.churn.master_view_limpa
)
```
Dividindo os clientes ativos em quartis de receita estimada por tipo de contrato:
```
CREATE OR REPLACE TABLE
  perda-de-clientes.churn.quartil AS (
  SELECT
    Customer_ID,
    NTILE(4) OVER(PARTITION BY Contract ORDER BY Estimated_Revenue ASC) AS quartil_estimado
  FROM
    perda-de-clientes.churn.master_view_limpa
  WHERE
    Churn_Value=0 )
```

## 6. Dashboard
![Dashboard: Visao Geral](https://github.com/Bertimaz/Data-Science/blob/cd4245c35a830036da9a2cc4f5555a2a51bdbb41/Telecomunicacoes/img/Telecom_DashBoard01.png)
![Dashboard: Encerramentos](https://github.com/Bertimaz/Data-Science/blob/cd4245c35a830036da9a2cc4f5555a2a51bdbb41/Telecomunicacoes/img/Telecom_DashBoard02.png)
![Dashboard: Grupos de Risco](https://github.com/Bertimaz/Data-Science/blob/cd4245c35a830036da9a2cc4f5555a2a51bdbb41/Telecomunicacoes/img/Telecom_DashBoard03.png)
![Dashboard: Grupos de Risco ](https://github.com/Bertimaz/Data-Science/blob/cd4245c35a830036da9a2cc4f5555a2a51bdbb41/Telecomunicacoes/img/Telecom_DashBoard04.png)
