# manager-loggin

Stack de monitoramento para APIs Node.js/Express usando OpenTelemetry, Loki, Prometheus e Grafana. Projetado para deploy no [EasyPanel](https://easypanel.io) como um serviço Docker Compose ao lado da sua API.

## Arquitetura

```
EasyPanel Project
├── App: sua-api (Dockerfile)           ← sua API, não muda nada
└── Service: manager-loggin (Compose)
    ├── otel-collector :4317/:4318      ← recebe logs+métricas da API
    ├── loki           :3100            ← armazena logs
    ├── prometheus     :9090            ← armazena métricas
    └── grafana        :3000            ← dashboards (acesso externo)

Fluxo:
API → [OTLP HTTP :4318] → otel-collector → Loki (logs)
                                         → Prometheus (métricas)
Grafana lê de Loki + Prometheus
```

## Pré-requisitos

- EasyPanel com projeto existente onde sua API já está rodando
- Sua API usa Node.js/Express

---

## 1. Deploy do stack de monitoramento no EasyPanel

### 1.1 Criar o serviço

1. No EasyPanel, abra o **mesmo projeto** da sua API
2. Clique em **+ Create Service** → **Docker Compose**
3. Aponte para este repositório (GitHub)
4. Nome sugerido: `monitoring`

### 1.2 Configurar as variáveis de ambiente

No painel do serviço, em **Environment**, adicione:

```env
ENVIRONMENT=production
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=sua_senha_segura
LOKI_RETENTION_PERIOD=744h
```

> ⚠️ **Troque `GRAFANA_ADMIN_PASSWORD`** — nunca use a senha padrão em produção.

### 1.3 Expor o Grafana

1. No serviço `monitoring`, vá em **Domains**
2. Adicione um domínio para a porta `3000`
3. Ex: `grafana.seudominio.com`

> Os outros serviços (otel-collector, loki, prometheus) **não precisam** de domínio externo — eles ficam apenas na rede interna.

---

## 2. Configurar a API para enviar telemetria

### 2.1 Descobrir o hostname interno do OTel Collector

O EasyPanel usa uma convenção de nomes para a rede interna Docker. Para descobrir o hostname correto:

1. No EasyPanel, abra o **terminal** do container da sua API (ou use SSH no servidor)
2. Execute: `getent hosts otel-collector` ou tente fazer `curl` para possíveis hostnames
3. O padrão costuma ser: `<projeto>_otel-collector-<servico>`

**Dica prática:** Baseado no padrão do seu banco de dados (`planofit_bdd-plano-fit:5432`), o hostname do collector provavelmente será algo como:
```
<nome-do-projeto>_otel-collector-monitoring:4318
```

Teste com:
```bash
curl http://<hostname>:4318/
```

### 2.2 Instalar os pacotes na API

```bash
npm install \
  @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-logs-otlp-http \
  @opentelemetry/exporter-metrics-otlp-http \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions \
  @opentelemetry/sdk-logs \
  @opentelemetry/sdk-metrics
```

### 2.3 Copiar o arquivo de instrumentação

Copie `api-instrumentation/instrumentation.js` para dentro da sua API:

```bash
cp api-instrumentation/instrumentation.js ../sua-api/src/instrumentation.js
```

### 2.4 Adicionar as env vars na API (EasyPanel)

No app da sua API, em **Environment**, adicione:

```env
OTEL_EXPORTER_OTLP_ENDPOINT=http://<hostname-do-collector>:4318
OTEL_SERVICE_NAME=minha-api
NODE_OPTIONS=--require ./src/instrumentation.js
```

> Ajuste o caminho `./src/instrumentation.js` conforme onde você colocou o arquivo.

---

## 3. Verificar se está funcionando

1. Acesse o Grafana em `grafana.seudominio.com`
2. Login com as credenciais configuradas
3. Vá em **Dashboards** → **Node.js API — Overview**
4. Faça algumas requisições para a sua API e aguarde ~30s para os dados aparecerem

### Checklist de troubleshooting

| Problema | O que verificar |
|---|---|
| Grafana não abre | Porta 3000 exposta no EasyPanel? Domínio configurado? |
| Nenhum dado no dashboard | Hostname do collector correto? Pacotes OTel instalados? `NODE_OPTIONS` configurado? |
| Logs não aparecem | `OTEL_SERVICE_NAME` definido? Testar com `curl -X POST http://<collector>:4318/v1/logs` |
| Métricas não aparecem | Prometheus consegue scrape? Acessar `http://<collector>:8889/metrics` internamente |

---

## 4. Estrutura do repositório

```
manager-loggin/
├── docker-compose.yml                          ← orquestração dos serviços
├── .env.example                                ← template de variáveis
├── configs/
│   ├── otel-collector-config.yaml             ← recebe OTLP, exporta para Loki/Prometheus
│   ├── loki-config.yaml                       ← armazenamento de logs
│   ├── prometheus.yml                         ← coleta de métricas
│   └── grafana/
│       ├── provisioning/
│       │   ├── datasources/datasources.yaml   ← Loki + Prometheus auto-configurados
│       │   └── dashboards/dashboards.yaml     ← auto-carrega dashboards da pasta abaixo
│       └── dashboards/
│           └── nodejs-api.json                ← dashboard pronto para Express
└── api-instrumentation/
    ├── instrumentation.js                     ← copie para sua API
    └── INSTALL.md                             ← guia de instalação dos pacotes
```

---

## Retenção de dados

| Dado | Padrão | Variável |
|---|---|---|
| Logs (Loki) | 31 dias | `LOKI_RETENTION_PERIOD` |
| Métricas (Prometheus) | 30 dias | fixo no compose (altere `--storage.tsdb.retention.time`) |
| Dashboards (Grafana) | permanente | volume persistente |
