# Pacotes OpenTelemetry para instalar na sua API Node.js

# Instale com npm:
npm install \
  @opentelemetry/sdk-node \
  @opentelemetry/auto-instrumentations-node \
  @opentelemetry/exporter-logs-otlp-http \
  @opentelemetry/exporter-metrics-otlp-http \
  @opentelemetry/resources \
  @opentelemetry/semantic-conventions \
  @opentelemetry/sdk-logs \
  @opentelemetry/sdk-metrics

# ─── Env vars que você precisa adicionar no EasyPanel (App da API) ───────────
#
# OTEL_EXPORTER_OTLP_ENDPOINT=http://<nome-do-servico>:4318
#   Substitua <nome-do-servico> pelo hostname interno do otel-collector.
#   O formato do EasyPanel costuma ser: <projeto>_<servico>-<stack>
#   Exemplo: http://meuprojeto_otel-collector-monitoring:4318
#   → Descubra o hostname exato conforme o README principal.
#
# OTEL_SERVICE_NAME=minha-api
#   Nome do seu serviço que aparecerá nos dashboards e nos logs.
#
# NODE_OPTIONS=--require ./src/instrumentation.js
#   Carrega o bootstrap do OTel antes de qualquer outro módulo.
#   Ajuste o caminho conforme onde você colocou o arquivo instrumentation.js.
#
# ─────────────────────────────────────────────────────────────────────────────

# ─── Como usar o arquivo instrumentation.js ──────────────────────────────────
#
# 1. Copie o arquivo instrumentation.js para dentro da sua API:
#    cp instrumentation.js ../sua-api/src/instrumentation.js
#
# 2. Adicione a env var NODE_OPTIONS no EasyPanel OU altere o script no package.json:
#    "start": "node --require ./src/instrumentation.js src/index.js"
#
# ─────────────────────────────────────────────────────────────────────────────
