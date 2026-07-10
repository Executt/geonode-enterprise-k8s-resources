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
