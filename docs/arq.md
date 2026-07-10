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
