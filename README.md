
# Pipeline ELT - Concessionária de veículos

- Este projeto mostra uma pipeline ELT (Extract - Load - Transform) de um banco de dados PostgreSQL em ambiente on-premise. Os dados são de uma concessionária de veículos e o banco de dados recebe atualizações constantes, recebendo valores transacionais assim que uma venda é realizada. 
- O objetivo deste projeto é integrar ferramentas dos três estágios da ELT, respondendo as questões de negócio:

## Questões de negócio:

### 1. Realizar a análise de vendas por concessionária;
### 2. Quais foram as vendas por modelo de veículo;
### 3. Vendas por vendedor;
### 4. Vendas em análise temporal (por mês ou por ano, por exemplo);.


## Pré requisitos:
- PgAdmin;
- Conta AWS (verificar custos, dependendo da EC2 escolhida);
- Conta dbt cloud;
- Conta Snowflake (possui $400 doláres de free trial);
- Conta Google (para utilizar o Looker como ferramenta de BI).


## Arquitetura

![Arquitetura](https://github.com/rogerrdasilva/elt_postgres_to_snowflake/blob/main/ImagesForReadme/arquitetura.png)


## Modelagem de dados dimensional:

![StarSchema](https://github.com/rogerrdasilva/elt_postgres_to_snowflake/blob/main/ImagesForReadme/star_schema_relacionamento.png)


## Conexão banco de dados Postgre (os dados podem sofrer alterações):

- host: 159.223.187.110
- dbname: novadrive
- user: etlreadonly
- password: novadrive376A@

Com essas informações, conseguimos conectar ao Postgres de exemplo com a ferramenta PgAdmin4 e realizar querys para consultar as tabelas de origem.

Estrutura de tabelas na origem (PgAdmin):

![OnPremises](https://github.com/rogerrdasilva/elt_postgres_to_snowflake/blob/main/ImagesForReadme/database_origem.jpg)

# Orquestrador: Apache Airflow

Existem muitas formas de utilizar o Apache Airflow em cloud para ingestão ou migração de dados de um ambiente on premises, como por exemplo:
- Airflow gerenciado por cloud (AWS - MWAA / GCP Composer): Esta forma é escalável e gerenciada pelos provedores cloud, sendo bastante recomendado;
- Airflow com K8s: No Kubenetes, se tem o melhor do open source e gerenciamento;
- Utilizar o Astro CLI: A forma mais simples de estudar Airflow, por ser local, e bem abstraída. Poderiamos, por exemplo, realizar todo o desenvolvimento local em seguida iniciar uma VM em cloud e realizar o pull da imagem docker local para o repositório de imagens docker, e dentro da VM realizar um Push desta imagem.
- A forma que utilizaremos o Airflow neste projeto, embora com comandos shell um pouco mais complexos, foi a que uniu o melhor custo e escalabilidade para este estudo: Executar uma instância EC2 e dentro dessa máquina virtual instalar o Docker, Docker compose e persistir o volume. O passo a passo será explicado a seguir.

## Provisionar o Airflow na AWS EC2.
- Com uma conta ativa na AWS, no painel inicial procurar por EC2, escolher pela opção "Ubuntu". Para este estudo, utilizei a "t2.large" e os gastos estimados ficaram em $0,27.
- É necessário criar uma "key pair" no momento de criação da instância, pois essa chave vai ser a conexão via SSH. Marcar a opção "Allow SSH traffic from", e podemos escolher somente o IP ou "anywhere". Por segurança, aconselho escolher apenas o IP para conexão. Por fim, clicar em instanciar e aguardar até que o status esteja em "running".

- Uma vez que a instância estiver disponível, clicamos nela e na aba "connect", clicar em "SSH" e copiar o comando que vai ser no formato a seguir:

` ssh -i "keyaws.pem" ubuntu@ec2-18-233-66-127.compute-1.amazonaws.com` 
Colar este endereço em um terminal, por exemplo git-bash. Necessário ter a chave "key-pair" no mesmo diretório para que a conexão funcione corretamente.
- Nota: Ao pausar a instância e ligá-la novamente, o comando de conexão vai mudar, pois o nome `@ec2-18-233-66-127` geralmente muda também.

- Uma vez conectado, digitar os comandos abaixo seguindo a ordem. Os comandos basicamente realizam a instalação do Docker, persistem volume, e fazem o pull da imagem Docker-compose oficial do Airflow para a instalação dentro da EC2.

### 1. Atualizar a lista de pacotes do APT:
`sudo apt-get update`;
### 2. Instalar pacores necessários para adicionar um novo repositório via HTTPS:
`sudo apt-get install ca-certificates curl gnupg lsb-release`
### 3. Criar diretório para armazenar as chaves de repositórios: 
`sudo mkdir -m 0755 -p /etc/apt/keyrings`
### 4. Adicionar a chave GPG do repositório do Docker:
`curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg`
### 5. dicionar o repositório do Docker às fontes do APT:
`echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null `
### 6. Atualiza a lista de pacotes após adicionar o novo repositório do Docker:
`sudo apt-get update`
### 7. Instalar o Docker e componentes:
`sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
### 8. Baixar o arquivo docker-compose.yaml do Airflow:
`curl -LfO 'https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml'`
### 9. Criar diretórios para DAGs, logs e plugins:
`mkdir -p ./dags ./logs ./plugins`
### 10. Criar um arquivo .env com o UID do usuário, usado pelo docker para permissões:
`echo -e "AIRFLOW_UID=$(id -u)" > .env`
### 11. Inicia o Airflow
`sudo  docker compose up airflow-init`
### 12 Subir o Airflow em modo desacoplado:
`sudo docker compose up -d`

- Aguardar os containers entrarem no status "Healthy". Podemos realizar a checagem via terminal com o comando:
`sudo docker ps`

- Assim que todos os containers estiverem com o status Healthy, podemos acessar o Airflow copiando o nome da EC2 e adicionando no final a porta padrão `:8080`em qualquer browser:
`http://ec2-18-233-66-127.compute-1.amazonaws.com:8080`

- O proximo passo não é mandatório, mas deixa o seu ambiente no Airflow mais limpo. Marcamos a opção de FALSE para que ele não suba os exemplos nativos, editando o docker-compose.yaml. Digitando o comando abaixo, basta procurar por "AIRFLOW__CORE__LOAD_EXAMPLES='true'" e trocar por false, utilizando o editor Nano:
`nano /home/ubuntu/docker-compose.yaml`

- Reiniciar o Airflow:
```
#reiniciar
sudo docker compose stop
sudo docker compose up -d
sudo docker ps

```
- O próximo passo seria criar as conexões, entretanto, é necessário ter o ambiente no data warehouse criado, podendo ser: AWS Redshift, GCP Big Query, Azure Fabric/Synapse ou Snowflake. Neste estudo, foi utilizado Snowflake.


# Configuração Snowflake:
Com a conta criada no Snowflake, o próximo passo é preparar o ambiente para consumir a ingestão dos dados que virá do Airflow.

Abrir um editor de código do Snowflake, digitar os códigos abaixo e clicar em "run all". Os códigos abaixo tem o papel de criar um database, um schema, e a definição das tabelas que são carregadas pelo Airflow.
```
create database novadrive;
create schema stage;
 
CREATE WAREHOUSE DEFAULT_WH;
 
CREATE TABLE veiculos (
    id_veiculos INTEGER,
    nome VARCHAR(255) NOT NULL,
    tipo VARCHAR(100) NOT NULL,
    valor DECIMAL(10, 2) NOT NULL,
    data_atualizacao TIMESTAMP_LTZ,
    data_inclusao TIMESTAMP_LTZ
);
 
CREATE TABLE estados (
    id_estados INTEGER,
    estado VARCHAR(100) NOT NULL,
    sigla CHAR(2) NOT NULL,
    data_inclusao TIMESTAMP_LTZ,
    data_atualizacao TIMESTAMP_LTZ
);
 
CREATE TABLE cidades (
    id_cidades INTEGER,
    cidade VARCHAR(255) NOT NULL,
    id_estados INTEGER NOT NULL,
    data_inclusao TIMESTAMP_LTZ,
    data_atualizacao TIMESTAMP_LTZ
 
);
 
CREATE TABLE concessionarias (
    id_concessionarias INTEGER,
    concessionaria VARCHAR(255) NOT NULL,
    id_cidades INTEGER NOT NULL,
    data_inclusao TIMESTAMP_LTZ,
    data_atualizacao TIMESTAMP_LTZ
);
 
CREATE TABLE vendedores (
    id_vendedores INTEGER,
    nome VARCHAR(255) NOT NULL,
    id_concessionarias INTEGER NOT NULL,
    data_inclusao TIMESTAMP_LTZ,
    data_atualizacao TIMESTAMP_LTZ
);
 
CREATE TABLE clientes (
    id_clientes INTEGER,
    cliente VARCHAR(255) NOT NULL,
    endereco TEXT NOT NULL,
    id_concessionarias INTEGER NOT NULL,
    data_inclusao TIMESTAMP_LTZ,
    data_atualizacao TIMESTAMP_LTZ
);
 
CREATE TABLE vendas (
    id_vendas INTEGER,
    id_veiculos INTEGER NOT NULL,
    id_concessionarias INTEGER NOT NULL,
    id_vendedores INTEGER NOT NULL,
    id_clientes INTEGER NOT NULL,
    valor_pago DECIMAL(10, 2) NOT NULL,
    data_venda TIMESTAMP_LTZ,
    data_inclusao TIMESTAMP_LTZ,
    data_atualizacao TIMESTAMP_LTZ
);


```

No tutorial, temos informações que serão utilizadas durante todo o estudo, como credenciais, nome de database, nome do schema, etc.

Login: < vai ser unico para cada pessoa >
Password: < unico para cada pessoa >
database: NOVADRIVE
warehouse: DSEFAULT_DW
schema: STAGE
account: `https://app.snowflake.com/XXXXX/YYYYY/worksheets` este é o formato comum da URL, a account vai ser XXXXX-YYYYY.
 
Para conectar ao Looker Studio, será necessário ter essa informação também:
`XXXXX-YYYYY.snowflakecomputing.com`

- Tendo o snowflake ou outra ferramenta de data warehouse configurada, voltamos para o Airflow para criar as conexões. Essas conexões são importantes, pois vão realizar a integração do Airflow com Snowflake.

# Criando Conexões no Airflow:

- Tendo a EC2 já instanciada e com a URL do Airflow já disponível para ser acessada, por exemplo:
`http://ec2-35-175-126-189.compute-1.amazonaws.com:8080/`

- No menu canto superior direito clicar em conexões, depois clicar em criar.

- Conexão para integrar com o Postgres on-premises:
```
connection id: postgres
login: etlreadonly
host: 159.223.187.110
database: novadrive
password: novadrive376A@
port: 5432

```

- Conexão Airflow-Snowflake:
```
connection id: snowflake
schema: STAGE
login: <seu login>
account: XXXXX-YYYYY
password: <sua-senha-snowflake>
database: NOVADRIVE
warehouse: DEFAULT_WH


```

# Levando o código da DAG, que está na pasta `./dag`
- O ideal seria um processo de CI/CD, onde faríamos um push para um repositório remoto, e lá o processo CI/CD iria publicar a DAG. Mas neste estudo, foi implementado o código da DAG através do editor Nano e do git-bash já conectado à EC2.
- Digitar o comando :  `touch dag.py`;
- Editar : `nano dag.py`;
- Copiar o código da DAG no VSCode para o editor Nano (no terminal bash);
- Control + X para salvar;
- digitar: `cat dag.py` para verificar se o código foi salvo na DAG do container.


# dbt:
Ferramenta utilizada para a modelagem de dados dimensional, construção das tabelas fato e dimensão.

- Criar uma conta no dbt;
- A própria ferramenta vai guiar para criar um `enviroment` e um `projeto`;
- Selecionar o conector Snowflake;
- Seguir com next até entrar na `DBT cloud IDE`;
- Observar a estrutura de pastas e arquivos e clicar para editar a file `dbt_project.yml`:
Alterar apenas o nome do projeto conforme a necessidade.
- Clicar na pasta Models:
Esta pasta vai ser muito importante, pois irá conter os scripts para modelagem das tabelas.

- Criar o arquivo `source.yml`:
```
version: 2

sources:
  - name: sources
    database: NOVADRIVE
    schema: STAGE
    tables:
      - name: cidades
      - name: clientes
      - name: concessionarias
      - name: estados
      - name: veiculos
      - name: vendas
      - name: vendedores

```
- Separar dentro da pasta models, as pastas referentes a cada transformação: `stage`, `dimensions`, `facts`, `analysis`. Consultar a pasta `./dbt/models` deste repositório para encontrar os códigos referentes a cada processo.

- Stage é a camada "raw/crua" das tabelas que vem do Postgres, no dbt foi feita uma materialização do tipo view para ser a base da criação das outras tabelas.

- Dimensions e Fact: são as tabelas do star schema para modelagem dimensional e geração de insights no negócio.

- Analysis: são as tabelas modeladas especificamente para atender e responder as questões de negócios levantadas nos primeiros tópicos desta documentação. Foi feito desta forma para simplificar o relacionamento das tabelas nas ferramentas de BI. Entretanto, seria perfeitamente possível realizar a mesma análise apenas com as tabelas fato e dimensão. Porém, seria necessário ter um nivel de atenção no relacionamento das tabelas na ferramenta de BI, para não prejudicar a integridade do relatório.

- Para rodar os modelos e visualizar as tabelas no Snowflake, basta clicar na CLI do próprio dbt IDE e digitar: 
`dbt run`

![DataWarehouse](https://github.com/rogerrdasilva/elt_postgres_to_snowflake/blob/main/ImagesForReadme/snowflake_DW_completo.png)

- Antes de realizar o deploy, é interessante realizar um test de data quality e data integrity. Na pasta tests do dbt, criar um arquivo chamado test.sql que está neste repositório na pasta `./tests`. Rodar o teste com o comando:
`dbt test`

- É possivel consultar a linhagem dos dados clicando em cada arquivo no dbt, por exemplo, a linhagem da tabela fato mostra o projeto completo:

![DataLineage](https://github.com/rogerrdasilva/elt_postgres_to_snowflake/blob/main/ImagesForReadme/data_lineage_dbt.png)

# Criação do Dashboard:

Foi criado via Google Looker Studio o dashboard que contém informações de vendas de carros por concessionárias, sendo possível filtrar por estado e por concessionária:


![Dashboard](https://github.com/rogerrdasilva/elt_postgres_to_snowflake/blob/main/ImagesForReadme/dashboard_looker_studio.png)

