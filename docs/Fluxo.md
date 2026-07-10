# Arquitetura da Solução Geoespacial – ANA / AWS

Este documento descreve a arquitetura de infraestrutura e fluxo de dados da plataforma geoespacial, conectando a rede corporativa da ANA (Agência Nacional de Águas) aos serviços hospedados na AWS.

---

## Diagrama de Arquitetura (Mermaid)

O diagrama abaixo pode ser visualizado em qualquer ferramenta compatível com Mermaid (GitHub, GitLab, Notion, Obsidian, etc.). Para uma renderização interativa, acesse o [Mermaid Live Editor](https://mermaid.live/) e cole o código.

```mermaid
graph TD
    subgraph "Rede Corporativa (Rede ANA / VPN)"
        Browser[Navegador Web<br>Usuários Comuns]
        QGIS[QGIS Desktop<br>Analistas SIG]
    end

    subgraph "AWS Cloud (VPC)"
        ALB[AWS Application Load Balancer<br>Proxy Ingress - Porta 443]

        subgraph "Camada de Aplicação (Sub-redes Privadas)"
            GN[GeoNode - Django/UI<br>Porta 8000]
            GS[GeoServer - OGC<br>Porta 8080]
            Redis[Redis - Cache/Filas<br>Porta 6379]
        end

        subgraph "Camada de Dados (Sub-redes Privadas)"
            RDS[(AWS RDS - PostgreSQL<br>PostGIS / Configs<br>Porta 5432)]
            S3Local[AWS S3 Bucket<br>Arquivos Estáticos / COGs Privados]
        end

        NAT[NAT Gateway<br>Tráfego de Saída - Porta 443]
    end

    subgraph "Bases Externas"
        INPE[INPE - Brazil Data Cube<br>STAC / WTMS]
        Sentinel[AWS Open Data - Sentinel-2<br>Imagens COG]
    end

    %% Conexões Rede ANA -> AWS
    Browser -- "HTTPS :443<br>(Acesso Web Portal)" --> ALB
    QGIS -- "HTTPS :443<br>(Consumo WMS/WFS/WMTS)" --> ALB
    QGIS -. "VPN / TCP :5432<br>(Acesso Nativo PostGIS)" .- RDS

    %% Conexões Internas (AWS ALB -> App)
    ALB -- "HTTP :8000<br>(Roteamento UI)" --> GN
    ALB -- "HTTP :8080<br>(Roteamento OGC)" --> GS

    %% Conexões Entre Componentes
    GN <--> "HTTP :8080<br>(Sincronização/Autenticação)" GS
    GN --> "TCP :6379<br>(Celery Tasks)" Redis

    %% Conexões App -> Banco/Storage
    GN -- "TCP :5432<br>(Tabelas Django)" --> RDS
    GS -- "TCP :5432<br>(Vetores Espaciais JDBC)" --> RDS
    GN -- "S3 API (Boto3)<br>(Uploads/Media)" --> S3Local
    GS -- "S3 API<br>(Leitura COG Nativa)" --> S3Local

    %% Conexões com o Mundo Externo (Egress e INPE)
    QGIS -- "HTTPS :443<br>(Plugin STAC API)" --> INPE
    GS -- "Tráfego Outbound<br>(WMS Cascateado / AWS S3 API)" --> NAT
    NAT --> INPE
    NAT --> Sentinel


## 2. Fluxo Estrutural da Arquitetura (Resumo Visual Gráfico)
Caso prefira uma visualização rápida em texto, aqui está a topologia estruturada de como os dados trafegam do analista na ANA até os servidores na AWS:

[ ZONA 1: REDE ANA / VPN ]
Analista (QGIS Desktop):
Consulta catálogo do Inpe via Plugin STAC (HTTPS 443).
Edita vetores e vê mapas do GeoNode via WFS/WMS (HTTPS 443).
Faz queries pesadas direto no banco (TCP 5432 - via VPN restrita).
⬇ Tráfego de Internet/VPN protegido por AWS WAF & Security Groups ⬇

[ ZONA 2: AWS VPC - CAMADA DE ENTRADA PÚBLICA ]
AWS ALB (Load Balancer): Recebe requisições na porta 443 (com certificado SSL), descriptografa e faz o roteamento interno.
⬇ Tráfego roteado para instâncias isoladas (Sub-redes Privadas) ⬇

[ ZONA 3: AWS VPC - CAMADA DE COMPUTAÇÃO (Privada) ]
GeoNode (Django/Python): Roda internamente (Porta 8000). Gerencia usuários, interface web, permissões e envia tarefas em background para o Redis (Porta 6379).
GeoServer: Roda internamente (Porta 8080). Renderiza as imagens, aplica estilos cartográficos e valida as permissões com o GeoNode.
⬇ Requisição de Dados ⬇

[ ZONA 4: AWS VPC - CAMADA DE DADOS E STORAGE (Privada) ]
AWS RDS (PostgreSQL + PostGIS): Recebe requisições na porta 5432 do GeoNode e GeoServer. Guarda tanto o modelo de usuários quanto as geometrias vetoriais da ANA.
AWS S3 (Bucket Interno): Armazena arquivos do site, PDFs, Shapes originais enviados por usuários e recortes raster gerados pela equipe da ANA.
➡ Tráfego de Saída via NAT Gateway ➡

[ ZONA 5: INTERNET E FONTES EXTERNAS ]
AWS Open Data S3: O GeoServer busca os blocos de imagens Sentinel-2 (COGs) diretamente da infraestrutura aberta da AWS, sem baixar o arquivo inteiro.
INPE (Brazil Data Cube): O GeoServer atua como proxy (WMS Cascateado) e busca matrizes temporais de desmatamento/uso do solo do INPE para sobrepor no portal corporativo.


 ## 3. Fluxo
[ REDE ANA / QGIS / VPN ]
                  │
                  │ HTTPS (443) / TCP (5432 restrito)
                  ▼
      [ AWS APPLICATION LOAD BALANCER ]
                  │
                  │
 ┌────────────────┴─────────────────────────────────────────┐
 │ AWS VPC (Sub-redes Privadas)                             │
 │                                                          │
 │      [ CAMADA DE APLICAÇÃO ]                             │
 │       │                                                  │
 │       ├─► GeoNode (Django - Porta 8000) ◄─────┐          │
 │       │                                       │          │
 │       └─► GeoServer (OGC - Porta 8080)  ◄─────┤          │
 │                                               │          │
 │                                               │          │
 │      [ CAMADA DE DADOS E STORAGE ]            │          │
 │       │                                       │          │
 │       ├─► AWS RDS (PostgreSQL/PostGIS) ◄──────┘          │
 │       │     (Porta 5432)                                 │
 │       │                                                  │
 │       └─► AWS S3 (Bucket Interno) ◄──────────────────────┤
 │             (Imagens COG e Mídias)                       │
 │                                                          │
 │                                                          │
 │      [ SAÍDA EXTERNA (EGRESS) ]                          │
 │       │                                                  │
 │       └─► NAT Gateway (Porta 443) ────────────────┐      │
 └───────────────────────────────────────────────────┼──────┘
                                                     │
               ┌─────────────────────────────────────┴──────────┐
               ▼                                                ▼
   [ INPE (Brazil Data Cube) ]                   [ AWS Open Data (Sentinel-2) ]
      (Via WMS Cascateado)                           (Via S3 GeoTIFF Plugin)
