# Roteamento e APIs da Plataforma

A aplicação é fragmentada em rotas lógicas que atendem diferentes perfis de sistemas e usuários.

## 1. Endpoints Principais (OpenShift Routes)
* **Catálogo e Front-end:** `https://geonode.ana.gov.br/` (Acesso principal para busca de metadados e visualização de mapas).
* **Motor OGC (GeoServer):** `https://geonode.ana.gov.br/geoserver/` (Acesso programático para consumo de serviços espaciais).
* **Business Intelligence (Superset):** `https://bi-espacial.ana.gov.br/` (Acesso ao painel analítico avançado de dados).

## 2. APIs de Interoperabilidade OGC
Os sistemas clientes devem consumir as camadas geográficas utilizando os padrões abertos:
* **WMS (Web Map Service):** Para renderização de mapas em aplicações web/dashboards.
  * *Endpoint:* `/geoserver/ows?service=WMS&request=GetCapabilities`
* **WFS (Web Feature Service):** Para extração e edição de feições vetoriais reais (pontos, linhas, polígonos). Habilitar sempre a paginação para evitar *timeouts*.
  * *Endpoint:* `/geoserver/ows?service=WFS&request=GetCapabilities`
* **WCS (Web Coverage Service):** Para consumo de dados raster brutos (Modelos Digitais de Elevação, Imagens COG).

## 3. API REST do GeoNode
Utilizada para integrações de backend corporativo (ex: robôs de carga de dados).
* *Endpoint v2:* `/api/v2/`
* *Autenticação:* Bearer Token OAuth2 gerado via interface administrativa.
