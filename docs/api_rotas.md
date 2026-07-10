# Roteamento e APIs da Plataforma

A aplicação é fragmentada em rotas lógicas que atendem diferentes perfis de sistemas e usuários, incluindo suporte nativo para cadernos de ciência de dados.

## 1. Endpoints Principais (OpenShift Routes)
* **Catálogo e Front-end:** `https://geonode.ana.gov.br/`
* **Motor OGC (GeoServer):** `https://geonode.ana.gov.br/geoserver/`
* **Business Intelligence (Superset):** `https://bi-espacial.ana.gov.br/`

## 2. APIs de Interoperabilidade OGC
Os sistemas clientes (QGIS, Dashboards) devem consumir as camadas utilizando os padrões abertos WMS, WFS e WCS.

## 3. Integração com JupyterHub e Databricks (Data Science)
A infraestrutura está configurada para atuar como provedora de dados diretos para scripts Python (Pandas/GeoPandas), eliminando a necessidade de baixar Shapefiles.

**Exemplo de consumo via JupyterHub / Azure Databricks:**
As origens estão liberadas no CORS da aplicação. O cientista de dados pode instanciar conexões diretas via OWSLib:

```python
import geopandas as gpd
from owslib.wfs import WebFeatureService

# Conexão transparente via WFS
wfs_url = "[https://geonode.ana.gov.br/geoserver/wfs](https://geonode.ana.gov.br/geoserver/wfs)"
wfs = WebFeatureService(url=wfs_url, version='2.0.0')

# Extração de polígonos/pontos processados
layer_id = 'ana:bacias_hidrograficas_processadas'
gdf = gpd.read_file(f"{wfs_url}?service=WFS&version=2.0.0&request=GetFeature&typeName={layer_id}&outputFormat=application/json")
