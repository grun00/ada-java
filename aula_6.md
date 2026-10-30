# Template de System Design: Aula 06 - Observabilidade & Diagnóstico em Produção

> **Contexto do Cenário: O Checkout que Trava sem Dar Erro**
>
> Sua equipe responde pelo serviço de **Checkout** do marketplace, que chama **Pagamento** e **Estoque**. Os três rodam em imagens *Distroless*.
>
> 1. **O Incidente:** Na abertura da Black Friday, com **5.000 TPS**, uma versão nova do Checkout entrou em *Canary*. Minutos depois, os painéis continuavam verdes, com latência média de 120 ms, CPU abaixo de 40% e nenhum pico de HTTP 500.
> 2. **O Impacto:** A conversão despencou. Milhares de reclamações chegaram ao suporte, todas com a mesma cena. O cliente clica em "Finalizar Pedido", a tela fica carregando e o aplicativo desiste por timeout.
> 3. **As Restrições:** A imagem não tem `sh` e não há SSH nos servidores. Cada serviço escreve logs em texto livre, do seu jeito, sem um identificador que ligue a chamada do Checkout à chamada que ele fez ao Pagamento. E dado de cliente que vai parar em log continua sujeito à LGPD.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Logs e Correlação

- **Campos do Log:** Que campos todo log do Checkout precisa ter para você encontrar, no meio de milhares de linhas, as da requisição que travou? Liste os campos e diga para que serve cada um.
  > **Formato:** log estruturado em JSON, uma linha por evento, no mesmo formato nos três serviços. No Spring Boot 3.4+ isso é nativo (`logging.structured.format.console=ecs` ou `logstash`), sem appender customizado. O texto livre de hoje impede filtrar por campo.
  >
  > | Campo | Para que serve |
  > |---|---|
  > | `@timestamp` (ISO-8601, UTC, ms) | Ordenar eventos de serviços diferentes na mesma linha do tempo |
  > | `log.level` | Filtrar `WARN`/`ERROR` sem perder o contexto `INFO` da mesma requisição |
  > | `service.name` | Saber se a linha veio do Checkout, Pagamento ou Estoque |
  > | `service.version` (+ SHA do commit) | **Separar canary e estável**: o incidente começou com a versão nova (Aula 05) |
  > | `deployment.environment`, `k8s.pod.name` | Isolar um pod com problema (pool esgotado, GC) |
  > | `trace_id` | **Juntar todas as linhas da mesma requisição** nos três serviços |
  > | `span_id` | Saber em qual operação (chamada ao Pagamento, query, publicação Kafka) a linha foi escrita |
  > | `event.action` | Nome estável do evento (`pagamento.chamada.inicio`, `pagamento.chamada.timeout`), para buscas e contagens |
  > | `http.request.method`, `http.route`, `http.response.status_code` | Rota no formato de template (`/pedidos/{id}`), nunca a URL com valores |
  > | `event.duration` (ms) | Encontrar as requisições lentas mesmo quando terminam com 200 |
  > | `order.id`, `saga.state` | Seguir o pedido pela Saga (Aula 04) e saber em que passo ele parou |
  > | `idempotency.key_hash` | Ligar retries do aplicativo à mesma tentativa de compra (Aula 02) |
  > | `peer.service`, `timeout.ms` | Em chamadas externas: quem foi chamado e com qual limite de tempo |
  > | `error.type`, `error.message`, `error.stack_trace` | Exceção em campos próprios, e não espalhada em várias linhas |
  >
  > Além disso, registro **o início e o fim** das chamadas ao Pagamento e ao Estoque. Uma requisição travada não tem linha de fim: a busca por `event.action=pagamento.chamada.inicio` sem o `fim` correspondente no mesmo `trace_id` mostra exatamente onde ela parou.
- **O Caminho do Contexto:** Como o mesmo identificador de rastreamento passa do Checkout para o Pagamento e o Estoque numa chamada HTTP?
  > **Padrão: W3C Trace Context.** Cada chamada HTTP leva o cabeçalho `traceparent: 00-<trace_id 32 hex>-<span_id 16 hex>-<flags>`. O `flags` informa se a requisição foi amostrada (seção 3). O cabeçalho `baggage` pode levar poucos valores não sensíveis, como `canary=true`.
  >
  > **Como funciona no Spring Boot 3:**
  > 1. **Entrada:** o API Gateway ou o próprio Checkout cria o `trace_id` se ele não vier no cabeçalho. O filtro de observação do Spring MVC/WebFlux (Micrometer Observation) lê o `traceparent` e abre o span do servidor.
  > 2. **Dentro do processo:** o Micrometer Tracing com a ponte `micrometer-tracing-bridge-otel` guarda o contexto e o copia para o MDC do Logback. Assim, `trace_id` e `span_id` entram em todo log automaticamente, sem código manual.
  > 3. **Saída:** o `RestClient` ou `WebClient` **criado a partir do `Builder` injetado pelo Spring** já vem instrumentado e escreve o `traceparent` com um novo `span_id` filho. Um `new RestTemplate()` criado à mão não propaga nada; é o erro mais comum, e o teste da seção 4 o detecta.
  > 4. **Do outro lado:** Pagamento e Estoque usam a mesma configuração, continuam o mesmo `trace_id` e respondem com ele nos logs.
  >
  > **Dois caminhos, o mesmo `trace_id` (Aula 04):** as consultas síncronas do Checkout ao Pagamento e ao Estoque levam o `traceparent` no cabeçalho HTTP; os comandos da Saga o levam nos headers do Kafka. O trace de um checkout cobre os dois caminhos.
  >
  > **Pontos que costumam quebrar a cadeia:**
  > - **Threads trocadas:** `@Async`, `CompletableFuture` e executores próprios precisam de `ContextPropagatingTaskDecorator` (ou `ContextSnapshot` da `context-propagation`). Com virtual threads, o mesmo cuidado vale para `Executors.newVirtualThreadPerTaskExecutor()` criado à mão. No WebFlux, `Hooks.enableAutomaticContextPropagation()`.
  > - **Kafka:** a Saga e a Outbox (Aula 04) seguem assíncronas. Com `observation-enabled=true` no `KafkaTemplate` e no listener, o `traceparent` vai nos headers da mensagem. Na Outbox, gravo o `traceparent` numa coluna e o Debezium o devolve como header, para o trace não morrer no banco.
  > - **Resposta ao cliente:** devolvo o `trace_id` num cabeçalho (`X-Trace-Id`) e o aplicativo o mostra na tela de erro. O suporte recebe o identificador junto com a reclamação e vai direto ao trace.
- **LGPD nos Logs:** Quais dados de cliente ficam fora de logs e traces? Para os identificadores que precisam aparecer, qual tratamento você aplica e quem pode ler esses registros?
  > **Princípio: minimização (art. 6º, III, da LGPD).** Log serve para diagnóstico. Só entra o que ajuda a encontrar o problema.
  >
  > **Nunca entram em logs, traces nem labels:**
  > - Dados de cartão (número, CVV, validade), que também são proibidos pelo PCI-DSS, e chave Pix.
  > - CPF, nome, e-mail, telefone, endereço e data de nascimento.
  > - Senhas, tokens JWT, cabeçalho `Authorization`, cookies e chaves de API do PSP.
  > - Corpo completo de requisição ou resposta e query string. O span HTTP registra a rota em template (`/clientes/{id}`), não a URL real.
  >
  > **Identificadores que precisam aparecer:**
  > - `order.id` e `trace_id` não identificam a pessoa sozinhos e ficam em claro.
  > - `customer.id` vira **pseudônimo com HMAC-SHA256** (chave guardada no Vault e rotacionada). Para investigar uma reclamação, o suporte calcula o HMAC do cliente e busca por ele, sem que o log guarde o identificador real. Hash simples (SHA-256 sem chave) não basta, porque CPF tem poucas combinações e pode ser descoberto por força bruta.
  > - IP do cliente com o último octeto zerado (`200.150.10.0`), quando necessário para antifraude.
  >
  > **Como garantir na prática (camadas):**
  > 1. **No código:** logo só campos de uma lista permitida. DTOs com dado pessoal sobrescrevem `toString()` (um `record` gera `toString()` com todos os campos, então preciso sobrescrever) e nunca são logados inteiros.
  > 2. **No Logback:** um conversor de máscara com regex (CPF, cartão, e-mail) funciona como rede de segurança para o que escapar.
  > 3. **No OpenTelemetry Collector:** os processadores `attributes`/`redaction` apagam atributos fora da lista permitida antes de enviar ao backend.
  > 4. **Na esteira:** teste automatizado que falha se um log contiver padrões de CPF ou cartão (seção 4).
  >
  > **Quem lê:** acesso por perfil (RBAC) no backend de observabilidade. O time do Checkout e o plantão leem os logs do próprio serviço; o suporte vê apenas o trace pelo `trace_id`. Todo acesso fica registrado. Retenção curta (ex.: 30 dias para logs detalhados, 7 dias para traces), com exclusão automática.

---

## 2. Métricas

- **Visão do Cliente (RED):** Quais métricas do Checkout entram no painel? Explique como você mede a duração para que o travamento de uma parte dos clientes não desapareça na média.
  > **Por que o painel ficou verde:** a métrica `http.server.requests` do Spring só registra a requisição **quando ela termina**. As requisições travadas ainda não tinham terminado, ou o aplicativo desistiu antes, então não entraram na conta. A média de 120 ms mediu só quem foi atendido. Além disso, uma média esconde a cauda: se 95% levam 100 ms e 5% levam 30 s, a média ainda parece aceitável.
  >
  > **Métricas RED do endpoint `POST /checkout` (separadas por versão estável × canary):**
  > - **Rate:** requisições por segundo e, principalmente, **pedidos concluídos por segundo**, comparados com o mesmo horário da semana anterior. A queda de conversão do incidente aparece aqui primeiro.
  > - **Errors:** 5xx **mais** timeouts e cancelamentos. Conto como erro a requisição que passou do SLO e a que o cliente abandonou (`outcome=CLIENT_ERROR`/conexão encerrada). Também conto os erros das chamadas que o Checkout faz (`http.client.requests` com `TimeoutException`).
  > - **Duration:** **histograma**, não média.
  >   - `management.metrics.distribution.percentiles-histogram.http.server.requests=true` gera os buckets no Prometheus, e calculo `histogram_quantile(0.99, ...)` agregando todos os pods. Percentil calculado em cada pod não pode ser somado nem tirado a média.
  >   - Buckets de SLO (`slo.http.server.requests=300ms,1s,3s,10s`) para medir a **fração** de requisições abaixo de 1 s, que vira o SLI: "99,5% dos checkouts em menos de 1 s".
  >   - O painel mostra p50, p95, p99 e p99,9 lado a lado. A diferença entre p50 e p99 revela que uma parte dos clientes está travada.
  > - **Requisições em andamento:** `http.server.requests.active` (um `LongTaskTimer`) mostra quantas estão abertas agora e há quanto tempo a mais antiga espera. Essa métrica **não depende de a requisição terminar** e teria subido no incidente.
  >
  > **Alertas por SLO com *burn rate*:** alerta quando o orçamento de erro está sendo gasto rápido (ex.: 14× em 5 min e 1 h). Isso cobre lentidão e timeout, e não só HTTP 500. O mesmo critério entra na análise do Argo Rollouts (Aula 05), para que o canary seja revertido sozinho.
- **Visão dos Recursos (USE):** Que recursos você monitora? Que sinal acusaria saturação antes do primeiro HTTP 500?
  > **Hipótese do incidente:** CPU abaixo de 40% e nenhum erro indicam que as threads estavam **esperando**, e não trabalhando. Por isso monitoro filas e pools, além de CPU.
  >
  > | Recurso | Utilização | Saturação (sinal antecipado) | Erros |
  > |---|---|---|---|
  > | Pool HikariCP | `hikaricp.connections.active` / `max` | **`hikaricp.connections.pending`** e `hikaricp.connections.acquire` (p99) | `hikaricp.connections.timeout` |
  > | Pool de conexões HTTP para o Pagamento e o Estoque | conexões em uso / máximo | **pedidos esperando conexão livre** (`httpcomponents.httpclient.pool.total.pending`) | timeouts de conexão e de leitura |
  > | Conexões HTTP de entrada e virtual threads (Aula 02) | `tomcat.connections.current` / `tomcat.connections.config.max` (com `server.tomcat.mbeanregistry.enabled=true`) | `http.server.requests.active` crescendo; carriers presas (`jdk.VirtualThreadPinned` no JFR) | conexões recusadas com a fila `accept-count` cheia |
  > | Bulkhead e Circuit Breaker (Resilience4j, Aula 02) | chamadas simultâneas | `resilience4j.bulkhead.available.concurrent.calls` perto de 0 | chamadas rejeitadas, estado do circuito |
  > | JVM | heap usado, CPU | pausas de GC (`jvm.gc.pause`), memória perto do limite do pod | `OutOfMemoryError`, reinícios do pod |
  > | Kafka e Saga | — | **lag do consumidor**, pedidos parados no mesmo estado além do prazo | mensagens na DLQ |
  > | PostgreSQL | conexões, CPU | locks esperando (`pg_locks`), consultas longas | deadlocks |
  >
  > **Sinais que acusariam o problema antes do primeiro 500:** `hikaricp.connections.pending` > 0 por mais de 1 minuto, pedidos esperando conexão HTTP para o Pagamento, `http.server.requests.active` crescendo com a mais antiga passando de 5 s, e bulkhead sem vagas. Todos esses sobem enquanto o servidor ainda responde 200 para quem consegue passar.
  >
  > **Diagnóstico sem `sh` (Distroless):** o Actuator expõe `/actuator/threaddump` e `/actuator/heapdump` em porta de gerência interna (`management.server.port`), protegida e acessível só pela rede do cluster. Com ele, vejo em qual chamada as threads estão bloqueadas. O JFR fica ligado continuamente (`-XX:StartFlightRecording`, com baixo custo) e o `kubectl debug` com container efêmero (Aula 05) cobre o resto. A causa provável aqui é uma chamada ao Pagamento **sem timeout**, que segura as threads indefinidamente; então todo cliente HTTP tem `connectTimeout` e `readTimeout` explícitos e menores que o timeout do aplicativo.
- **Labels:** Quais labels você permite nessas métricas e quais proíbe? Diga o efeito de cada escolha no servidor de métricas.
  > **Regra:** cada combinação de valores de label cria uma **série temporal** nova no Prometheus. O número de séries é o produto da quantidade de valores de cada label. Por isso, só entram labels com poucos valores, conhecidos de antemão.
  >
  > **Permitidas (cardinalidade baixa e limitada):**
  > | Label | Valores aprox. | Efeito |
  > |---|---|---|
  > | `service`, `env` | 3 e 3 | Separar serviço e ambiente |
  > | `version` | 2 durante o canary | **Comparar canary × estável**; sem ela, o defeito do canary se dilui nos 95% estáveis |
  > | `uri` (template, ex.: `/pedidos/{id}`) | dezenas | Latência por rota |
  > | `method`, `status`, `outcome` | poucos | Separar sucesso, erro do cliente, erro do servidor |
  > | `exception` (classe) | dezenas | Distinguir `TimeoutException` de `SQLException` |
  > | `peer_service` / `client.name` | 2 (Pagamento, Estoque) | Saber qual dependência está lenta |
  > | `pod` | dezenas | Isolar um pod; adicionado pelo Prometheus, e removido nas agregações de painel |
  >
  > **Proibidas (cardinalidade ilimitada):** `customer_id`, `order_id`, `trace_id`, `idempotency_key`, e-mail, CPF, URL com valores (`/pedidos/123`) e mensagem de erro em texto livre.
  > - **Efeito:** cada pedido criaria séries novas. A 5.000 TPS seriam milhões de séries por hora: memória do Prometheus esgotada, consultas lentas, custo alto e, no caso de CPF ou e-mail, dado pessoal num sistema sem controle de acesso por registro (violação da LGPD).
  > - **Onde fica essa informação:** em logs e traces, que foram feitos para valores únicos. A ligação entre os dois mundos é feita por **exemplars**: o histograma do Prometheus guarda o `trace_id` de algumas amostras de cada bucket, e no Grafana eu clico no ponto do p99 e abro o trace daquela requisição lenta.
  >
  > **Proteção automática:** um `MeterFilter` no Micrometer (`maximumAllowableTags`) limita os valores de `uri`, e `sample_limit` no scrape do Prometheus recusa um alvo que exploda em séries. Uma rota desconhecida vira `uri=UNKNOWN`, e não uma série por URL.

---

## 3. Traces e Amostragem

- **A Escolha:** ( ) Head Sampling  ( ) Tail Sampling  (x) Combinação das duas
- **Justificativa:** Guardar 100% dos traces a 5.000 TPS custa caro. Qual é a sua regra de retenção, e como ela evita que justamente as requisições que travaram sejam descartadas?
  > **Problema do head sampling sozinho:** a decisão é tomada no início da requisição, antes de saber se ela vai travar. Com 1% de amostragem, 99% das requisições travadas seriam descartadas, e justamente o caso raro que precisamos investigar teria grande chance de sumir.
  >
  > **Desenho combinado:**
  > 1. **Head sampling na origem, só para cortar ruído:** no SDK, com `management.tracing.sampling.probability=1.0` e um `Sampler` customizado do OpenTelemetry (*parent-based*) que decide pela rota, o `POST /checkout` e as chamadas da Saga são gerados a **100%**. Rotas de pouco valor (health checks, `/actuator/*`, consulta de catálogo) ficam em 1% ou são excluídas. Isso reduz o volume que chega ao Collector sem perder nenhum checkout.
  > 2. **Tail sampling no OpenTelemetry Collector:** os spans chegam a um Collector *gateway*. Na frente dele, o `loadbalancing` exporter envia todos os spans do mesmo `trace_id` para a mesma instância, para que a decisão seja tomada sobre o trace completo. O processador `tail_sampling` espera o trace terminar (`decision_wait`) e aplica as políticas:
  >    - **Guardar 100%** dos traces com erro (`status_code = ERROR`), incluindo timeouts e cancelamentos.
  >    - **Guardar 100%** dos traces com duração acima de **1 s** (o limite do SLO).
  >    - **Guardar 100%** dos traces da versão canary durante o rollout (atributo `service.version`) e dos que passaram por compensação da Saga.
  >    - **Guardar 1% a 5%** dos traces normais, como base de comparação do que é "saudável".
  >
  > **Custo:** no tráfego normal, quase tudo é rápido e com sucesso, então o backend recebe cerca de 2% a 5% dos traces, mais os problemáticos. O custo maior fica na memória do Collector, que guarda os traces durante a espera. Dimensiono pela fórmula `TPS × spans por trace × decision_wait` e escalo o Collector horizontalmente.
  >
  > **Como as requisições travadas não se perdem:**
  > - Uma requisição travada **não fecha o span**, e um span aberto não é exportado. Por isso, todo ponto de espera tem timeout explícito (cliente HTTP, `TimeLimiter` da Aula 02, timeout de transação e `request-timeout` do servidor) **menor que o `decision_wait`**. Exemplo (orçamento da Aula 02): consultas síncronas com timeout de 1 s, `POST /checkout` com 3 s, chamada ao PSP com prazo total de 10 s e `decision_wait` de 30 s. Quando o tempo esgota, o span fecha com `ERROR` e duração alta, e as duas políticas de 100% o guardam.
  > - O cancelamento pelo cliente também fecha o span: o Spring marca a observação como encerrada quando a conexão cai, e registro o evento `client.aborted`.
  > - Os logs têm o `trace_id` em todas as linhas e não passam por amostragem de traces, então mesmo um trace descartado pode ser reconstruído pelos logs.

---

## 4. Estratégia de Testes

- **Qual a nossa estratégia para testar isso e garantir que funciona?** Antes da próxima Black Friday, como você prova que os painéis e os alertas mostrariam este incidente?
  > **Objetivo:** reproduzir de propósito um "travamento sem erro" e provar que logs, métricas, traces e alertas o mostram em poucos minutos, sem depender de alguém olhando o painel.
  >
  > **1. Testes automatizados na esteira (a cada PR):**
  > - **Propagação do trace:** teste de integração com Testcontainers e WireMock no lugar do Pagamento. Chamo o Checkout e verifico que a requisição recebida pelo WireMock tem `traceparent` com o **mesmo `trace_id`** da requisição de entrada. Esse teste pega o `RestTemplate` criado à mão (seção 1).
  > - **Contrato do log:** capturo a saída com `OutputCaptureExtension` e valido que cada linha é JSON com `trace_id`, `span_id`, `service.version` e `event.action`.
  > - **LGPD:** o mesmo teste processa um pedido com CPF, e-mail e cartão fictícios e falha se algum desses valores, ou um padrão de CPF ou cartão, aparecer nos logs ou nos spans exportados (`InMemorySpanExporter` do OpenTelemetry).
  > - **Timeout:** WireMock com `withFixedDelay(30_000)` na pré-validação do Pagamento. O teste confirma que a consulta desiste em cerca de 1 s, que o fallback deixa o pedido seguir (`202` em menos de 3 s), que o contador de timeout aumenta e que o span do cliente HTTP termina com `ERROR`.
  > - **Métricas e labels:** chamo `/actuator/prometheus` e confirmo que existem os buckets do histograma e `http.server.requests.active`, e que nenhuma métrica tem label `customer_id`, `order_id` ou URL com valores.
  > - **Regras de alerta:** testes unitários das regras com `promtool test rules`. Uma série sintética com 5% das requisições acima de 1 s precisa disparar o alerta de *burn rate*; uma série saudável não pode disparar.
  >
  > **2. Ensaio do incidente em staging (*game day*), antes da Black Friday:**
  > - Carga com k6 ou Gatling no volume da Black Friday, proporcional ao tamanho de staging.
  > - **Injeção de falha:** Toxiproxy (ou falha injetada no Istio) adiciona 30 s de atraso em 5% das chamadas do Checkout ao Pagamento, **sem erro**, exatamente como no incidente. Num segundo ensaio, o defeito vai numa versão canary pelo Argo Rollouts.
  > - **Critérios de aprovação**, medidos com cronômetro:
  >   - O alerta de SLO dispara em até 5 minutos, mesmo com latência média e CPU normais.
  >   - O painel mostra p99 alto, `http.server.requests.active` subindo e conexões pendentes no pool HTTP do Pagamento.
  >   - Pelo exemplar do p99, a pessoa de plantão chega ao trace e vê o span do Pagamento ocupando quase todo o tempo, sem abrir shell.
  >   - O trace da requisição lenta foi guardado pelo tail sampling, e a busca por `trace_id` nos logs retorna as linhas dos três serviços.
  >   - No ensaio com canary, a análise do Argo Rollouts executa o `abort` sozinha.
  > - O que falhar vira tarefa corrigida e o ensaio é repetido. O *runbook* do alerta é atualizado com os passos usados.
  >
  > **3. Em produção:**
  > - **Monitoramento sintético:** um robô faz checkouts de teste (com cliente e cartão de teste) a cada minuto, de fora do cluster. Ele mede o tempo visto pelo cliente, que pega o caso em que o servidor nem chega a registrar a requisição.
  > - **Teste do alerta ponta a ponta:** um alerta "sempre disparando" (*watchdog*/*dead man's switch*) confirma que Prometheus, Alertmanager e PagerDuty estão funcionando; se ele parar de chegar, o próprio sistema de alertas está quebrado.
