# Monitoramento_Docker

Este projeto oferece um ecossistema completo de monitoramento utilizando containers Docker, facilmente orquestrados via **docker-compose**. São incluídos Zabbix (com PostgreSQL), Grafana e Prometheus, permitindo deploy rápido para testes, ambiente de desenvolvimento ou produção.

## Estrutura de Pastas

A estrutura do projeto está organizada para separar dados persistentes e configurações de cada serviço, facilitando backup, versionamento e personalização. Veja um exemplo:

```
Monitoramento_Docker/
├── docker-compose.yml
├── prometheus/
│   ├── config/
│   │   └── prometheus.yml
│   └── data/
├── grafana/
│   ├── config/
│   │   └── datasources.yml
│   └── data/
└── zabbix/
    └── data/
        └── postgresql/
```

- **docker-compose.yml**: arquivo principal para subir todo o ambiente de monitoramento.
- **prometheus/config/prometheus.yml**: configuração dos targets e jobs do Prometheus.
- **prometheus/data/**: persistência de dados do Prometheus.
- **grafana/config/datasources.yml**: configurações das fontes de dados do Grafana (opcional).
- **grafana/data/**: persistência de dados do Grafana (dashboards, plugins, etc).
- **zabbix/data/postgresql/**: persistência dos dados do banco PostgreSQL usado pelo Zabbix Server.

**Obs:** Altere os caminhos dos volumes no docker-compose conforme sua necessidade ou organização no host.

## Serviços incluídos

Principais serviços orquestrados pelo `docker-compose.yml`:

- **Zabbix Server**: Ferramenta de monitoramento principal, integrada ao banco PostgreSQL.
- **Zabbix Web**: Interface web para gerenciamento e visualização do Zabbix.
- **Zabbix Agent**: Coleta dados localmente e envia ao Zabbix Server.
- **Zabbix Postgres**: Banco de dados PostgreSQL persistente para o Zabbix.
- **Grafana**: Visualização de dados, dashboards customizáveis, com plugin Zabbix já instalado.
- **Prometheus**: Coletor de métricas para ambientes e aplicações.


## Como usar

1. **Clone o repositório:**

```bash
git clone https://github.com/JeanPauloRN/Monitoramento_Docker.git
cd Monitoramento_Docker
```

2. **Ajuste os volumes:**

Modifique os caminhos dos volumes no `docker-compose.yml` ou crie as pastas correspondentes no seu host conforme a sua preferência.
3. **Adicione arquivos de configuração:**
    - Coloque arquivos como `prometheus.yml` (Prometheus) e `datasources.yml` (Grafana) nas respectivas pastas `config` apontadas no compose.
    - Edite as configurações segundo seu ambiente.
4. **Suba os containers:**

```bash
docker-compose up -d
```

O Docker irá baixar as imagens, criar as redes necessárias e iniciar os serviços com persistência de dados configurada.

## Dicas e Observações

- As portas padrão de cada serviço são expostas localmente conforme `docker-compose.yml`.
    - Zabbix Server: `10051`
    - Zabbix Agent: `10050`
    - Zabbix Web: `8080`
    - Grafana: `3000`
    - Prometheus: `9090`
    - PostgreSQL: `5432`
- Usuário e senha do Grafana padrão:
    - Usuário: **admin**
    - Senha: **admin123** (definida na variável de ambiente)
- **Volumes persistentes**: Certifique-se de que os caminhos usados nos volumes existem e estão acessíveis para o Docker.
- Você pode customizar configurações e dashboards pela UI do Grafana e Zabbix.


## docker-compose.yml

```yaml
# docker-compose.yml
services:
  zabbix-postgres:
    image: postgres:16-alpine
    container_name: zabbix-postgres
    restart: unless-stopped
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=zabbixpass
      - POSTGRES_DB=zabbix
    volumes: #volume do /data
      - /home/jean-carmo/Documentos/Projetos/Monitoramento/zabbix/data/postgresql:/var/lib/postgresql/data
      - /etc/localtime:/etc/localtime:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U zabbix -d zabbix"]
      interval: 10s
      timeout: 5s
      retries: 5

  zabbix-server:
    image: zabbix/zabbix-server-pgsql:latest
    container_name: zabbix-server
    restart: unless-stopped
    environment:
      - DB_SERVER_HOST=zabbix-postgres
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=zabbixpass
      - POSTGRES_DB=zabbix
      - ZBX_NODEADDRESS=zabbix-server
      - ZBX_NODEADDRESSPORT=10051
    depends_on:
      zabbix-postgres:
        condition: service_healthy
    ports:
      - "10051:10051"
    volumes:
      - /etc/localtime:/etc/localtime:ro

  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:latest
    container_name: zabbix-web
    restart: unless-stopped
    environment:
      - DB_SERVER_HOST=zabbix-postgres
      - POSTGRES_USER=zabbix
      - POSTGRES_PASSWORD=zabbixpass
      - POSTGRES_DB=zabbix
      - ZBX_SERVER_HOST=zabbix-server
      - ZBX_SERVER_PORT=10051
      - PHP_TZ=America/Sao_Paulo
    depends_on:
      zabbix-server:
        condition: service_started
      zabbix-postgres:
        condition: service_healthy
    ports:
      - "8080:8080"
    volumes:
      - /etc/localtime:/etc/localtime:ro

  zabbix-agent:
    image: zabbix/zabbix-agent:latest
    container_name: zabbix-agent
    restart: unless-stopped
    environment:
      - ZBX_SERVER_HOST=zabbix-server
      - ZBX_ACTIVE_ALLOW=false
      - ZBX_HOSTNAME=zabbix-agent
    ports:
      - "10050:10050"
    depends_on:
      - zabbix-server
    volumes:
      - /etc/localtime:/etc/localtime:ro

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: unless-stopped
    user: "0"
    ports:
      - "3000:3000"
    volumes: #Persistência de dados do Grafana e personalização de DataSources
      - /home/jean-carmo/Documentos/Projetos/Monitoramento/grafana/data:/var/lib/grafana 
      - /home/jean-carmo/Documentos/Projetos/Monitoramento/grafana/config/datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_INSTALL_PLUGINS=alexanderzobnin-zabbix-app
    
  
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes: #Persistência de dados do Prometheus e personalização dos Targets
      - /home/jean-carmo/Documentos/Projetos/Monitoramento/prometheus/config/prometheus.yml:/etc/prometheus/prometheus.yml
      - /home/jean-carmo/Documentos/Projetos/Monitoramento/prometheus/data:/etc/prometheus
```


### Enjoy This !
