# Arquitetura de Rede e Portas de Comunicação

A segurança e eficiência da plataforma dependem de uma segregação estrita de tráfego. A comunicação é dividida em zonas de confiança, agora englobando a estratégia multicloud.

## 1. Tráfego Externo (Ingress)
A porta de entrada da aplicação é protegida por um Application Load Balancer e pelas rotas nativas do OpenShift.
* **Porta 443 (HTTPS):** Única porta exposta para a rede corporativa e VPN. Recebe todo o tráfego web, consumo de APIs OGC (WMS/WFS) e acessos ao Superset. A terminação TLS/SSL ocorre na borda (Edge Termination).

## 2. Comunicação Interna (OpenShift Cluster / VPC)
O tráfego Leste-Oeste (entre contêineres e bancos de dados) é isolado em sub-redes privadas.
* **Portas 8000/8080 (HTTP):** Tráfego interno roteado para os pods do Django e GeoServer.
* **Porta 6379 (TCP):** Comunicação do Celery com o Redis.
* **Porta 5432 (TCP):** Comunicação exclusiva dos nós do OpenShift para o AWS RDS.

## 3. Tráfego de Especialistas (Acesso Direto)
* **Porta 5432 (TCP - via VPN restrita):** Liberada via AWS Security Group para a sub-rede da engenharia utilizar o QGIS Desktop conectado ao AWS RDS.

## 4. Integrações Outbound (Egress)
* **AWS S3 API (HTTPS 443):** Os pods leem/gravam nos buckets via IAM Roles (IRSA).
* **INPE / Brazil Data Cube:** O GeoServer realiza *WMS Cascading* via requisições HTTPS.

## 5. Integração Multicloud e Data Science (CORS)
A plataforma permite chamadas de API cross-domain (CORS) estritamente controladas para apoiar os pipelines analíticos.
* **JupyterHub (Rede Interna):** Tráfego liberado da origem `https://jupyter.ana.gov.br`.
* **Azure Databricks:** Tráfego liberado da origem `https://adb-3109412849216067.7.azuredatabricks.net` para submissão de cargas de IA.
