# Diretrizes Estratégicas e Decisões Arquiteturais (ADR)

Este documento registra as principais "orientações" e decisões de design de infraestrutura adotadas pela Coordenação de Infraestrutura e Operações (COOPI) na migração do ArcGIS Enterprise para o ecossistema GeoNode Enterprise no Red Hat OpenShift.

O objetivo é garantir que a plataforma permaneça escalável, segura e sustentável a longo prazo, evitando armadilhas comuns em migrações de sistemas de Informação Geográfica (GIS).

---

## 1. O Padrão "GitOps Wrapper" (Não ao Hard Fork)

* **O Desafio:** Ao migrar para o open-source (GeoNode/Django/GeoServer), existe a tentação de fazer um *fork* do código-fonte original para customizar a plataforma para a Agência.
* **A Decisão:** Foi terminantemente proibido o *hard fork* da aplicação. A plataforma adota o padrão **"GitOps Wrapper"** (Envelopamento).
* **A Justificativa:** Manter código de terceiros gera o problema de *Upstream Drift*. Quando a comunidade lançar patches de segurança, a Agência teria extrema dificuldade em atualizar. Ao invés disso, o nosso repositório contém apenas a **Infraestrutura como Código (Helm/Kustomize)** e Dockerfiles que estendem a imagem oficial. A lógica core do GeoNode é intocável, garantindo atualizações limpas e governança via **ArgoCD**.

## 2. Arquitetura 100% Stateless no OpenShift

* **O Desafio:** O ArcGIS no Kubernetes exige volumes de disco persistentes (PVCs - ReadWriteMany) mapeados dentro do cluster para manter bancos de dados e arquivos compartilhados, o que gera alta complexidade de storage e risco de corrupção.
* **A Decisão:** O cluster OpenShift foi transformado em um ambiente de pura **Computação (Stateless)**. Todo o Estado (State) da aplicação foi externalizado.
* **A Justificativa:** * Os bancos de dados e configurações do GeoServer foram movidos para o **AWS RDS (PostgreSQL/PostGIS)** via plugin JDBC.
  * Os uploads, mídias e imagens pesadas (COGs) foram movidos para o **AWS S3**.
  * Se os nós do OpenShift caírem, os pods podem ser recriados instantaneamente em qualquer outro nó ou provedor, sem perda de um único byte de dado geográfico.

## 3. Topologia Multicloud e Data Science (Integração vs. Hospedagem)

* **O Desafio:** Suportar o processamento pesado de IA (como a PoC do projeto "Declara Água") e a análise de ciência de dados dos hidrólogos (JupyterHub) sem derrubar o portal de mapas.
* **A Decisão:** Não provisionar o Databricks ou o JupyterHub como *Pods* dentro do OpenShift do GeoNode. Em vez disso, adotou-se a **Integração Multicloud via CORS e APIs**.
* **A Justificativa:** Cada ferramenta deve rodar em seu ambiente nativo e ideal. O Azure Databricks processa a visão computacional em sua própria VPC; o JupyterHub roda na infraestrutura interna da ANA. O GeoNode atua apenas como o hub central, liberando acesso a essas origens via CORS e entregando os dados processados pelo Databricks (gravados no RDS) para os cientistas de dados (WFS) de forma fluída e assíncrona.

## 4. Governança de Identidade Cloud-Native (SSO/OIDC)

* **O Desafio:** Gerenciar quem tem acesso aos painéis estratégicos (FinOps-Gov) e edição de camadas, sem criar silos de senhas ou depender de conexões frágeis com o Active Directory legado.
* **A Decisão:** Transição para provedores de identidade modernos utilizando **Single Sign-On (SAML 2.0 / OIDC)**, com o GeoNode atuando como cliente ou provedor OIDC.
* **A Justificativa:** Elimina o tráfego de senhas em texto puro pela rede. Usuários autenticam-se nas plataformas corporativas (ex: Microsoft Entra ID / Gov.br) utilizando múltiplos fatores de autenticação (MFA). O acesso ao JupyterHub e ao Superset é integrado, promovendo uma arquitetura de *Zero Trust*.

## 5. Padrões Abertos (OGC) como Regra de Interoperabilidade

* **O Desafio:** Sistemas satélites da agência e scripts de coleta acostumados a consumir a REST API proprietária da Esri.
* **A Decisão:** Adoção estrita e exclusiva dos protocolos do **Open Geospatial Consortium (OGC)**.
* **A Justificativa:** Todo consumo de imagem passa por WMS/WMTS, e toda manipulação vetorial e extração para machine learning passa por WFS paginado. Isso garante que a plataforma da ANA possa interoperar nativamente com qualquer outro órgão do governo federal (como o Brazil Data Cube do INPE) sem necessidade de desenvolvimento de *middlewares* ou tradutores de API.
