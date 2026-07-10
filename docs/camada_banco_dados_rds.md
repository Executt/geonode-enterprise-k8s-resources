# Ajustes na Camada de Banco de Dados (AWS RDS)

A migração da arquitetura monolítica (ArcGIS Data Store) para a abordagem *Cloud-Native* exige que todo o estado da plataforma seja externalizado do cluster OpenShift. O **AWS RDS for PostgreSQL** assume o papel central de persistência, garantindo alta disponibilidade (Multi-AZ), backups automatizados e suporte robusto a geoprocessamento.

Este documento detalha as configurações, divisões lógicas e extensões essenciais para o funcionamento do GeoNode e do GeoServer na infraestrutura da agência.

---

## 1. Topologia Lógica dos Bancos de Dados

Para garantir segurança e evitar concorrência direta de I/O entre transações web e queries espaciais pesadas, recomenda-se a criação de duas bases lógicas separadas dentro da mesma instância do RDS (ou instâncias distintas, dependendo da necessidade de escalonamento financeiro aprovada pela COOPI).

### A. Banco de Configuração (GeoNode Core)
* **Objetivo:** Armazenar metadados, perfis de usuários, permissões de acesso (Django/Guardian), histórico de uploads e sessões web.
* **Comportamento:** Alto volume de transações curtas (leitura/escrita rápidas).
* **Conexão:** Acessado exclusivamente pelo microsserviço `geonode-django` e pelos workers do `geonode-celery`.

### B. Banco Espacial e Catálogo (PostGIS + GeoServer JDBC)
* **Objetivo:** Armazenar as geometrias (SDE), camadas vetoriais reais e toda a configuração interna do motor de mapas (Workspaces, Stores, Layers e Estilos SLD) via plugin JDBC.
* **Comportamento:** Consultas analíticas pesadas e leituras massivas de blocos durante a renderização de WMS/WFS.
* **Conexão:** Acessado pelo `geonode-geoserver`, rotinas de análise de ciência de dados (JupyterHub/Databricks) e analistas via QGIS Desktop (VPN).

---

## 2. Extensões Essenciais do PostgreSQL (Extensions)

Para que o RDS atue como um verdadeiro *Spatial Database* capaz de suportar análises de recursos hídricos, roteamento e indexação complexa, as seguintes extensões devem ser habilitadas na base de dados espacial. 

A equipe de infraestrutura deve conectar-se ao RDS utilizando o usuário *superuser* (ex: `postgres`) e executar o seguinte script SQL:

```sql
-- 1. Habilita o suporte principal a geometrias, geografias e funções de análise espacial (Core)
CREATE EXTENSION IF NOT EXISTS postgis;

-- 2. Habilita a gestão de topologias (Essencial para hidrologia)
-- Garante que redes de drenagem e bacias compartilhem limites exatos, sem sobreposições ou buracos.
CREATE EXTENSION IF NOT EXISTS postgis_topology;

-- 3. Habilita o armazenamento e manipulação de metadados raster no banco
-- Embora as imagens pesadas fiquem no AWS S3 (COG), essa extensão apoia indexações menores e álgebra de mapas in-db.
CREATE EXTENSION IF NOT EXISTS postgis_raster;

-- 4. Habilita algoritmos de roteamento de redes espaciais
-- Fundamental para cálculos de fluxo d'água, caminho de menor custo e análises direcionais em rios.
CREATE EXTENSION IF NOT EXISTS pgrouting;

-- 5. Habilita o armazenamento de dados no formato Chave-Valor
-- Muito utilizado para importar tags dinâmicas no padrão OpenStreetMap (OSM) e metadados não estruturados.
CREATE EXTENSION IF NOT EXISTS hstore;

-- 6. Habilita a geração de identificadores únicos universais (UUID)
-- Padrão exigido pelo ORM do Django e pelo GeoNode para garantir chaves primárias seguras e distribuídas.
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- 7. Habilita o monitoramento avançado de performance
-- Crucial para as métricas de FinOps e identificação de queries lentas originadas pelo QGIS ou GeoServer.
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
