# Equivalência e Implementação de APIs: Do ArcGIS REST para o Ecossistema GeoNode

Na arquitetura anterior, a ArcGIS REST API centralizava absolutamente todas as operações — desde a consulta a um polígono de bacia hidrográfica até o gerenciamento de senhas de usuários e configurações do servidor.

A arquitetura Cloud-Native baseada no GeoNode e orquestrada no OpenShift adota um paradigma modular de microsserviços. Em vez de um monólito, a plataforma divide as responsabilidades em APIs distintas, focadas em padrões abertos (OGC) e RESTful. Este documento detalha a visão de negócios, os conceitos técnicos e as instruções de configuração para suportar as integrações corporativas.

---

## 1. O Conceito de Negócios: Por que Desacoplar?

A modularização das APIs garante que a plataforma atue como o "coração" dos dados geoespaciais, viabilizando a transformação digital de processos sem engessar as ferramentas das áreas de negócio.

* **Automação e Escalabilidade:** Em projetos como o Declara Água, algoritmos de visão computacional (processados no Azure Databricks) não precisam interagir com interfaces gráficas. Eles inserem telemetria massiva direto no banco via API.
* **Gestão Financeira e Administrativa:** Sistemas como o FinOps-Gov ou Pacto-Gov podem consultar a API do GeoNode para extrair a volumetria de dados hospedados no AWS S3, cruzando esses dados com o *billing* da nuvem.
* **Democratização dos Dados:** Cientistas de dados utilizando o JupyterHub na rede interna consomem polígonos via APIs OGC de forma padronizada, sem depender de bibliotecas proprietárias da Esri.

---

## 2. O Conceito Técnico: O Mapeamento das APIs

A nova arquitetura divide a "ArcGIS REST API" em três pilares fundamentais.

### Pilar 1: OGC API - Features (Substitui o `/FeatureServer/query`)
* **Função:** É o padrão moderno do Open Geospatial Consortium (evolução do WFS 3.0) para desenvolvedores web e análise de dados. Retorna geometrias em formato GeoJSON puro através de chamadas OpenAPI (Swagger).
* **Caso de Uso:** Um dashboard construído em React ou um script Python (GeoPandas) solicitando a lista de pontos de monitoramento ativos.
* **Endpoint Padrão:** `GET https://geonode.ana.gov.br/geoserver/ogc/features/collections/ana:pontos_monitoramento/items`

### Pilar 2: GeoNode REST API v2 (Substitui o ArcGIS Portal API)
* **Função:** API construída no Django REST Framework para a administração do catálogo. Não renderiza mapas, mas gerencia os metadados, uploads de *shapefiles*, permissões granulares e gestão de grupos.
* **Caso de Uso:** Um robô administrativo atualizando diariamente a descrição de uma camada de recursos hídricos ou transferindo a propriedade de um mapa de um analista para outro.
* **Endpoint Padrão:** `GET / POST https://geonode.ana.gov.br/api/v2/`

### Pilar 3: GeoServer REST API (Substitui o ArcGIS Server Admin API)
* **Função:** Interface programática exclusiva para gerenciar o motor cartográfico. Permite criar workspaces, registrar bancos de dados PostGIS e aplicar estilos cartográficos (SLD).
* **Caso de Uso:** Uma esteira de CI/CD (GitLab) cria uma tabela nova no AWS RDS e, na sequência, faz um POST para o GeoServer publicar essa tabela automaticamente como um mapa WMS.
* **Endpoint Padrão:** `GET / POST https://geonode.ana.gov.br/geoserver/rest/`

---

## 3. Implementação e Configuração na Arquitetura GitOps

Para que essa engrenagem de APIs funcione de forma segura e fluída através do ArgoCD e do OpenShift, três configurações essenciais devem estar implementadas na infraestrutura.

### A. Liberação de CORS (Cross-Origin Resource Sharing)
Aplicações externas (como o Apache Superset, JupyterHub ou Databricks) serão bloqueadas pelo navegador se tentarem consultar a API REST sem a devida liberação.

**Como configurar:** No repositório de infraestrutura (`values-prod.yaml`), as variáveis de ambiente do pod do Django (`geonode-django`) devem conter as origens confiáveis.

```yaml
# Configuração no helm-chart/values-prod.yaml
django:
  env:
    - name: CORS_ALLOW_ALL_ORIGINS
      value: "False" # Mantém o Zero Trust, bloqueando domínios não autorizados
    - name: CORS_ALLOWED_ORIGINS
      value: "[https://jupyter.ana.gov.br](https://jupyter.ana.gov.br),[https://bi-espacial.ana.gov.br](https://bi-espacial.ana.gov.br),[https://xxxxxxxxxxxxxxx.7.azuredatabricks.net](https://xxxxxxxxxxxx.7.azuredatabricks.net)"

### B. Autenticação e Autorização (OAuth2)
Assim como a ArcGIS REST API gerava um token para sessões seguras, a API do GeoNode utiliza Bearer Tokens OAuth2. O GeoNode está configurado como um Identity Provider (OIDC Provider).

Como implementar na prática:

O administrador acessa o painel Django (https://geonode.ana.gov.br/admin/oauth2_provider/application/).

Cria uma nova aplicação do tipo Confidential e Client credentials.

A plataforma gerará um Client ID e um Client Secret.

Sistemas clientes utilizam essas credenciais para obter o token transacional:

Bash
# Exemplo de requisição do Token para consumo da API
curl -X POST -d "grant_type=client_credentials" \
     -u "SEU_CLIENT_ID:SEU_CLIENT_SECRET" \
     [https://geonode.ana.gov.br/o/token/](https://geonode.ana.gov.br/o/token/)
C. Segurança no GeoServer REST API (Bloqueio de Exposição)
A GeoServer REST API é extremamente poderosa; quem tem acesso a ela pode apagar camadas ou visualizar a string de conexão do AWS RDS.

Como configurar: No OpenShift, a rota de entrada principal (Route) não deve expor o caminho /geoserver/rest para a internet pública.

Regra de Engenharia: A API administrativa do GeoServer deve ser acessada apenas por instâncias de processamento dentro da própria VPC (ex: esteiras do GitLab CI) ou através da VPN corporativa restrita para os administradores do sistema (equipe COOPI). As configurações de Ingress Controller ou Application Load Balancer (ALB) da AWS devem rejeitar (403 Forbidden) requisições externas para este sufixo.

## 4. Guia Rápido de Migração para Desenvolvedores
Se você está reescrevendo um script Python ou uma rotina de integração de um sistema legado, observe os mapeamentos abaixo:

Precisa baixar a geometria de todas as bacias de uma região?

Antigo: ArcGIS REST -> /query?where=1=1

Novo: Utilize a OGC API - Features (/geoserver/ogc/features/...) e trabalhe diretamente com o GeoJSON retornado.

Precisa cadastrar um PDF técnico associado a uma camada?

Antigo: Upload manual no Portal for ArcGIS.

Novo: Utilize a GeoNode REST API v2 (/api/v2/documents/) enviando o arquivo via requisição POST com seu token OAuth2. O GeoNode salvará o arquivo diretamente no AWS S3.

Precisa mudar a cor de uma camada (Renderização/Simbolização)?

Antigo: Alterar o renderer no ArcGIS Pro ou via REST API.

Novo: Submeter um arquivo Styled Layer Descriptor (SLD) via GeoServer REST API (/geoserver/rest/styles/).


