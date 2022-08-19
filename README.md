
# Consultas:

## 1ª Query (TPC-H Q12)
This query determines whether selecting less expensive modes of shipping is negatively affecting the critical-priority orders by causing more parts to be received by customers after the committed date.
Essa query determina se optar por formas mais baratas de transporte está afetando negativamente os pedidos de prioridade urgente, pelo atraso no prazo de entrega das partes.




```sql
 
SELECT l_shipmode,
       SUM(CASE
             WHEN o_orderpriority = '1-URGENT'
                   OR o_orderpriority = '2-HIGH' THEN 1
             ELSE 0
           END) AS high_line_count,
       SUM(CASE
             WHEN o_orderpriority <> '1-URGENT'
                  AND o_orderpriority <> '2-HIGH' THEN 1
             ELSE 0
           END) AS low_line_count
FROM   orders,
       lineitem
WHERE  o_orderkey = l_orderkey
       AND l_shipmode IN ( 'MAIL', 'SHIP' )
       AND l_commitdate < l_receiptdate
       AND l_shipdate < l_commitdate
       AND l_receiptdate >= DATE '1994-01-01'
       AND l_receiptdate < DATE '1994-01-01' + interval '1' year
GROUP  BY l_shipmode
ORDER  BY l_shipmode; 
```

### Média do tempo da consulta não otimizada

PostgreSQL: 0.320 - 0.330 - 0.315 - 0.437 - 0.310 = Média: 0.342 sec

MySQL: 0.297 - 0.282 - 0.298 - 0.285 - 0.282 = Média: 0.289 sec

![1](https://media.discordapp.net/attachments/441059716185980929/1009675074179170364/unknown.png)

Evitando o full table scan em lineitem, utilizamos a criação do index composto `CREATE INDEX L_SHIPMODE_RECEIPTDATE_Index ON lineitem (l_shipmode,l_receiptdate)` no MySQL.

Utilizando o mesmo método no PostgreSQL também obtivemos uma melhora de desempenho significativa durante a consulta.

![1](https://media.discordapp.net/attachments/744351225381781594/1009841375681982594/unknown.png)

### Média do tempo da consulta otimizada

PostgreSQL: 0.264 - 0.262 - 0.270 - 0.268 - 0.260 = Média: 0.265 sec

MySQL: 0.203 - 0.141 - 0.180 - 0.109 - 0.162 = Média: 0.159 sec

## 2ª Query (TPC-H Q3)
Essa query retorna os 10 pedidos mais caros que ainda não foram enviados.

```sql
SELECT l_orderkey,
       SUM(l_extendedprice * ( 1 - l_discount )) AS revenue,
       o_orderdate,
       o_shippriority
FROM   customer,
       orders,
       lineitem
WHERE  c_mktsegment = 'BUILDING'
       AND c_custkey = o_custkey
       AND l_orderkey = o_orderkey
       AND o_orderdate < DATE '1995-03-15'
       AND l_shipdate > DATE '1995-03-15'
GROUP  BY l_orderkey,
          o_orderdate,
          o_shippriority
ORDER  BY revenue DESC,
          o_orderdate; 
```

### Média do tempo da consulta não otimizada

PostgreSQL: 0.256 - 0.243 - 0.225 - 0.229 - 0.240 = Média: 0.238 sec

MySQL: 0.468 - 0.453 - 0.407 - 0.391 - 0.442 = Média: 0.432 sec

![1](https://media.discordapp.net/attachments/744351225381781594/1009676819366154313/unknown.png)

Evitando o full table scan em customer, utilizamos a criação do index `CREATE INDEX C_MKTSEGMENT_Index ON customer (c_mktsegment)` no MySQL

O mesmo método funcionou no PostgreSQL, porém não teve uma grande diferença de desempenho como teve no MySQL. No postgreSQL, a etapa que mais demora é a comparação do o_orderdate, que mesmo criando um index para o mesmo, o postgre nao o utiliza durante sua execução.

![1](https://media.discordapp.net/attachments/744351225381781594/1009827972666101801/unknown.png)

### Média do tempo da consulta otimizada

PostgreSQL: 0.235 - 0.242 - 0.243 - 0.238 - 0.252 = Média: 0.242 sec

MySQL: 0.204 - 0.187 - 0.190 - 0.195 - 0.182 = Média: 0.191 sec

## 3ª Query (TPC-H Q19)
Essa query retorna a receita descontada total atribuida a venda das partes selecionadas que são gerenciadas de uma maneira específica.


```sql
SELECT Sum(l_extendedprice * ( 1 - l_discount )) AS revenue
FROM   lineitem,
       part
WHERE  ( p_partkey = l_partkey
         AND p_brand = 'Brand#12'
         AND p_container IN ( 'SM CASE', 'SM BOX', 'SM PACK', 'SM PKG' )
         AND l_quantity >= 1
         AND l_quantity <= 1 + 10
         AND p_size BETWEEN 1 AND 5
         AND l_shipmode IN ( 'AIR', 'AIR REG' )
         AND l_shipinstruct = 'DELIVER IN PERSON' )
        OR ( p_partkey = l_partkey
             AND p_brand = 'Brand#23'
             AND p_container IN ( 'MED BAG', 'MED BOX', 'MED PKG', 'MED PACK' )
             AND l_quantity >= 10
             AND l_quantity <= 10 + 10
             AND p_size BETWEEN 1 AND 10
             AND l_shipmode IN ( 'AIR', 'AIR REG' )
             AND l_shipinstruct = 'DELIVER IN PERSON' )
        OR ( p_partkey = l_partkey
             AND p_brand = 'Brand#34'
             AND p_container IN ( 'LG CASE', 'LG BOX', 'LG PACK', 'LG PKG' )
             AND l_quantity >= 20
             AND l_quantity <= 20 + 10
             AND p_size BETWEEN 1 AND 15
             AND l_shipmode IN ( 'AIR', 'AIR REG' )
             AND l_shipinstruct = 'DELIVER IN PERSON' ); 
```

### Média do tempo da consulta não otimizada

PostgreSQL: 0.186 - 0.157 - 0.156 - 0.197 - 0.167 = Média: 0.172 sec

MySQL: 0.172 - 0.156 - 0.156 - 0.156 - 0.156 = Média: 0.159 sec

![1](https://media.discordapp.net/attachments/441059716185980929/1009676154585743490/unknown.png)

Evitando o full table scan em Part, utilizamos a criação do index `CREATE INDEX P_BRAND_CONTAINER_SIZE_Index ON part (P_BRAND, P_CONTAINER, P_SIZE)` no MySQL.

O mesmo método funcionou no PostgreSQL, porém não obteve uma melhora tão significativa quanto no MySQL.

![1](https://media.discordapp.net/attachments/815287365747736630/1009949750184919142/unknown.png)

### Média do tempo da consulta otimizada

PostgreSQL: 0.142 - 0.148 - 0.152 - 0.144 - 0.142 = Média: 146 sec

MySQL: 0.062 - 0.080 - 0.068 - 0.076 - 0.072 = Média: 0.071 sec

## 4ª Query (TPC-H Q8)
Essa query determina como a participação no mercado de uma nação em uma região mudou nos últimos dois anos, com relação a uma parte específica.

```sql
SELECT o_year,
       SUM(CASE
             WHEN nation = 'BRAZIL' THEN volume
             ELSE 0
           END) / SUM(volume) AS mkt_share
FROM   (SELECT Extract(year FROM o_orderdate)       AS o_year,
               l_extendedprice * ( 1 - l_discount ) AS volume,
               n2.n_name                            AS nation
        FROM   part,
               supplier,
               lineitem,
               orders,
               customer,
               nation n1,
               nation n2,
               region
        WHERE  p_partkey = l_partkey
               AND s_suppkey = l_suppkey
               AND l_orderkey = o_orderkey
               AND o_custkey = c_custkey
               AND c_nationkey = n1.n_nationkey
               AND n1.n_regionkey = r_regionkey
               AND r_name = 'AMERICA'
               AND s_nationkey = n2.n_nationkey
               AND o_orderdate BETWEEN DATE '1995-01-01' AND DATE '1996-12-31'
               AND p_type = 'ECONOMY ANODIZED STEEL') AS all_nations
GROUP  BY o_year
ORDER  BY o_year;
```
### Média do tempo da consulta não otimizada

PostgreSQL: 0.263 - 0.164 - 0.155 - 0.168 - 0.160 = Média: 0.182 sec

MySQL: 0.656 - 0.641 - 0.328 - 0.312 - 0.328 = Média: 0.453 sec

![1](https://media.discordapp.net/attachments/744351225381781594/1009677155711594556/unknown.png?width=1440&height=415)

Evitando o full table scan em Orders, utilizamos a criação do index `CREATE INDEX O_CUSTKEY_ORDERDATE_Index ON orders (O_CUSTKEY, O_ORDERDATE)` e do `CREATE INDEX R_NAME_index on region (R_NAME)` no MySQL.

O mesmo método não funcionou no postgreSQL, o mesmo não utilizou os index criados.

![1](https://media.discordapp.net/attachments/815287365747736630/1009962989497090128/unknown.png?width=1440&height=316)

### Média do tempo da consulta otimizada

PostgreSQL: (Otimização não aplicada)

MySQL: 0.235 - 0.250 - 0.262 - 0.243 - 0.252 = Média: 0.248 sec

## 5ª Query (TPC-H Q10)
Essa query identifica clientes que podem estar tendo problema com partes que foram enviadas a eles.

```sql
SELECT c_custkey,
       c_name,
       SUM(l_extendedprice * ( 1 - l_discount )) AS revenue,
       c_acctbal,
       n_name,
       c_address,
       c_phone,
       c_comment
FROM   customer,
       orders,
       lineitem,
       nation
WHERE  c_custkey = o_custkey
       AND l_orderkey = o_orderkey
       AND o_orderdate >= DATE '1993-10-01'
       AND o_orderdate < DATE ' 1993-10-01' + interval '3' month
       AND l_returnflag = 'R'
       AND c_nationkey = n_nationkey
GROUP  BY c_custkey,
          c_name,
          c_acctbal,
          c_phone,
          n_name,
          c_address,
          c_comment
ORDER  BY revenue DESC; 
```

### Média do tempo da consulta não otimizada

PostgreSQL: 0.380 - 0.357 - 0.370 - 0.333 - 0.338 = Média: 0.395 sec

MySQL: 0.406 - 0.359 - 0.343 - 0.359 - 0.343 = Média: 0.362 sec

![1](https://media.discordapp.net/attachments/744351225381781594/1009678791796346920/unknown.png)

Evitando o full table scan em Orders, utilizamos a criação do index `CREATE INDEX O_ORDERDATE_Index ON orders (O_ORDERDATE)` no MySQL

Essa mesma medida não teve eficácia no PostegreSQL, pois o index criado acabou não sendo utilizado

![1](https://media.discordapp.net/attachments/744351225381781594/1009701802276560926/unknown.png)

### Média do tempo da consulta otimizada

PostgreSQL: (Otimização não aplicada)

MySQL: 0.157 - 0.182 - 0.164 - 0.153 - 0.162 = Média: 0.164 sec


