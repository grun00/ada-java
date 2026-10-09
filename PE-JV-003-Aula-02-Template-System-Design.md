# Template de System Design: Aula 02 - APIs Resilientes

> **Instruções:** Faça uma cópia deste arquivo e preencha as seções abaixo com as conclusões da sua pesquisa e os debates. O foco é a integração da API de Checkout com o Serviço de Pagamento Externo para suportar 10.000 usuários simultâneos.

## 1. Escolha de Paradigma da API

- **A Escolha:** ( ) Spring MVC com Virtual Threads ou ( ) Spring WebFlux (Reativo)?
- **Justificativa:** Considerando a capacidade de suportar 10k acessos concorrentes, por que você fez essa escolha técnica?
  > [Sua resposta aqui]

## 2. Desenho de Resiliência

Dependemos de uma API terceira instável. Quais recursos do padrão de resiliência você vai utilizar?

- **Proteção contra Falhas e Lentidão:**
  Quais padrões (Circuit Breaker, TimeLimiter, RateLimiter, Bulkhead, Retry) você implementaria na chamada para o Parceiro de Pagamentos? Por que?
  > [Sua resposta aqui]

- **Proteção contra Duplicidade:**
  Como a `Idempotency-Key` será utilizada na requisição do Checkout para evitar cobranças duplas? Qual o ciclo de vida dessa chave?
  > [Sua resposta aqui]

## 3. Os Edge Cases (Engenharia do Caos)

1. **E se...** o parceiro **cobrar o cliente, mas a resposta se perder** no caminho (timeout ou queda de conexão)? **Resposta do Sistema:** a API não sabe se houve cobrança, então **não pode assumir sucesso nem falha**.
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
  > [Sua resposta aqui. Exemplo: simular latência de 10s no parceiro usando XYZ...]
