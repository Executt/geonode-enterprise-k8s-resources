# Arquitetura de Rede e Portas de Comunicação

A segurança e eficiência da plataforma dependem de uma segregação estrita de tráfego. A comunicação é dividida em três zonas principais de confiança.

## 1. Tráfego Externo (Ingress)
A porta de entrada da aplicação é protegida por um Application Load Balancer e pelas rotas nativas do OpenShift.

* **Porta 443 (HTTPS):** Única porta exposta para a rede corporativa e VPN. Recebe todo o tráfego web, consumo de APIs OGC (WMS/WFS) e acessos ao Superset. A terminação TLS/SSL ocorre na borda (Edge Termination).

## 2. Comunicação Interna (OpenShift Cluster / VPC)
O tráfego Leste-Oeste (entre contêineres e bancos de dados) é isolado em sub-redes privadas.

* **Porta 8000 (HTTP):** Tráfego interno roteado para o pod do `geonode-django` (Catálogo e UI).
* **Porta 8080 (HTTP):** Tráfego interno roteado para o pod do `geonode-geoserver` (Motor de Mapas OGC).
* **Porta 6379 (TCP):** Comunicação dos workers assíncronos (Celery) com o broker **Redis** no cluster.
* **Porta 5432 (TCP):** Comunicação exclusiva de saída dos nós do OpenShift para a instância gerenciada do **AWS RDS (PostgreSQL/PostGIS)**.

## 3. Tráfego de Especialistas (Acesso Direto)
* **Porta 5432 (TCP - via VPN restrita):** Liberada via AWS Security Group exclusivamente para a sub-rede da engenharia/analistas GIS. Permite que o QGIS Desktop faça consultas pesadas diretamente no AWS RDS, sem onerar a camada web do GeoServer.

## 4. Integrações Outbound (Egress)
* **AWS S3 API (HTTPS 443):** Os pods utilizam perfis de acesso (IAM) para gravar uploads e ler imagens Cloud Optimized GeoTIFF (COG) sem baixar os arquivos localmente.
* **INPE / Brazil Data Cube:** O GeoServer realiza requisições HTTPS de saída para realizar o *WMS Cascading* de matrizes temporais.
