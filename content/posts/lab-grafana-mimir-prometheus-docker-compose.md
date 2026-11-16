+++
title = "Lab Grafana Mimir com Prometheus e Docker Compose"
date = 2026-11-15T18:00:00-03:00
draft = false
tags = ["observabilidade", "prometheus", "mimir", "grafana", "docker"]

[cover]
    image = "/images/mimir-logo.png"
    alt = "Logo do Grafana Mimir"
    relative = false
+++

## Por que o Prometheus sozinho não basta

O Prometheus resolve muito bem a coleta e a consulta de métricas, mas o TSDB local dele não foi desenhado para retenção de longo prazo, alta disponibilidade nem multi-tenancy. Quando o disco enche ou a instância morre, o histórico vai junto.

O [Grafana Mimir](https://grafana.com/oss/mimir/) ataca exatamente esse problema: é um backend de métricas horizontalmente escalável, sucessor do Cortex, que recebe séries via `remote_write` e expõe uma API compatível com a do Prometheus para consulta. Na prática, o Prometheus vira um agente de coleta e o Mimir assume o papel de armazenamento central.

Neste post, monto um laboratório local completo com Docker Compose: uma aplicação de exemplo instrumentada gerando métricas de verdade, Prometheus coletando e enviando via `remote_write`, Mimir armazenando em modo single-binary e Grafana consultando tudo num dashboard pronto — além de duas armadilhas reais que encontrei no caminho.

Subir os containers e ver tudo verde é fácil. O que eu queria era ver métricas de aplicação atravessando o pipeline inteiro — por isso o lab inclui uma API falsa de e-commerce com um gerador de carga em cima dela.

## Prometheus vs Thanos vs Mimir: um breve comparativo

Antes de pôr a mão na massa, vale situar o Mimir no ecossistema. As três ferramentas não competem exatamente no mesmo nível — o Prometheus é a base de coleta, enquanto Thanos e Mimir são as duas respostas mais populares para escalar essa base:

| Critério | Prometheus | Thanos | Mimir |
|---|---|---|---|
| **Papel** | Coleta, armazenamento local e consulta | Camada de longo prazo sobre Prometheus | Backend central de métricas (sucessor do Cortex) |
| **Retenção de longo prazo** | Limitada ao disco local | Object storage (S3, GCS…) | Object storage (S3, GCS…) |
| **Alta disponibilidade** | Não nativa (instâncias duplicadas e independentes) | Deduplicação de réplicas na consulta | Replicação nativa entre ingesters |
| **Multi-tenancy** | Não | Parcial (via labels externos) | Nativa e isolada por tenant (`X-Scope-OrgID`), com limites por tenant |
| **Modelo de ingestão** | Scrape (pull) | Sidecar junto a cada Prometheus (upload de blocos) ou receiver (push) | Push via `remote_write` |
| **Escalabilidade horizontal** | Vertical / sharding manual | Boa, mas com mais partes independentes para operar | Todos os componentes escalam horizontalmente; testado a 1 bilhão de séries ativas |
| **Performance de consulta** | Ótima no escopo local | Boa; store gateways podem ser gargalo em janelas longas | Query sharding e split paralelizam consultas longas |
| **Compactação** | Local | Compactor único por stream | Split-and-merge compactor, escala horizontalmente |
| **Complexidade operacional** | Baixa | Média (sidecars espalhados pelos clusters) | Média (componentes centralizados; modo single-binary para começar) |

**E por que o Mimir é hoje considerado a melhor opção para escala?** Ele herdou a arquitetura do Cortex — maduro e battle-tested em produção na Grafana Labs — e corrigiu as duas maiores dores dos antecessores: consultas longas lentas (resolvidas com query sharding, que divide uma query grande em dezenas de execuções paralelas) e a compactação como gargalo (o split-and-merge compactor escala horizontalmente, o que permitiu o teste público de 1 bilhão de séries ativas). Some a isso multi-tenancy nativa com limites por tenant, replicação embutida sem depender de deduplicação na leitura e uma arquitetura centralizada via `remote_write` — mais simples de operar do que espalhar sidecars do Thanos por cada cluster. O Thanos ainda faz sentido quando você quer manter os dados dentro de cada cluster e federar a consulta; mas quando o objetivo é um backend central, multi-tenant e de alta escala, o Mimir é o estado da arte.

## Arquitetura

```
┌─────────┐   requisições    ┌─────────────┐
│ loadgen │ ───────────────> │  Shop App   │
└─────────┘                  │  (FastAPI)  │
                             └──────┬──────┘
                                    │ /metrics
                                    │ scrape
                             ┌──────┴──────┐  remote_write   ┌─────────┐
                             │ Prometheus  │ ──────────────> │  Mimir  │
                             └─────────────┘                 │ (store) │
                                                             └────┬────┘
                                        PromQL API /prometheus    │
                                                             ┌────┴────┐
                                                             │ Grafana │
                                                             └─────────┘
```

São cinco containers. O `loadgen` bate na Shop App sem parar; a app expõe suas métricas em `/metrics`; o Prometheus coleta (da app, do Mimir e de si mesmo) e envia tudo via `remote_write`; o Grafana nunca fala com o Prometheus — consulta o Mimir pela API compatível. Só o volume do Mimir precisa persistir.

## Estrutura do projeto

Tirando o código da aplicação, o lab é só configuração:

```
lab_mimir/
├── docker-compose.yml
├── app/main.py                                 # API FastAPI instrumentada (Shop App)
├── app/loadgen.py                              # gerador de carga
├── app/Dockerfile
├── mimir/mimir.yaml                            # config do Mimir (single-binary)
├── prometheus/prometheus.yml                   # scrape + remote_write
├── grafana/provisioning/datasources/mimir.yaml # datasource provisionado
├── grafana/provisioning/dashboards/dashboards.yaml
└── grafana/dashboards/shop.json                # dashboard da Shop App
```

O `docker-compose.yml` orquestra os cinco serviços. O único volume nomeado é o do Mimir — é ele quem guarda os dados de longo prazo; o Prometheus pode perder o TSDB local sem dor, já que tudo é replicado via `remote_write`:

```yaml
services:
  mimir:
    image: grafana/mimir:2.14.0
    container_name: mimir
    command: ["-config.file=/etc/mimir/mimir.yaml"]
    volumes:
      - ./mimir/mimir.yaml:/etc/mimir/mimir.yaml:ro
      - mimir-data:/data
    ports:
      - "9009:9009"

  prometheus:
    image: prom/prometheus:v2.54.1
    container_name: prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
    ports:
      - "9090:9090"
    depends_on:
      - mimir

  app:
    build: ./app
    container_name: shop-app
    ports:
      - "8000:8000"

  loadgen:
    build: ./app
    container_name: loadgen
    command: ["python", "loadgen.py"]
    environment:
      - APP_URL=http://app:8000
    depends_on:
      - app

  grafana:
    image: grafana/grafana:11.2.0
    container_name: grafana
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
    depends_on:
      - mimir

volumes:
  mimir-data:
```

Repare que `app` e `loadgen` usam a mesma imagem (`build: ./app`) — o que muda é só o comando: um sobe o servidor uvicorn, o outro roda o script de carga. Economiza um Dockerfile.

O login anônimo com role Admin no Grafana é confortável para laboratório — e, claro, impensável em qualquer ambiente real.

## Configurando o Mimir em modo single-binary

Em produção o Mimir roda como microsserviços (distributor, ingester, querier, compactor…), mas para estudo o modo single-binary (`-target=all`, o default) roda tudo num processo só. A config mínima:

```yaml
multitenancy_enabled: false

server:
  http_listen_port: 9009
  log_level: info

distributor:
  ring:
    kvstore:
      store: memberlist

ingester:
  ring:
    kvstore:
      store: memberlist
    replication_factor: 1

store_gateway:
  sharding_ring:
    replication_factor: 1

compactor:
  data_dir: /data/compactor
  sharding_ring:
    kvstore:
      store: memberlist

blocks_storage:
  backend: filesystem
  filesystem:
    dir: /data/blocks
  tsdb:
    dir: /data/tsdb
  bucket_store:
    sync_dir: /data/tsdb-sync

ruler:
  rule_path: /data/ruler-tmp

ruler_storage:
  backend: filesystem
  filesystem:
    dir: /data/ruler

limits:
  max_global_series_per_user: 0
```

Com `replication_factor: 1` e `memberlist` como kvstore dos rings, uma instância única sobe sem depender de Consul ou etcd. O `backend: filesystem` simula o object storage (S3, GCS…) num diretório local — perfeito para lab.

### Armadilha 1: o WAL do ingester fora do volume

O default de `blocks_storage.tsdb.dir` é `./tsdb` — um caminho **relativo, dentro do container**. Se você não o apontar explicitamente para dentro do volume (`/data/tsdb` acima), o WAL e o head block do ingester — até cerca de 2 horas de métricas ainda não compactadas para o bucket — somem a cada restart do container. O lab aparenta funcionar, mas perde dados silenciosamente.

Dica relacionada: evite chamar o diretório do bucket de `tsdb`. Ele é o "object storage", não o TSDB do ingester — nomear como `/data/blocks` deixa os papéis claros.

### Armadilha 2: a imagem é distroless — esqueça o healthcheck clássico

Minha primeira versão tinha um healthcheck com `wget` no `/ready`. Resultado: o container ficava `unhealthy` para sempre, porque a imagem `grafana/mimir` não tem `wget`, `curl` nem shell — é distroless, só o binário do Mimir. Com `depends_on: condition: service_healthy`, isso ainda **bloqueia** Prometheus e Grafana de subir.

A solução pragmática é não ter healthcheck e usar `depends_on` simples: o `remote_write` do Prometheus e o datasource do Grafana fazem retry até o Mimir responder.

## A aplicação de exemplo: a Shop App

Um lab que só coleta a métrica `up` prova pouco. Por isso adicionei uma API em FastAPI que simula um e-commerce simples — cinco produtos, listagem, detalhe e criação de pedidos — instrumentada com a biblioteca `prometheus-client`.

As métricas são as que a gente realmente olha no dia a dia:

```python
REQUESTS = Counter(
    "shop_http_requests_total",
    "Total de requisições HTTP",
    ["method", "endpoint", "status"],
)
LATENCY = Histogram(
    "shop_http_request_duration_seconds",
    "Duração das requisições HTTP",
    ["endpoint"],
    buckets=(0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5),
)
ORDERS = Counter("shop_orders_total", "Total de pedidos", ["status"])
ORDER_VALUE = Histogram("shop_order_value_brl", "Valor dos pedidos em BRL", ...)
STOCK = Gauge("shop_products_in_stock", "Unidades em estoque (todos os produtos)")
```

Três dos quatro tipos de métrica do Prometheus aparecem aí: contador (requisições e pedidos), histograma (latência e valor do pedido) e gauge (estoque, que sobe e desce conforme os pedidos entram e a reposição acontece).

A coleta acontece num middleware, que mede toda requisição sem poluir os handlers:

```python
@app.middleware("http")
async def metrics_middleware(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)
    route = request.scope.get("route")
    endpoint = route.path if route else request.url.path
    if endpoint not in ("/metrics", "/health"):
        REQUESTS.labels(request.method, endpoint, str(response.status_code)).inc()
        LATENCY.labels(endpoint).observe(time.perf_counter() - start)
    return response
```

Um detalhe que faz diferença: o label `endpoint` usa `route.path`, não `request.url.path`. A diferença é entre registrar `/products/{product_id}` (uma série) ou `/products/1`, `/products/2`, `/products/3`… (uma série por produto). Esse é o caminho mais curto para uma explosão de cardinalidade — o problema número um de quem opera métricas em escala, e justamente o que o Mimir cobra via limites por tenant.

Para o gráfico não ficar uma linha reta e sem graça, a app simula o mundo real: latência variável (uma gaussiana em torno de 80ms) e 5% de erros 500. O `loadgen` complementa gerando tráfego contínuo, incluindo requisições a um produto que não existe, só para render 404s:

```python
roll = random.random()
if roll < 0.50:
    client.get("/products")
elif roll < 0.85:
    # ids 1-6: o 6 não existe e gera 404s para o dashboard
    client.get(f"/products/{random.randint(1, 6)}")
else:
    client.post("/orders", json={...})
```

## Prometheus e Grafana: as duas pontas

No Prometheus, a única novidade em relação a um setup comum é o bloco `remote_write`, apontando para o endpoint de push do Mimir:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

remote_write:
  - url: http://mimir:9009/api/v1/push

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: mimir
    static_configs:
      - targets: ["mimir:9009"]

  - job_name: shop-app
    static_configs:
      - targets: ["app:8000"]
```

São três jobs: a aplicação, o Prometheus e o próprio Mimir — que também expõe `/metrics`. Monitorar o backend de monitoramento sai de graça aqui.

No Grafana, o datasource é provisionado por arquivo — nada de configurar na UI. O detalhe importante é a URL: o Mimir expõe a API compatível com Prometheus sob o prefixo `/prometheus`:

```yaml
apiVersion: 1

datasources:
  - name: Mimir
    uid: mimir
    type: prometheus
    access: proxy
    url: http://mimir:9009/prometheus
    isDefault: true
    editable: true
```

Fixar o `uid` parece detalhe, mas evita dor de cabeça: dashboards exportados referenciam o datasource pelo uid, e sem ele cada reprovisionamento gera um uid aleatório novo — quebrando os dashboards importados. Com o uid fixo em `mimir`, o `shop.json` do lab referencia o datasource com segurança.

![Lista de datasources do Grafana com o Mimir provisionado como default](/images/mimir-grafana-datasource.png)

*O datasource Mimir já aparece como default no primeiro acesso — nenhum clique na UI foi necessário.*

Os dashboards seguem a mesma lógica de provisionamento por arquivo:

```yaml
apiVersion: 1

providers:
  - name: default
    orgId: 1
    type: file
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```

O `updateIntervalSeconds: 30` é o detalhe que torna o ciclo de edição agradável: o Grafana relê os JSONs a cada 30 segundos, então dá para editar o dashboard no arquivo e ver o resultado sem reiniciar container nenhum.

## Subindo e validando o fluxo

Um comando sobe tudo (o `--build` compila a imagem da Shop App na primeira vez):

```bash
docker compose up -d --build
```

O ingester tem um warm-up de 15 segundos antes de aceitar escrita — é normal o `/ready` responder `Ingester not ready: waiting for 15s after being ready` logo após o start. Depois disso:

```bash
# 1. Mimir pronto?
curl http://localhost:9009/ready
# ready

# 2. As métricas estão chegando via remote_write?
# (consulta feita NO MIMIR, não no Prometheus)
curl 'http://localhost:9009/prometheus/api/v1/query?query=count(up)'
# {"status":"success",..."value":[...,"3"]}
```

O `count(up)` retornando `3` é a prova do fluxo completo: os três jobs de scrape (shop-app, prometheus e mimir) foram coletados pelo Prometheus, enviados via `remote_write` e consultados direto na API do Mimir.

Mais interessante que o `up`, porém, é perguntar ao Mimir sobre a aplicação:

```bash
# requisições/s por endpoint
curl -sG 'http://localhost:9009/prometheus/api/v1/query' \
  --data-urlencode 'query=sum by (endpoint) (rate(shop_http_requests_total[1m]))'

# latência p95
curl -sG 'http://localhost:9009/prometheus/api/v1/query' \
  --data-urlencode 'query=histogram_quantile(0.95, sum by (le) (rate(shop_http_request_duration_seconds_bucket[5m])))'
```

O `-G` com `--data-urlencode` não é frescura: PromQL é cheio de espaços, parênteses e colchetes, e jogar isso direto na query string entrega uma URL inválida — o `curl` manda assim mesmo e a resposta vem vazia, sem erro nenhum que explique o motivo. Já perdi tempo com isso.

Essas respostas vêm do Mimir, não do Prometheus — ou seja, a métrica saiu da aplicação, passou pelo scrape, atravessou o `remote_write` e está sendo servida pelo armazenamento central. É o pipeline inteiro provado com dois `curl`.

![Página de targets do Prometheus com os jobs shop-app, mimir e prometheus em estado UP](/images/mimir-prometheus-targets.png)

*Os três targets em estado UP na página `/targets` do Prometheus: a coleta está funcionando.*

A mesma pergunta, em forma de gráfico, no Explore do Grafana (`http://localhost:3000`):

![Grafana Explore mostrando requisições por segundo por endpoint, consultadas no datasource Mimir](/images/mimir-grafana-explore.png)

*Requisições/s por endpoint, respondidas pelo Mimir. Repare na legenda: `/products/{product_id}` é uma série só, não uma por produto.*

E o dashboard da Shop App é provisionado junto, sem nenhum clique na UI — requisições por endpoint, latência p50/p95, taxa de erro, estoque, pedidos por minuto e ticket médio, tudo lendo do Mimir:

![Dashboard Shop App no Grafana com seis painéis de métricas da aplicação](/images/mimir-shop-dashboard.png)

*O dashboard provisionado em `grafana/dashboards/shop.json`, com refresh de 10s. A taxa de erro gira em torno dos 5% simulados na aplicação.*

Para conferir que os dados do ingester estão mesmo persistindo no volume (a Armadilha 1 resolvida):

```bash
docker run --rm -v lab_mimir_mimir-data:/data alpine ls /data
# blocks  compactor  tsdb  tsdb-sync

docker run --rm -v lab_mimir_mimir-data:/data alpine ls /data/tsdb
# anonymous
```

O `tsdb/anonymous` é o TSDB do ingester para o tenant "anonymous" — o nome que o Mimir usa para o tenant único quando `multitenancy_enabled: false`. Em `blocks/` ficam os blocos já compactados, o equivalente ao que iria para o S3 num ambiente real.

## Conclusão e próximos passos

Com um punhado de arquivos de configuração e uma API de brinquedo, o lab reproduz a arquitetura essencial de métricas de longo prazo: aplicação instrumentada, Prometheus como coletor, Mimir como armazenamento central e Grafana como camada de visualização. As duas armadilhas — o `tsdb.dir` fora do volume e o healthcheck impossível na imagem distroless — são o tipo de detalhe que só aparece quando se valida o fluxo de verdade, não só quando os containers sobem.

A aplicação de exemplo mudou o lab de lugar: em vez de olhar a métrica `up` e concluir que "está funcionando", dá para fazer as perguntas que a gente realmente faz em produção — qual é a latência p95, quanto da carga está falhando, o que acontece com o ticket médio — e ver cada uma delas sendo respondida pelo Mimir.

Alguns caminhos naturais para evoluir o lab:

- **Multi-tenancy**: habilitar `multitenancy_enabled: true` e enviar o header `X-Scope-OrgID` no `remote_write` e no datasource, simulando múltiplos times ou clientes.
- **Object storage real**: trocar o backend `filesystem` por um [MinIO](https://min.io/) no próprio compose, aproximando o lab de um deploy com S3.
- **Ruler**: mover as regras de gravação e alerta do Prometheus para o ruler do Mimir, centralizando a avaliação.
- **Modo microsserviços**: quebrar o single-binary em componentes separados para entender o papel de cada um sob carga.

O código completo do laboratório está no meu GitHub: [oalvaroaqui-git/lab_mimir](https://github.com/oalvaroaqui-git/lab_mimir). Referências: [documentação do Grafana Mimir](https://grafana.com/docs/mimir/latest/) e [remote_write do Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write).
