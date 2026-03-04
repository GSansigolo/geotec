# Carregando Dados na Linha de Comando: `shp2pgsql`


**1.** Unidades Federativas do Brasil - 2018:

```bash
shp2pgsql -c -g "geom" -s 4674 -i -I -t "2D" -W UTF-8 BRUFE250GC_SIR.shp public.uf > uf.sql
```

```bash
psql -h localhost -p 5432 -d bdgeo -U postgres -f uf.sql
```

----


**2.** Municípios do Brasil - 2018:

```bash
shp2pgsql -c -g "geom" -s 4674 -i -I -t "2D" -W UTF-8 BRMUE250GC_SIR.shp public.municipios > municipios.sql
```

```bash
psql -U postgres -h localhost -p 5432 -d bdgeo -f municipios.sql
```

----


**3.** Terras Indígenas:

```bash
shp2pgsql -c -g "geom" -s 4674 -i -I -t "2D" -W LATIN1 tis_poligonaisPolygon.shp public.terras_indigenas > terras_indigenas.sql
```

```bash
psql -U postgres -h localhost -p 5432 -d bdgeo -f terras_indigenas.sql
```

----


**4.** Unidades de Conservação:

```bash
shp2pgsql -c -g "geom" -s 4674 -i -I -t "2D" -W LATIN1 cnuc_2024_02.shp public.unidades_conservacao > unidades_conservacao.sql
```

```bash
psql -h localhost -p 5432 -d bdgeo -U postgres -f unidades_conservacao.sql
```

----


**5.** Focos de Queimada (setembro/outubro de 2025):

```bash
shp2pgsql -c -g "geom" -s 4326 -i -t "2D" -W UTF-8 focos-queimada.shp public.focos > focos.sql
```

```bash
psql -h localhost -p 5432 -d bdgeo -U postgres -f focos.sql
```

```sql
ALTER TABLE focos
    ALTER COLUMN datahora
        TYPE TIMESTAMP WITHOUT TIME ZONE
        USING datahora::timestamp without time zone;
```

```sql
ALTER TABLE focos
    ALTER COLUMN geom
        TYPE GEOMETRY(POINT, 4674)
        USING ST_Transform(geom, 4674);
```

```sql
CREATE INDEX focos_datahora_idx ON focos(datahora);
```

```sql
CREATE INDEX focos_geom_idx ON focos USING GIST(geom);
```

----


**6.** Trechos Rodoviários – 2019:

```bash
shp2pgsql -c -g "geom" -s 4674 -i -I -t "2D" -W UTF-8 rod_trecho_rodoviario_l.shp public.trechos_rodoviarios > trechos_rodoviarios.sql
```

```bash
psql -h localhost -p 5432 -d bdgeo -U postgres -f trechos_rodoviarios.sql
```

----


**7.** Mapa Pedológico – 2017:

```bash
shp2pgsql -c -g "geom" -s 4674 -i -I -t "2D" -W UTF-8 Brasil_pedo_area.shp public.pedologia > pedologia.sql
```

```bash
psql -h localhost -p 5432 -d bdgeo -U postgres -f pedologia.sql
```
