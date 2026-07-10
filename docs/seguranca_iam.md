# Segurança, Identidade (IAM) e DevSecOps

A plataforma foi desenhada sob os princípios de *Zero Trust* e conformidade rígida exigida no setor público.

## 1. Security Context Constraints (SCC)
O OpenShift bloqueia, por padrão, a execução de contêineres como `root`. 
Nossas imagens customizadas do GeoNode e GeoServer (hospedadas no **Red Hat Quay**) são construídas no pipeline do GitLab para operar estritamente sob o UID/GID `1001`. A aplicação depende do arquivo `scc-geonode.yaml` para autorizar a execução segura no namespace.

## 2. DevSecOps Pipeline
Nenhum código vai para produção sem passar pelas seguintes barreiras:
* **SonarQube (SAST):** Análise do código Python/Django em busca de *code smells* e credenciais vazadas.
* **Red Hat Advanced Cluster Security (RHACS):** Analisa o binário da imagem Docker armazenada no Quay. Se for detectada uma vulnerabilidade Crítica ou Alta (ex: falhas no Log4j ou no Tomcat do GeoServer), o *build* falha e o deploy no ArgoCD é bloqueado.

## 3. Gestão de Identidade (IAM)
A plataforma não utiliza banco de usuários local isolado. A gestão é centralizada via **Single Sign-On (SSO / OIDC)**.
* **Provedor:** Microsoft Entra ID (Azure AD) ou Gov.br.
* **Fluxo:** O usuário não trafega senhas na infraestrutura do GeoNode. A autenticação é delegada via token OAuth2.
* **Fallback Legado:** Caso a rede exija, o código possui suporte nativo ao Active Directory (LDAP) local configurável no `values-prod.yaml`.

## 4. Gerenciamento de Segredos (Secrets)
Nenhuma senha (banco de dados, AWS Access Keys ou certificados) reside em texto plano no Git. O cluster utiliza um *Secret Operator* (ou integração nativa com AWS Secrets Manager) para injetar essas credenciais no OpenShift em tempo de execução.
