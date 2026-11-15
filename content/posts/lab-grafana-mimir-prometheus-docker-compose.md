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

Neste post, monto um laboratório local completo com Docker Compose: Prometheus coletando e enviando métricas, Mimir armazenando em modo single-binary e Grafana consultando tudo — além de duas armadilhas reais que encontrei no caminho.

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
┌────────────┐  remote_write   ┌─────────┐   PromQL API    ┌─────────┐
│ Prometheus │ ──────────────> │  Mimir  │ <────────────── │ Grafana │
│  (scrape)  │                 │ (store) │  /prometheus    │  (viz)  │
└────────────┘                 └────┬────┘                 └─────────┘
                                    │
                               mimir-data
                               (volume)
```

O Prometheus coleta e envia tudo via remote_write; o Grafana nunca fala com o Prometheus — consulta o Mimir pela API compatível, e só o volume do Mimir precisa persistir.

## Estrutura do projeto

O lab inteiro cabe em quatro arquivos de configuração:

```
lab_mimir/
├── docker-compose.yml
├── mimir/mimir.yaml                            # config do Mimir (single-binary)
├── prometheus/prometheus.yml                   # scrape + remote_write
└── grafana/provisioning/datasources/mimir.yaml # datasource provisionado
```

O `docker-compose.yml` orquestra os três serviços. O único volume nomeado é o do Mimir — é ele quem guarda os dados de longo prazo; o Prometheus pode perder o TSDB local sem dor, já que tudo é replicado via `remote_write`:

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

  grafana:
    image: grafana/grafana:11.2.0
    container_name: grafana
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
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
```

O próprio Mimir expõe métricas em `/metrics`, então ele também vira alvo de scrape — monitorar o backend de monitoramento é de graça aqui.

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

Fixar o `uid` parece detalhe, mas evita dor de cabeça: dashboards exportados referenciam o datasource pelo uid, e sem ele cada reprovisionamento gera um uid aleatório novo — quebrando os dashboards importados.

![Lista de datasources do Grafana com o Mimir provisionado como default](/images/mimir-grafana-datasource.png)

*O datasource Mimir já aparece como default no primeiro acesso — nenhum clique na UI foi necessário.*

## Subindo e validando o fluxo

Um comando sobe tudo:

```bash
docker compose up -d
```

O ingester tem um warm-up de 15 segundos antes de aceitar escrita — é normal o `/ready` responder `Ingester not ready: waiting for 15s after being ready` logo após o start. Depois disso:

```bash
# 1. Mimir pronto?
curl http://localhost:9009/ready
# ready

# 2. As métricas estão chegando via remote_write?
# (consulta feita NO MIMIR, não no Prometheus)
curl 'http://localhost:9009/prometheus/api/v1/query?query=count(up)'
# {"status":"success",..."value":[...,"2"]}
```

O `count(up)` retornando `2` é a prova do fluxo completo: os dois jobs de scrape (prometheus e mimir) foram coletados pelo Prometheus, enviados via `remote_write` e consultados direto na API do Mimir.

![Página de targets do Prometheus com os jobs mimir e prometheus em estado UP](/images/mimir-prometheus-targets.png)

*Os dois targets em estado UP na página `/targets` do Prometheus: a coleta está funcionando.*

Por fim, o Grafana em `http://localhost:3000` já abre com o datasource **Mimir** como default — qualquer query PromQL no Explore consulta o Mimir, não o Prometheus.

![Grafana Explore consultando a métrica up no datasource Mimir, com as duas séries no gráfico](/images/mimir-grafana-explore.png)

*A query `up` no Explore, respondida pelo Mimir: as duas séries chegaram via remote_write.*

Para conferir que os dados do ingester estão mesmo persistindo no volume (a Armadilha 1 resolvida):

```bash
docker run --rm -v lab_mimir_mimir-data:/data alpine ls /data
# blocks  tsdb
```

O `tsdb/anonymous` é o TSDB do tenant "anonymous" — o nome que o Mimir usa para o tenant único quando `multitenancy_enabled: false`.

## Conclusão e próximos passos

Com quatro arquivos de configuração, o lab reproduz a arquitetura essencial de métricas de longo prazo: Prometheus como coletor, Mimir como armazenamento central e Grafana como camada de visualização. As duas armadilhas — o `tsdb.dir` fora do volume e o healthcheck impossível na imagem distroless — são o tipo de detalhe que só aparece quando se valida o fluxo de verdade, não só quando os containers sobem.

Alguns caminhos naturais para evoluir o lab:

- **Multi-tenancy**: habilitar `multitenancy_enabled: true` e enviar o header `X-Scope-OrgID` no `remote_write` e no datasource, simulando múltiplos times ou clientes.
- **Object storage real**: trocar o backend `filesystem` por um [MinIO](https://min.io/) no próprio compose, aproximando o lab de um deploy com S3.
- **Ruler**: mover as regras de gravação e alerta do Prometheus para o ruler do Mimir, centralizando a avaliação.
- **Modo microsserviços**: quebrar o single-binary em componentes separados para entender o papel de cada um sob carga.

O código completo do laboratório está no meu GitHub: [oalvaroaqui-git/lab_mimir](https://github.com/oalvaroaqui-git/lab_mimir). Referências: [documentação do Grafana Mimir](https://grafana.com/docs/mimir/latest/) e [remote_write do Prometheus](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#remote_write).
