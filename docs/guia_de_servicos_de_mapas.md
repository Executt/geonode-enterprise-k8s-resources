# Guia de Transição de Serviços de Mapas: REST API (ArcGIS) para Padrões Abertos (OGC)

A modernização da nossa Infraestrutura de Dados Espaciais (IDE) para o ecossistema GeoNode/GeoServer altera fundamentalmente a forma como as aplicações interagem com os mapas. A arquitetura anterior dependia de protocolos proprietários da Esri (ArcGIS REST API). A nova arquitetura adota estritamente os padrões abertos do OGC (Open Geospatial Consortium).

Este documento instrucional detalha os impactos dessa mudança e orienta as equipes de desenvolvimento e engenharia sobre como ajustar suas aplicações, scripts e rotinas de análise para o novo padrão.

---

## 1. Mapeamento de Serviços (De / Para)

Sistemas clientes (dashboards analíticos, scripts em Python ou aplicações web customizadas) que consumiam a API REST da Esri precisarão ter seus endpoints e parâmetros de requisição reescritos.

| Padrão Anterior (ArcGIS REST API) | Equivalente Atual (OGC / GeoServer) | Finalidade e Casos de Uso |
| :--- | :--- | :--- |
| **MapServer** (`/MapServer`) | **WMS** (Web Map Service) | Retorna uma **imagem renderizada** (PNG/JPEG) do mapa. Ideal para visualização leve em sistemas web, portais e iFrames no Superset. |
| **FeatureServer** (`/FeatureServer`) | **WFS** (Web Feature Service) | Retorna o **dado vetorial bruto** (GeoJSON/GML). Essencial para extração de coordenadas, edições de geometrias e ingestão de dados de hidrômetros em rotinas de IA. |
| **ImageServer** (`/ImageServer`) | **WCS** (Web Coverage Service) | Retorna **dados matriciais brutos** (bandas de satélite, MDEs). Usado para análises temporais pesadas e álgebra de mapas de bacias hidrográficas. |
| **TileServer** / Cached Map Service | **WMTS** (Web Map Tile Service) | Retorna **imagens pré-cacheadas** em blocos (Tiles). Obrigatório para camadas de altíssimo acesso que não sofrem mudanças diárias, aliviando a CPU do cluster. |

---

## 2. Instruções Técnicas por Perfil de Uso

A alteração do endpoint afeta de forma diferente quem programa, quem consome dados em lote e quem analisa mapas visualmente.

### A. Para Desenvolvedores Web (Front-end e Dashboards)

As aplicações construídas com a *ArcGIS API for JavaScript* precisarão ser migradas para bibliotecas open-source como **OpenLayers**, **Leaflet** ou **MapStore/React**.

Ao invés de carregar um `FeatureLayer` apontando para um serviço REST, o código passará a solicitar parâmetros WMS ao servidor.

**Exemplo de alteração em uma aplicação web:**

```javascript
// ANTIGO (ArcGIS REST)
const url = "[https://gis.antigo.gov.br/server/rest/services/hidrologia/MapServer](https://gis.antigo.gov.br/server/rest/services/hidrologia/MapServer)";

// NOVO (Padrão WMS OGC)
const wmsUrl = "[https://geonode.ana.gov.br/geoserver/ows](https://geonode.ana.gov.br/geoserver/ows)?";
const parametros = new URLSearchParams({
    service: 'WMS',
    version: '1.3.0',
    request: 'GetMap',
    layers: 'ana:bacias_hidrograficas',
    format: 'image/png',
    crs: 'EPSG:4326',
    bbox: '-33.7,-73.9,5.2,-34.7',
    width: '800',
    height: '600'
});

**Atenção (Performance): Evite carregar grandes volumes de dados como WFS (GeoJSON) direto no navegador do usuário. Se o objetivo é apenas visualizar a camada, utilize WMS ou WMTS. Reserve o WFS apenas para quando o usuário clicar no mapa para ver atributos específicos (GetFeatureInfo) ou precisar editar um ponto de outorga.**

### B. Para Cientistas de Dados (JupyterHub e Databricks)
Os cadernos analíticos que faziam chamadas via arcgis.gis Python API ou lidavam com as paginações customizadas da REST API deverão migrar para bibliotecas padrão do ecossistema espacial Python, como o OWSLib e o GeoPandas.

A grande vantagem é a padronização: o código escrito para consumir dados da IDE da agência funcionará da mesma forma para ler dados do INPE (Brazil Data Cube) ou de qualquer outro órgão governamental do mundo que siga o OGC.

**Exemplo de consumo automatizado de feições:**

´´´import geopandas as gpd

# URL base do serviço WFS protegido
wfs_url = "[https://geonode.ana.gov.br/geoserver/wfs](https://geonode.ana.gov.br/geoserver/wfs)"

# Requisição parametrizada retornando GeoJSON nativo
parametros = {
    "service": "WFS",
    "version": "2.0.0",
    "request": "GetFeature",
    "typeName": "ana:pontos_monitoramento",
    "outputFormat": "application/json",
    # Boas práticas: Paginação e filtros espaciais reduzem tráfego na rede
    "count": 5000, 
    "startIndex": 0
}

url_completa = f"{wfs_url}?" + "&".join(f"{k}={v}" for k, v in parametros.items())

# Carrega direto para um GeoDataFrame
gdf = gpd.read_file(url_completa)
print(gdf.head())
);

### C. Para Analistas GIS (QGIS Desktop)
A transição no QGIS é a mais transparente, pois o software é construído em torno dos padrões OGC. A instrução primária é descontinuar o uso das antigas conexões "ArcGIS REST Server" no painel do QGIS.
Para adicionar mapas de fundo, utilize o menu Camada > Gerenciador de Fonte de Dados > WMS/WMTS.
Para editar dados diretamente no banco a partir do QGIS (quando a conexão nativa com o banco RDS PostGIS não estiver disponível via VPN), utilize o menu WFS / OGC API - Features.

Ponto Crítico: Nas configurações de qualquer conexão WFS no QGIS, marque a opção "Ativar paginação de feições" e defina um limite seguro (ex: 2000). Tentar baixar 1 milhão de pontos de uma vez sem paginação causará um erro de Timeout no Application Load Balancer da AWS e o travamento do QGIS.

## 3. Segurança e Interação com a API REST do GeoNode
É importante não confundir a "ArcGIS REST API" com a API REST do GeoNode.

O GeoNode possui sua própria API REST (acessível em /api/v2/), mas ela não serve mapas. A API do GeoNode serve para administrar a plataforma:
> Automatizar o upload de shapefiles.
> Atualizar a tabela de metadados.
> Modificar os direitos de acesso e grupos de usuários.
> Integrar painéis administrativos (como módulos do FinOps-Gov) para ler estatísticas de acesso e volume de camadas.

Para toda e qualquer manipulação de coordenadas espaciais ou renderização cartográfica, a comunicação com o cluster passará invariavelmente pelo tráfego de serviços OGC orquestrados no GeoServer.
