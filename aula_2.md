# Template de System Design: Aula 02 - APIs Resilientes

> **Instruções:** Faça uma cópia deste arquivo e preencha as seções abaixo com as conclusões da sua pesquisa e os debates. O foco é a integração da API de Checkout com o Serviço de Pagamento Externo para suportar 10.000 usuários simultâneos.

## 1. Escolha de Paradigma da API

- **A Escolha:** ( ) Spring MVC com Virtual Threads ou (x) Spring WebFlux (Reativo)?
- **Justificativa:** Considerando a capacidade de suportar 10k acessos concorrentes, por que você fez essa escolha técnica?
  > Escolhi o **Spring WebFlux** por três motivos:
  >
  > 1. **Poucas threads para muitas conexões:** o WebFlux usa um event loop (Netty) com aproximadamente uma thread por núcleo de CPU. As 10 mil requisições simultâneas ficam esperando a resposta do parceiro de pagamento sem ocupar uma thread cada, o que mantém o uso de memória baixo e previsível.
  > 2. **Backpressure:** o consumidor controla o ritmo em que recebe os dados. Se o parceiro de pagamento ficar lento, a aplicação consegue limitar e enfileirar a demanda de forma controlada (por exemplo, com `limitRate` e limites de concorrência em `flatMap`), em vez de acumular trabalho sem controle até esgotar a memória.
  > 3. **Stack reativa de ponta a ponta:** a chamada ao parceiro usa `WebClient`, a persistência usa R2DBC (Postgres) e o controle de idempotência usa Redis Reactive. Como nenhuma etapa bloqueia, o event loop nunca fica travado. Além disso, o Resilience4j tem operadores próprios para Reactor (`CircuitBreakerOperator`, `RetryOperator`, `BulkheadOperator`, `RateLimiterOperator`).
  >
  > **Trade-offs aceitos:** a curva de aprendizado é maior, o código com `Mono`/`Flux` é mais difícil de ler e debugar e não dá para usar JPA/JDBC. Para reduzir esses riscos, a regra é **nenhuma chamada bloqueante no event loop**, verificada com o BlockHound nos testes. Se algum trecho bloqueante for inevitável, ele roda em `Schedulers.boundedElastic()`.

## 2. Desenho de Resiliência

Dependemos de uma API terceira instável. Quais recursos do padrão de resiliência você vai utilizar?

- **Proteção contra Falhas e Lentidão:**
  Quais padrões (Circuit Breaker, TimeLimiter, RateLimiter, Bulkhead, Retry) você implementaria na chamada para o Parceiro de Pagamentos? Por que?
  > Vou usar **os cinco padrões** (com Resilience4j), cada um contra um tipo diferente de problema. Os valores abaixo são um ponto de partida e devem ser ajustados com testes de carga.
  >
  > | Padrão | Problema que resolve | Configuração inicial |
  > |---|---|---|
  > | **TimeLimiter** (timeout) | Parceiro lento prendendo conexões indefinidamente | `responseTimeout` de 3 s no `WebClient` + `timeout(3s)` no Reactor |
  > | **Circuit Breaker** | Continuar chamando um parceiro que já está fora do ar | Abre com 50% de falhas **ou** 50% de chamadas lentas (> 2 s) em uma janela de 100 chamadas; fica aberto 30 s; em seguida libera 10 chamadas de teste (half-open) |
  > | **Bulkhead** | Uma dependência lenta consumir todos os recursos da aplicação | Máximo de 200 chamadas simultâneas ao parceiro; as demais falham rápido |
  > | **RateLimiter** | Ultrapassar a cota contratada com o parceiro (e receber HTTP 429) | Limite igual à cota do contrato, por exemplo 500 req/s |
  > | **Retry** (com cuidado) | Falhas temporárias, como erro de rede ou HTTP 503 | No máximo 2 novas tentativas, com backoff exponencial e jitter (200 ms → 400 ms); **somente** para timeout, erro de conexão, 502/503/504 e 429; **nunca** para outros erros 4xx; **sempre com a mesma `Idempotency-Key`** |
  >
  > **Por que o Retry exige cuidado:** em pagamentos, um timeout não significa que a cobrança falhou. O parceiro pode ter cobrado e a resposta se perdido no caminho. Repetir a chamada só é seguro porque o parceiro recebe a mesma `Idempotency-Key` e não cobra duas vezes.
  >
  > **Ordem de aplicação** (de fora para dentro, padrão do Resilience4j): `Retry → CircuitBreaker → RateLimiter → TimeLimiter → Bulkhead → chamada ao parceiro`. Assim, cada nova tentativa passa de novo pelo Circuit Breaker, e quando o circuito está aberto a chamada falha imediatamente, sem novas tentativas.
  >
  > **Fallback:** quando o circuito está aberto ou o resultado é incerto (timeout depois de todas as tentativas), o sistema **não** confirma nem cancela a compra. O pedido fica com status `PAGAMENTO_PENDENTE`, a API responde `202 Accepted` e um processo de **reconciliação** (webhook do parceiro ou consulta periódica pelo status) define o resultado final. Se o circuito estiver aberto antes de qualquer tentativa de cobrança, a API responde `503` com o header `Retry-After`.

- **Proteção contra Duplicidade:**
  Como a `Idempotency-Key` será utilizada na requisição do Checkout para evitar cobranças duplas? Qual o ciclo de vida dessa chave?
  > **Uso:** o **cliente** (front-end ou app) gera um UUID v4 para cada **intenção de compra** e o envia no header `Idempotency-Key: <uuid>`. Se o usuário clicar duas vezes, a rede falhar ou o app reenviar a requisição, a chave continua a mesma. O servidor guarda a chave no **Redis** e repassa o mesmo valor ao parceiro de pagamento, que assim também não cobra duas vezes.
  >
  > **Ciclo de vida da chave:**
  >
  > 1. **Recebimento:** o servidor valida o header (obrigatório, formato UUID) e calcula um hash do corpo da requisição.
  > 2. **Reserva:** faz um `SET idem:{clienteId}:{chave} PROCESSANDO NX EX 86400` no Redis. A operação é atômica: só a primeira requisição consegue gravar.
  >    - **Chave nova:** segue para a cobrança.
  >    - **Chave já existe com status `PROCESSANDO`:** responde `409 Conflict` (outra requisição igual está em andamento).
  >    - **Chave já existe com status `CONCLUIDO`:** devolve a **mesma resposta salva** (status HTTP e corpo), sem cobrar de novo.
  >    - **Mesma chave com corpo diferente** (o hash não bate): responde `422 Unprocessable Entity`, porque é uso indevido da chave.
  > 3. **Processamento:** chama o parceiro com a mesma chave, passando pelos padrões de resiliência acima.
  > 4. **Conclusão:** atualiza o registro para `CONCLUIDO` com a resposta final, ou `PENDENTE` em caso de resultado incerto, mantendo o TTL.
  > 5. **Falha antes da cobrança:** se ocorrer um erro que comprovadamente aconteceu **antes** de chamar o parceiro (por exemplo, erro de validação), a chave é removida para permitir uma nova tentativa.
  > 6. **Expiração:** a chave expira após **24 horas** (TTL). Depois disso, o mesmo UUID seria tratado como uma nova compra; por isso, o cliente nunca reutiliza chaves.
  >
  > **Segurança extra:** a chave no Redis inclui o `clienteId`, para que um cliente não consiga colidir com a chave de outro. Além disso, o pedido tem uma restrição `UNIQUE` em `idempotency_key` no Postgres, como segunda barreira caso o Redis perca dados.

## 3. Os Edge Cases (Engenharia do Caos)

Cole aqui as 3 perguntas de risco que você formulou e defina como a arquitetura responde a cada uma delas.

1. **E se...** o parceiro **cobrar o cliente, mas a resposta se perder** no caminho (timeout ou queda de conexão)?
   - **Resposta do Sistema:** a API não sabe se houve cobrança, então **não pode assumir sucesso nem falha**.
     - O Retry repete a chamada com a **mesma `Idempotency-Key`**. Se a cobrança já tinha acontecido, o parceiro devolve o resultado original em vez de cobrar de novo.
     - Se todas as tentativas falharem, o pedido fica `PAGAMENTO_PENDENTE`, a chave fica `PENDENTE` no Redis e a API responde `202 Accepted`.
     - A **reconciliação** resolve o caso: o sistema recebe o webhook do parceiro ou consulta o status do pagamento pela chave (por exemplo, a cada 1 minuto, por até 24 h) e atualiza o pedido para `PAGO` ou `FALHOU`. O cliente é notificado do resultado.
     - Se a mesma requisição for reenviada nesse meio-tempo, recebe o status atual (`PENDENTE`), nunca uma nova cobrança.
2. **E se...** o **Redis** usado para controlar a idempotência **ficar fora do ar**?
   - **Resposta do Sistema:** a regra é **não cobrar sem garantia de idempotência** (fail closed: na dúvida, bloquear).
     - A API passa a usar a segunda barreira: a restrição `UNIQUE` em `idempotency_key` na tabela de pedidos do Postgres (via R2DBC). O pedido é inserido **antes** de chamar o parceiro; se a chave já existir, a inserção falha e a API devolve o pedido existente em vez de cobrar.
     - Se o Postgres também estiver indisponível, a API responde `503 Service Unavailable` com `Retry-After`, e o cliente reenvia depois com a mesma chave.
     - As chamadas ao Redis também têm timeout curto (por exemplo, 200 ms) e um Circuit Breaker próprio, para que a queda do Redis não deixe as requisições esperando.
     - Um alerta é disparado para o time de plantão, porque o sistema está funcionando com menos proteção.
3. **E se...** houver um **pico de 3x o tráfego** (Black Friday, 30 mil usuários) **ao mesmo tempo em que o parceiro fica lento**?
   - **Resposta do Sistema:** o sistema deve **degradar de forma controlada**, sem cair por completo.
     - O **TimeLimiter** encerra as chamadas lentas em 3 s, para que as conexões não se acumulem.
     - O **Bulkhead** (200 chamadas simultâneas) e o **RateLimiter** (cota do parceiro) recusam o excesso imediatamente, sem consumir recursos.
     - O **backpressure** do WebFlux, junto com um limite de conexões no Netty, impede que a fila de requisições cresça sem controle e esgote a memória.
     - As requisições recusadas recebem `503` ou `429` com `Retry-After`, e o front-end mostra "tente novamente em instantes", sem cobrar.
     - Se a lentidão passar do limite de chamadas lentas, o **Circuit Breaker** abre e protege o parceiro e a aplicação. Enquanto isso, o autoscaling sobe novas instâncias da API para atender o tráfego que não depende do parceiro.

## 4. Testando a Resiliência

- **Estratégia:** Como você vai garantir que o Circuit Breaker abre na hora certa e o Timeout não deixa a Thread presa?
  > A estratégia usa **testes em camadas**, do mais rápido e barato para o mais realista:
  >
  > | Camada | Ferramentas | O que valida |
  > |---|---|---|
  > | **Unitário** | JUnit 5 + Reactor `StepVerifier` (com tempo virtual) | Regras do Retry (quais erros repetem e quais não), Circuit Breaker abrindo nos limites configurados, timeout em 3 s |
  > | **Integração** | WireMock (simula o parceiro) + Testcontainers (Redis e Postgres reais) + BlockHound | Comportamento com latência e erros reais do parceiro, idempotência com requisições simultâneas, nenhuma chamada bloqueante no event loop |
  > | **Carga** | Gatling ou k6 | 10 mil usuários simultâneos, uso de memória e threads, tempos de resposta (p95/p99) |
  > | **Caos** | Toxiproxy + derrubar containers durante o teste de carga | Latência e queda de rede, queda do Redis e do parceiro (edge cases da seção 3) |
  >
  > **1. O Circuit Breaker abre na hora certa:**
  > - Configurar o WireMock para responder `500` em 60% das chamadas e enviar 100 requisições (o tamanho da janela).
  > - Verificar pelo `CircuitBreakerRegistry` que o estado mudou para `OPEN` depois da janela, e não antes.
  > - Verificar que, com o circuito aberto, as chamadas seguintes falham com `CallNotPermittedException` **sem chegar ao WireMock** (`verify(0, postRequestedFor(...))`).
  > - Avançar o tempo e verificar a passagem para `HALF_OPEN`: com o parceiro saudável, o circuito volta para `CLOSED`; com o parceiro ainda falhando, volta para `OPEN`.
  > - Repetir o teste com chamadas lentas (atraso de 2,5 s) para validar a regra de chamadas lentas.
  >
  > **2. O timeout não deixa a thread presa:**
  > - Simular **latência de 10 s** no parceiro com `withFixedDelay(10_000)` no WireMock (ou um *toxic* de latência no Toxiproxy).
  > - Verificar que a chamada termina com `TimeoutException` em aproximadamente **3 s**, e não em 10 s. No teste unitário, usar `StepVerifier.withVirtualTime` para isso não levar 3 s reais.
  > - Verificar que a conexão HTTP foi cancelada e devolvida ao pool (métricas do Reactor Netty).
  > - Rodar os testes com o **BlockHound** ativo: qualquer chamada bloqueante no event loop faz o teste falhar.
  > - No teste de carga com o parceiro lento, verificar pelo Micrometer/Actuator que o número de threads continua estável (aproximadamente o número de núcleos) e que a memória não cresce sem limite.
  >
  > **3. Critérios de aceite:**
  > - Nenhuma cobrança duplicada em 1.000 requisições simultâneas com a mesma `Idempotency-Key`: apenas 1 chamada chega ao WireMock e todas as respostas são iguais.
  > - Com 10 mil usuários e o parceiro saudável: p99 abaixo de 1 s e taxa de erro abaixo de 0,1%.
  > - Com o parceiro fora do ar: o Circuit Breaker abre em menos de 30 s, a API responde rápido (`503`) e nenhum pedido fica sem status final depois da reconciliação.
