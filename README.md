# tutorial-terminal-d3js-ibge
Tutorial de como fazer mapas apenas com a linha de comando em D3js com os 
dados do IBGE. Na real, isso é uma adaptação do [tutorial do Mike Bostock](https://medium.com/@mbostock/command-line-cartography-part-1-897aa8f8ca2c), 
com a diferença que eu foquei em integrar com os dados da malha censitária.

## Convertendo a malha censitária em um mapa
Pegue os dados relativos a malha censitária de Rio de Janeiro. No FTP do IBGE
é possível encontrar de todas as UFs em _ftp://geoftp.ibge.gov.br/organizacao_do_territorio/malhas_territoriais/malhas_de_setores_censitarios__divisoes_intramunicipais/censo_2010/setores_censitarios_shp/_.  
  
Como não queremos sair do terminal, vamos usar o __curl__ para isso. E vamos
apontar para a malha de Rio de Janeiro 

```bash
curl \
  'ftp://geoftp.ibge.gov.br/organizacao_do_territorio/malhas_territoriais/malhas_de_setores_censitarios__divisoes_intramunicipais/censo_2010/setores_censitarios_shp/rj/rj_setores_censitarios.zip' \
  -o rj_setores_censitarios.zip
```

Depois é dezipar a pasta

```bash
unzip -o rj_setores_censitarios.zip
```

E vamos nós instalar mais um pacote, o shapefile. Ele precisa de 
Node e do NPM, então se você estiver usando um Mac eu recomendo usar o Homebrew. 
Entretanto, você pode confiar  com o instalador da página deles mesmo.

```bash
npm install -g shapefile
```

Vamos converter o SHP para GeoJSON

```bash
shp2json 33SEE250GC_SIR.shp --encoding 'utf8' -o rj.json
```

Vamos instalar as projeções do D3

```bash
npm install -g d3-geo-projection
```

E aplicar a projeção ortográfica ao estado do Rio de Janeiro

```bash
geoproject \
  'd3.geoOrthographic().rotate([42.5, 22.5, 0]).fitSize([1000, 600], d)' \
  < rj.json \
  > rj-ortho.json
```

Para finalizar, vamos converter a projeção em SVG

```bash
geo2svg \
  -w 1000 \
  -h 600 \
  < rj-ortho.json \
  > rj-ortho.svg
```

## Unindo os dados censitários a malha

```bash
npm install -g ndjson-cli
```

E vamos separar em um ndjson o mapa projetado

```bash
ndjson-split 'd.features' \
  < rj-ortho.json \
  > rj-ortho.ndjson
```

Depois vamos mapear os códigos dos setores para bater com a coluna do CSV

```bash
ndjson-map 'd.Cod_setor = d.properties.CD_GEOCODI, d' \
  < rj-ortho.ndjson \
  > rj-ortho-sector.ndjson
```

Vamos baixar todos os dados da amostra geral do Rio de Janeiro

```bash
curl \
  'ftp://ftp.ibge.gov.br/Censos/Censo_Demografico_2010/Resultados_do_Universo/Agregados_por_Setores_Censitarios/RJ_20171016.zip' \
  -o RJ_20171016.zip
```

Dezipar os dados

```bash
unzip -o RJ_20171016.zip
```

Depois vamos instalar o módulo de conversão de CSVs, TSVs e afins

```bash
npm install -g d3-dsv
```

Converteremos o CSV do censo para ndjson

```bash
dsv2json \
  -r ';' \
  -n \
  < RJ/Base\ informa\%E7oes\ setores2010\ universo\ RJ/CSV/Entorno01_RJ.csv \
  > rj-census.ndjson
```

E uniremos o arquivo de dados do censo com a projeção ortográfica

```bash
ndjson-join 'd.Cod_setor' \
  rj-ortho-sector.ndjson \
  rj-census.ndjson \
  > rj-ortho-census.ndjson
```

E descobriremos a % da população que não tem acesso a iluminação pública

```bash
ndjson-map \
  'd[0].properties = {light: Math.floor(100 * (Number(d[1].V009) + Number(d[1].V011)) / d[1].V001)}, d[0]' \
  < rj-ortho-census.ndjson \
  > rj-ortho-light.ndjson
```

## Colorindo o mapa

instalar o D3

```bash
npm install -g d3
```

Então vamos colorir de maneira aleatória o mapa

```bash
ndjson-map -r d3 \
  '(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 100])(d.properties.light), d)' \
  < rj-ortho-light.ndjson \
  > rj-ortho-color.ndjson
```

E vamos printá-lo num SVG

```bash
geo2svg -n --stroke none -w 1000 -h 600 \
  < rj-ortho-color.ndjson \
  > rj-ortho-color.svg
```

Instalar o TopoJSON

```bash
npm install -g topojson
```

Unificando os elementos em um topo

```bash
geo2topo -n \
  tracts=rj-ortho-light.ndjson \
  > rj-tracts-topo.json
```

Simplificando a geometria

```bash
toposimplify -p 1 -f \
  < rj-tracts-topo.json \
  > rj-simple-topo.json
```

Quantificando a geometria

```bash
topoquantize 1e5 \
  < rj-simple-topo.json \
  > rj-quantized-topo.json
```

Instalar a escala cromática do D3

```bash
npm install -g d3-scale-chromatic
```

E gera o JSON com os recortes

```bash
topo2geo tracts=- \
  < rj-simple-topo.json \
  | ndjson-map -r d3 -r d3=d3-scale-chromatic 'z = d3.scaleThreshold().domain([0, 10, 20, 30, 40, 50, 60, 70, 80, 90, 100]).range(d3.schemeYlOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.light)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -w 1000 -h 600 \
  > rj-tracts-threshold-light.svg
  ```
