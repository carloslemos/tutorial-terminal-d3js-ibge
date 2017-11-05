# tutorial-terminal-d3js-ibge
Tutorial de como fazer mapas apenas com a linha de comando em D3js com os dados do IBGE

## Convertendo a malha censitária em um mapa
Pegue os dados relativos a malha censitária do Rio de Janeiro [nest link](ftp://geoftp.ibge.gov.br/organizacao_do_territorio/malhas_territoriais/malhas_de_setores_censitarios__divisoes_intramunicipais/censo_2010/setores_censitarios_shp/sp/sp_setores_censitarios.zip). Como não queremos sair do terminal, vamos usar o __curl__ para isso.  

```terminal
curl \
  'ftp://geoftp.ibge.gov.br/organizacao_do_territorio/malhas_territoriais/malhas_de_setores_censitarios__divisoes_intramunicipais/censo_2010/setores_censitarios_shp/rj/rj_setores_censitarios.zip' \
  -o rj_setores_censitarios.zip
```

Depois é dezipar a pasta

```terminal
unzip -o rj_setores_censitarios.zip
```

Vamos instalar o shapefile do nosso querido amigo Mike Bostock. Ele precisa de Node e do NPM, então se você estiver usando um Mac eu recomendo o Homebrew. Do contrário, podemos ir com o instalador da página deles mesmo.

```terminal
npm install -g shapefile
```

Vamos convertero SHP para GeoJSON

```terminal
shp2json 33SEE250GC_SIR.shp --encoding 'utf8' -o rj.json
```

Vamos instalar as projeções do D3

```terminal
npm install -g d3-geo-projection
```

E aplicar a projeção ortográfica ao estado do Rio de Janeiro

```terminal
geoproject \
  'd3.geoOrthographic().rotate([42.5, 22.5, 0]).fitSize([1000, 600], d)' \
  < rj.json \
  > rj-orthographic.json
```

Para finalizar, vamos converter a projeção em SVG

```terminal
geo2svg \
  -w 1000 \
  -h 600 \
  < rj-orthographic.json \
  > rj-orthographic.svg
```

## Unindo os dados censitários a malha

```terminal
npm install -g ndjson-cli
```

E vamos separar em um ndjson o mapa projetado

```terminal
ndjson-split 'd.features' \
  < rj-orthographic.json \
  > rj-orthographic.ndjson
```

Depois vamos mapear os códigos dos setores para bater com a coluna do CSV

```terminal
ndjson-map 'd.Cod_setor = d.properties.CD_GEOCODI, d' \
  < rj-orthographic.ndjson \
  > rj-orthographic-sector.ndjson
```

Vamos baixar todos os dados da amostra geral do Rio de Janeiro

```terminal
curl \
  'ftp.ibge.gov.br/Censos/Censo_Demografico_2010/Resultados_do_Universo/Agregados_por_Setores_Censitarios/RJ_20171016.zip' \
  -o RJ_20171016.zip
```

Dezipar os dados

```terminal
unzip -o RJ_20171016.zip
```

Depois vamos instalar o módulo de conversão de CSVs, TSVs e afins

```terminal
npm install -g d3-dsv
```

Converteremos o CSV do censo para ndjson

```terminal
dsv2json \
  -r ';' \
  -n \
  < RJ/Base\ informa\%E7oes\ setores2010\ universo\ RJ/CSV/Pessoa03_RJ.csv \
  > rj-census.ndjson
```

E uniremos o arquivo de dados do censo com a projeção ortográfica

```terminal
ndjson-join 'd.Cod_setor' \
	rj-orthographic-sector.ndjson \
	rj-census.ndjson \
	> rj-orthographic-census.ndjson
```

E descobriremos a % da população que não se considera branca

```terminal
ndjson-map \
  'd[0].properties = {black: Math.floor(1000 * Number(d[1].V003) / d[1].V001)}, d[0]' \
  < rj-orthographic-census.ndjson \
  > rj-orthographic-black.ndjson
```

## Colorindo o mapa

instalar o D3

```terminal
npm install -g d3
```

Então vamos colorir de maneira aleatória o mapa

```terminal
ndjson-map -r d3 \
  '(d.properties.fill = d3.scaleSequential(d3.interpolateViridis).domain([0, 1000])(d.properties.black), d)' \
  < rj-orthographic-black.ndjson \
  > rj-orthographic-color.ndjson
```

E vamos printá-lo num SVG

```terminal
geo2svg -n --stroke none -w 1000 -h 600 \
  < rj-orthographic-color.ndjson \
  > rj-orthographic-color.svg
```

Instalar o TopoJSON

```terminal
npm install -g topojson
```

Unificando os elementos em um topo

```terminal
geo2topo -n \
  tracts=rj-orthographic-black.ndjson \
  > rj-tracts-topo.json
```

Simplificando a geometria

```terminal
toposimplify -p 1 -f \
  < rj-tracts-topo.json \
  > rj-simple-topo.json
```

Quantificando a geometria

```terminal
topoquantize 1e5 \
  < rj-simple-topo.json \
  > rj-quantized-topo.json
```

Instalar a escala cromática do D3

```terminal
npm install -g d3-scale-chromatic
```

E gera o JSON com os recortes

```terminal
topo2geo tracts=- \
  < rj-tracts-topo.json \
  | ndjson-map -r d3 -r d3=d3-scale-chromatic 'z = d3.scaleThreshold().domain([0, 100, 200, 300, 400, 500, 600, 700, 800, 900, 1000]).range(d3.schemeYlOrRd[9]), d.features.forEach(f => f.properties.fill = z(f.properties.black)), d' \
  | ndjson-split 'd.features' \
  | geo2svg -n --stroke none -w 1000 -h 600 \
  > rj-tracts-threshold.svg
  ```
