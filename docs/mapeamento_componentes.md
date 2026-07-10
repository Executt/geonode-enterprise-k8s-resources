# Mapeamento de Componentes: ArcGIS Enterprise vs. Ecossistema Cloud-Native (GeoNode)

Este documento estabelece o paralelo técnico definitivo entre a stack proprietária da Esri (ArcGIS Enterprise) e a nova arquitetura de Infraestrutura de Dados Espaciais (IDE) adotada pela Coordenação de Infraestrutura e Operações (COOPI).

A transição não representa apenas uma troca de software, mas uma mudança de paradigma: de um modelo **Monolítico e Stateful** (onde a aplicação retém e gerencia os discos e bancos de dados) para um modelo **Distribuído, Stateless e Multicloud**, focado em serviços gerenciados (AWS) e orquestração ágil (OpenShift).

---

## 1. Camada Core: Portal, Servidor de Mapas e Metadados

Esta é a espinha dorsal do sistema, responsável pela gestão de usuários, renderização cartográfica e disponibilização de catálogos.

| Componente ArcGIS Enterprise | Equivalente Open Source (IDE) | Função na Nova Arquitetura |
| :--- | :--- | :--- |
| **Portal for ArcGIS** | **GeoNode (Django/Python)** | Interface web principal, CMS, gestão de permissões de camadas, perfis de usuários e catálogo interativo. |
| **ArcGIS Server / Map Server** | **GeoServer (Java/Spring)** | Motor de renderização pesado. Recebe os dados brutos e os transforma em serviços OGC padronizados (WMS, WFS, WCS). |
| **Geoportal Server** | **pycsw** | Catálogo de metadados embutido no GeoNode, garantindo interoperabilidade no padrão CSW. |
| **ArcGIS Web Adaptor** | **AWS ALB + OpenShift Route** | O proxy reverso, terminação TLS/SSL (Edge) e balanceamento de carga de entrada para a rede corporativa. |

---

## 2. Camada de Dados e Armazenamento (Storage)

O maior ganho de resiliência ocorre aqui. O estado da aplicação foi externalizado do cluster Kubernetes para serviços gerenciados da AWS.

| Componente ArcGIS Enterprise | Equivalente Open Source (IDE) | Função na Nova Arquitetura |
| :--- | :--- | :--- |
| **ArcGIS Relational Data Store** | **AWS RDS (PostgreSQL)** | Armazena o banco de configuração interno do GeoNode (tabelas do Django) e do GeoServer (via plugin JDBC). |
| **ArcGIS Spatial Data Store (SDE)**| **AWS RDS (PostGIS)** | Armazenamento de alta performance para os dados vetoriais consumidos ativamente nas análises e mapas. |
| **ArcGIS Image Server / Raster Store**| **AWS S3 + Cloud Optimized GeoTIFF (COG)**| Hospedagem de imagens de satélite e modelos digitais de elevação. O GeoServer consome apenas os fragmentos necessários do bucket AWS S3 via requisições HTTP (Stateless). |
| **Spatiotemporal Big Data Store** | **Externalizado / Não aplicável** | Substituído por consultas diretas ao **AWS RDS** otimizadas ou por ingestão via **Databricks**. |

---

## 3. Camada de Aplicações e Visualização (Analytics & BI)

Onde os dados se transformam em painéis de decisão para a diretoria e analistas finalísticos.

| Componente ArcGIS Enterprise | Equivalente Open Source (IDE) | Função na Nova Arquitetura |
| :--- | :--- | :--- |
| **ArcGIS Dashboards** | **Apache Superset** | Plataforma de BI avançada. Conecta-se ao RDS para cruzar milhões de linhas (ex: dados do FinOps-Gov e Declara Água) sem onerar a engine de mapas. |
| **ArcGIS Experience Builder** | **Superset (via iFrame) + MapStore** | Criação de portais e painéis interativos híbridos (gráficos + mapas) incorporados nas páginas do GeoNode. |
| **ArcGIS StoryMaps** | **MapStore GeoStories** | Ferramenta nativa do GeoNode para construção de narrativas imersivas guiadas por mapas e mídias. |
| **ArcGIS Web Map Viewer** | **MapStore Dashboards** | Interface React embutida no GeoNode para composição e estilização rápida de mapas web pelos analistas. |

---

## 4. Camada de Ciência de Dados e Inteligência Artificial

A separação do processamento pesado da camada de publicação web, garantindo performance escalável.

| Componente ArcGIS Enterprise | Equivalente Open Source (IDE) | Função na Nova Arquitetura |
| :--- | :--- | :--- |
| **ArcGIS Notebook Server** | **JupyterHub** | Infraestrutura provisionada internamente para que hidrólogos desenvolvam em Python (Pandas/GeoPandas), consumindo dados da IDE via APIs WFS protegidas por CORS. |
| **ArcGIS GeoAnalytics Server** | **Azure Databricks** | Serviço PaaS para processamento de Machine Learning (ex: leitura de hidrômetros por visão computacional). Processa volumes massivos de dados no Azure e grava os resultados no AWS RDS. |

---

## 5. Camada de Operações em Campo (Field Ops)

Substituição do licenciamento por usuário em campo por alternativas de código aberto robustas e offline-first.

| Componente ArcGIS Enterprise | Equivalente Open Source (IDE) | Função na Nova Arquitetura |
| :--- | :--- | :--- |
| **ArcGIS Survey123** | **ODK (Open Data Kit)** | Formulários inteligentes para coleta de dados tabulares e coordenadas em campo. |
| **ArcGIS Field Maps** | **QField / QFieldCloud** | Aplicativo mobile (Android/iOS) que espelha os projetos do QGIS. Permite trabalho offline com sincronização posterior direta no banco PostGIS via VPN/HTTPS. |

---

## 6. Camada de Governança, Segurança e Infraestrutura (DevSecOps)

Mecanismos de orquestração e gestão de ciclo de vida da plataforma.

| Componente ArcGIS Enterprise | Equivalente Open Source (IDE) | Função na Nova Arquitetura |
| :--- | :--- | :--- |
| **Portal Identity (Built-in)** | **SSO (OIDC / OAuth2)** | Delegação da autenticação para provedores modernos (Microsoft Entra ID/Gov.br) ou Active Directory local (LDAP), garantindo Single Sign-On com o JupyterHub. |
| **ArcGIS Enterprise Operator (K8s)**| **ArgoCD (GitOps)** | Agente orquestrador que monitora o repositório Git e aplica as mudanças de infraestrutura automaticamente no cluster OpenShift. |
| **N/A (Manutenção Manual)** | **GitLab CI + RHACS + Quay** | Esteira automatizada de DevSecOps. O código é analisado pelo SonarQube, as imagens armazenadas no Red Hat Quay e o StackRox (RHACS) bloqueia deploys com vulnerabilidades (CVEs). |

---

> **Nota Técnica de Implementação:** > A principal vantagem desta transição reside no isolamento de domínios. Se o cluster OpenShift falhar ou for recriado do zero, **nenhum dado geográfico, mapa ou configuração de usuário é perdido**, pois o estado reside integralmente na AWS (RDS e S3) e os manifestos residem no repositório Git. Essa é a essência de uma plataforma em nuvem para missões críticas.
