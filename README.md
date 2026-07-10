# GeoNode Enterprise - Infraestrutura de Dados Espaciais (IDE)

Bem-vindo ao repositório oficial de infraestrutura do **GeoNode Enterprise**, a plataforma open-source e *cloud-native* de dados espaciais mantida pela Coordenação de Infraestrutura e Operações (COOPI).

Este projeto substitui o antigo paradigma monolítico (ArcGIS Enterprise), entregando uma arquitetura 100% *Stateless* orquestrada via **Red Hat OpenShift**, com persistência delegada para os serviços gerenciados da nuvem (**AWS RDS** e **AWS S3**).

## 🧭 Índice de Documentação

Para entender profundamente o funcionamento da plataforma, consulte os documentos abaixo:

1. [Arquitetura de Rede e Portas](docs/arquitetura_portas.md)
2. [Segurança, IAM e DevSecOps](docs/seguranca_iam.md)
3. [APIs e Rotas de Acesso](docs/api_rotas.md)
4. [Guia de Operação Desktop (QGIS e Campo)](docs/guia_qgis.md)
5. [Visão de Negócios e Projetos Estratégicos](docs/negocios_operacao.md)
6. [Mapeamento de Componentes](docs/mapeamento_componentes.md)
7. [Ajustes na Camada de Banco de Dados (AWS RDS)](docs/camada_banco_dados_rds.md)
8. [Diretrizes Estratégicas e Decisões Arquiteturais (ADR)](docs/diretrizes_arquitetura.md)
9. [Guia de Transição de Serviços de Mapas](docs/guiia_de_servicos_de_mapas.md)
   

## 🚀 Como fazer o Deploy (GitOps / ArgoCD)

Este repositório é gerenciado automaticamente pelo **ArgoCD**. Não realize implantações manuais (`oc apply` ou `helm install`) no cluster de produção.

Para atualizar a plataforma:
1. Faça a alteração no código (ex: atualização da tag da imagem no `values-prod.yaml`).
2. Faça o `commit` e `push` para a branch `main`.
3. O **GitLab CI** acionará o pipeline, o **RHACS** fará o scan de vulnerabilidades da imagem no **Red Hat Quay**.
4. Sendo aprovado, o ArgoCD sincronizará o estado do OpenShift automaticamente com o Git.
