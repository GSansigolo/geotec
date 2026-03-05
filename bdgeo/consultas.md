# Criação do banco de dados 

**1.** Para criar um novo banco de dados chamado `bdgeo`, use o banco `template1` como molde:

```sql
CREATE DATABASE bdgeo TEMPLATE template1;
```


**2.** Ao se conectar no novo banco criado, carregue a extensão PostGIS com o seguinte comando:

```sql
CREATE EXTENSION postgis; 
```


**3.** Para saber as configurações da sua extensão:

```sql
SELECT postgis_full_version();
```

# Criando uma tabela com coluna geométrica


**1.** Vamos criar uma tabela para armazenar a localização de escolas:

```sql
CREATE TABLE escolas
(
    gid          SERIAL PRIMARY KEY,
    nome         VARCHAR(100),
    localizacao  GEOMETRY(POINT, 4326)
);
```


**2.** Vamos inserir três novas linhas na tabela `escolas`:

```sql
INSERT INTO escolas (nome, localizacao)
     VALUES ('Escola Estadual Arlindo Bittencourt',
             ST_GeomFromText('POINT(-47.88497 -22.02557)', 4326)),

           ('Colégio Arquidiocesano de Ouro Preto',
            ST_GeomFromText('POINT(-43.51592 -20.38144)', 4326)),

           ('Instituto São José',
            ST_GeomFromText('POINT(-45.90245 -23.20000)', 4326));
```


**3.** Para recuperar os dados da tabela, faça:

```sql
SELECT * FROM escolas;
```


**4.** Para obter a representação textual (WKT) da coluna geométrica, use a função `ST_AsText`:

```sql
SELECT gid, nome, ST_AsText(localizacao) 
  FROM escolas;
```


# OGC Well Know Text (WKT) e Well Know Binary (WKB)

Exemplos:

```sql
SELECT ST_GeomFromText('POINT(1 8)');
```

```sql
SELECT ST_GeomFromText('LINESTRING(1 5, 3 6, 4 5)');
```

```sql
SELECT ST_GeomFromText(
'POLYGON( (1 1, 2 3, 5 4, 5 1, 1 1),
          (3 2, 4 3, 4 2, 3 2) )');
```

```sql
SELECT ST_GeomFromText('POINT(1 8)', 4326);
```

```sql
SELECT ST_GeomFromText('LINESTRING(1 5, 3 6, 4 5)', 4326);
```

```sql
SELECT ST_GeomFromText(
'POLYGON( (1 1, 2 3, 5 4, 5 1, 1 1),
          (3 2, 4 3, 4 2, 3 2) )', 4326);
```

```sql
SELECT 'SRID=4326;POINT(1 8)'::geometry;
```

```sql
SELECT 'SRID=4326;LINESTRING(1 5, 3 6, 4 5)'::geometry;
```

```sql
SELECT 'SRID=4326;POLYGON(
                    (1 1, 2 3, 5 4, 5 1, 1 1),
                     (3 2, 4 3, 4 2, 3 2) )'::geometry;
```

```sql
SELECT ST_AsBinary(ST_GeomFromText('POLYGON(
                                      (1 1, 2 3, 5 4, 5 1, 1 1),
                                      (3 2, 4 3, 4 2, 3 2) )', 4326));
```


# Consultando as tabelas de metadados espaciais


```sql
SELECT * FROM geometry_columns;
```

```sql
SELECT * FROM spatial_ref_sys WHERE srid=4326;
```

# Realizando a transformação entre sistemas de coordenadas de referência (CRS)  

```sql
SELECT * FROM spatial_ref_sys WHERE srid=5880;
```

```sql
SELECT gid, 
       nome,
       ST_AsText(localizacao) AS geom_wgs84,
       ST_AsText(
           ST_Transform(localizacao, 5880)
       ) AS geom_polyconic
  FROM escolas;
```


# Consultas Espaciais

**Q1.** Computar a área dos municípios brasileiros.

```sql
SELECT gid, nm_municip, ST_Area(geom)
  FROM municipios;
```

```sql
SELECT gid, nm_municip,
       ST_Area(ST_Transform(geom, 5880))
  FROM municipios;
```

```sql
INSERT INTO spatial_ref_sys (srid, proj4text)
    VALUES (100000, '+proj=aea +lat_1=-2 +lat_2=-22 +lat_0=-12 +lon_0=-54 +x_0=5000000 +y_0=10000000 +ellps=GRS80 +units=m +no_defs ');
```

```sql
SELECT gid, nm_municip,
       ST_Area(ST_Transform(geom, 100000))
  FROM municipios;
```

----


**Q2.** Qual UF encontra-se na localização de longitude -44.29 e latitude -18.61?

```sql
SELECT *
  FROM uf
 WHERE ST_Contains(
         geom,
         ST_GeomFromText('POINT(-44.29 -18.61)', 4674)
       );
```

----


**Q3.** Quais UF possuem geometrias com alguma interação espacial com o retângulo de coordenadas: 
xmin: -54.23    xmax: -43.89
ymin: -12.90    ymax: -21.49

```sql
SELECT *
  FROM uf
 WHERE ST_Intersects(
         geom,
         ST_MakeEnvelope(-54.23, -21.49, -43.89, -12.90, 4674)
       );

```

----


**Q4.** Quais os municípios num raio de 2 graus da coordenada:
- longitude: -43.59
- latitude.: -20.32

```sql
SELECT *
  FROM municipios
 WHERE ST_Distance(
           geom,
           ST_GeomFromText('POINT(-43.59 -20.32)', 4674)
       ) <= 2.0;

```

```sql
SELECT *
  FROM municipios
 WHERE ST_DWithin(
           geom,
           ST_GeomFromText('POINT(-43.59 -20.32)', 4674),
           2.0
       );
```

----


**Q5.** Quantos focos de incêndio na vegetação foram detectados em Unidades de Conservação Estaduais do Estado do Tocantins?

```sql
SELECT ucs.nome_uc AS nome,
         COUNT(*) AS total_focos
    FROM focos,
         unidades_conservacao AS ucs,
         uf
   WHERE uf.nm_estado = 'TOCANTINS'
     AND ST_Intersects(uf.geom, ucs.geom)
     AND ucs.esfera = 'Estadual'
     AND ST_Contains(ucs.geom, focos.geom)
GROUP BY ucs.gid,
         ucs.nome_uc
ORDER BY total_focos DESC;
```


# Métodos de Acesso

## Árvores-B<sup>+</sup> no PostgreSQL

```sql
CREATE TABLE pts
(
    id     UUID DEFAULT gen_random_uuid(),
    x      DOUBLE PRECISION,
    y      DOUBLE PRECISION,
    word   TEXT
);
```

```sql
INSERT INTO pts (x, y, word)
                (
                 SELECT 360.0 * random() - 180.0 AS x,
                        180.0 * random() - 90.0 AS y,
                        rpad(i::text, 10) AS word
                   FROM generate_series(1, 1000000) AS i
                );
```

```sql
EXPLAIN ANALYZE
    SELECT *
      FROM pts
     WHERE x > 1.0 AND x < 2.0
       AND y > 1.0 AND y < 2.0;
```

```sql
CREATE INDEX pts_x_y_idx ON pts(x, y);
```

```sql
ANALYZE pts;
```

```sql
EXPLAIN ANALYZE
    SELECT *
      FROM pts
     WHERE x > 1.0 AND x < 2.0
       AND y > 1.0 AND y < 2.0;
```

## Árvores-R no PostgreSQL com PostGIS e GiST

```sql
CREATE TABLE pts_pgis
(
    id     UUID DEFAULT gen_random_uuid(),
    geom   GEOMETRY(POINT, 4326),
    word   TEXT
);
```

```sql
INSERT INTO pts_pgis (geom, word)
         (
             SELECT ST_SetSRID(
                        ST_MakePoint(
                            360.0 * random() - 180.0,
                            180.0 * random() - 90.0
                        ),
                        4326
                    ),
                    rpad(i::text, 10) AS word
               FROM generate_series(1, 1000000) AS i
         );
```

```sql
EXPLAIN ANALYZE
    SELECT *
      FROM pts_pgis
     WHERE ST_Intersects(
              geom,
              ST_MakeEnvelope(1.0, 1.0, 2.0, 2.0, 4326)
           );
```


```sql
CREATE INDEX pts_pgis_geom_idx
          ON pts_pgis USING GIST (geom);
```


```sql
ANALYZE pts_pgis;
```


```sql
EXPLAIN ANALYZE
    SELECT *
      FROM pts_pgis
     WHERE ST_Intersects(
              geom,
              ST_MakeEnvelope(1.0, 1.0, 2.0, 2.0, 4326)
           );
```


```sql
```
