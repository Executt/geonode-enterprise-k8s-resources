# Guia de Operação: QGIS e Coleta em Campo

Este guia é direcionado aos hidrólogos, especialistas ambientais e analistas de geoprocessamento da Agência que operam dados na ponta.

## 1. Conexão via QGIS Desktop (Pela Rede ANA / VPN)
O QGIS é o cliente principal para manipulação avançada de dados. Esqueça os arquivos locais (.shp); a nova arquitetura é centralizada.

### A. Consumo de Mapas e Metadados
1. Instale o plugin **GeoNode Connection** no QGIS.
2. Adicione a URL: `https://geonode.ana.gov.br`.
3. Faça login com sua credencial corporativa. Você poderá arrastar as camadas direto do servidor para a tela.

### B. Edição Direta no Banco de Dados (Avançado)
Para processamento pesado, conecte-se direto ao AWS RDS (apenas via VPN aprovada):
1. No QGIS, vá em *Camada > Adicionar Camada > PostGIS > Nova Conexão*.
2. **Host:** Endpoint do AWS RDS (ex: `postgis-prod.rds.amazonaws.com`).
3. **Porta:** `5432`
4. **Banco de Dados:** `geonode_spatial`
5. *Atenção:* Marque a opção de SSL Mode para `Require` ou `Verify-Full`.

## 2. Consumo de Bases Externas (INPE e Sentinel)
Não baixe imagens de satélite para sua máquina. 
* Instale o plugin **STAC API Browser** no QGIS.
* **INPE:** Adicione a URL `https://brazildatacube.dpi.inpe.br/stac/`.
* Com isso, o QGIS carrega as imagens diretamente dos servidores em nuvem, poupando a banda da VPN corporativa.

## 3. Operação em Campo (Substituindo o Survey123/Field Maps)
Para atividades de campo sem internet:
1. Instale o aplicativo **QField** (Android/iOS) nos tablets operacionais.
2. No QGIS Desktop, utilize o plugin **QFieldSync** para preparar o pacote do mapa e enviar ao tablet.
3. A equipe vai a campo, coleta geometrias e fotos offline.
4. Ao retornar para uma área com Wi-Fi ou 4G corporativo, o QField sincroniza diretamente com o servidor via HTTPS (porta 443), populando o banco PostGIS no AWS RDS automaticamente.
