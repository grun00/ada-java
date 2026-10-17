# Template de System Design: Aula 04 - Microsserviços & Padrão Saga

> **Contexto do Cenário: O Pedido Pago Sem Produto na Black Friday**
>
> Sua equipe está arquitetando a transação distribuída de compra na Black Friday para o marketplace:
>
> 1. **O Incidente:** Um cliente comprou uma Smart TV de R$ 3.000 via Pix. O gateway de pagamentos debitou o valor imediatamente e confirmou a transação. No milissegundo seguinte, o serviço de Estoque esgotou as unidades e rejeitou a reserva. O dinheiro sumiu da conta do cliente, o produto não existe para entrega e o pedido travou em um estado inconsistente.
> 2. **O Desafio Distribuído:** Não existe transação única (`@Transactional`) entre bancos de dados autônomos de Checkout, Pagamento e Estoque.
> 3. **A Carga:** 5.000 TPS de checkout simultâneo e milhões de clientes consultando o status da compra em "Meus Pedidos".
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Transações Distribuídas: Coordenação e Trade-offs

### 1.1 Volumetria e Pragmatismo: 2PC vs Saga

- **Análise Crítica:** Em quais cenários arquiteturais uma transação ACID local ou Two-Phase Commit (2PC) resolveria o problema com menor custo operacional? Por que, para o cenário de 5.000 TPS na Black Friday com serviços e bancos autônomos, o 2PC é inviável e a consistência eventual se torna necessária?
  > **Resumo:** o 2PC garante consistência imediata, mas cobra caro em bloqueio e disponibilidade. Neste cenário, esse custo é alto demais.
  >
  > **Quando ACID local ou 2PC resolveria com menor custo:**
  > - **Um único banco (monólito modular):** se Checkout, Pagamento e Estoque usassem o mesmo PostgreSQL, como na Aula 03, um `@Transactional` local bastaria: pedido, reserva e registro do pagamento são gravados juntos, ou nada é gravado. É a solução mais simples e deve ser a primeira considerada. Bancos autônomos só se justificam quando a escala e a organização dos times exigem.
  > - **2PC (XA via JTA, ex.: Narayana ou Atomikos):** faz sentido com poucos participantes, todos controlados pela empresa e com suporte a XA, na mesma rede, com baixo volume e transações curtas. Ex.: um processo interno de back-office que grava em dois bancos.
  >
  > **Por que o 2PC é inviável aqui:**
  > - **Protocolo bloqueante:** na fase de *prepare*, cada participante mantém as linhas travadas até o coordenador decidir. A linha do SKU da TV (a linha "quente" da Aula 03) ficaria travada durante várias idas e voltas de rede entre três serviços, em vez de milissegundos. A 5.000 TPS, isso esgota os pools de conexão e gera fila.
  > - **Coordenador como ponto único de falha:** se ele cair entre o *prepare* e o *commit*, os participantes ficam "em dúvida", segurando locks até ele voltar.
  > - **Participantes fora do nosso controle:** o gateway Pix é uma API HTTP externa e não participa de XA. O Kafka também não. Não há como colocar a cobrança dentro de uma transação distribuída.
  > - **Disponibilidade multiplicada:** a transação só conclui se todos estiverem no ar. Três serviços com 99,9% cada resultam em ~99,7% para o checkout, e a latência passa a ser a do participante mais lento.
  > - **Acoplamento:** os serviços perdem autonomia de deploy, de escala e de escolha de banco.
  >
  > **Conclusão (CAP):** entre bancos autônomos ligados por rede, a escolha é disponibilidade com **consistência eventual**. Cada passo é uma transação ACID local, e a consistência global vem de uma **Saga** com ações compensatórias. A integridade financeira continua garantida porque toda falha tem um caminho de volta definido e automático.

### 1.2 O Modelo de Coordenação da Saga

- **A Escolha:** ( ) Coreografia (Orientada a Eventos)  (x) Orquestração (Process Manager / Máquina de Estados)
- **Justificativa e Mitigação das Desvantagens do seu Modelo:**
  - *Se você escolheu Coreografia:* Como seu design resolve a falta de visibilidade centralizada (saber em tempo real onde o pedido está) e previne dependências cíclicas à medida que novos serviços entrarem no fluxo de compra?
  - *Se você escolheu Orquestração:* Qual componente atua como Orquestrador, como ele se comunica com os participantes (síncrono ou assíncrono) e como você evita que ele se transforme em um "serviço Deus", tirando a inteligência e a autonomia dos outros microsserviços?
  > **Por que orquestração:** há dinheiro envolvido, então é preciso saber exatamente em que passo cada pedido está (suporte, auditoria, reconciliação). O fluxo tem ordem rígida e as compensações precisam rodar em ordem reversa. Em coreografia, essa lógica ficaria espalhada em listeners de vários serviços, e cada serviço novo (antifraude, frete, nota fiscal) aumentaria o risco de ciclos de eventos.
  >
  > **Quem é o orquestrador:** um componente `CheckoutSaga` dentro do serviço de Checkout, que é o dono do pedido. Ele é uma máquina de estados persistida no PostgreSQL do Checkout (tabela `saga_instance` com `order_id`, `estado`, `prazo`, `tentativas` e `version`, usando o lock otimista da Aula 03). Alternativa equivalente: **Temporal.io**, cuja classe `Saga` registra as compensações e as executa em ordem reversa. Ela reduz o código de infraestrutura, em troca de operar um cluster Temporal.
  >
  > **Comunicação: assíncrona, via Kafka, no modelo comando/resposta.**
  > - O orquestrador publica comandos (`ReservarEstoque`, `CobrarPagamento`, `ConfirmarReserva`, `EstornarPagamento`, `LiberarReserva`) nos tópicos de cada participante. Eles respondem no tópico `checkout.saga.replies` (`EstoqueReservado`, `EstoqueIndisponivel`, `PagamentoAprovado`, `PagamentoEstornado`...).
  > - A chave da mensagem é o `orderId`: todas as mensagens de um pedido caem na mesma partição, o que preserva a ordem.
  > - **Por que não síncrono (HTTP):** a 5.000 TPS, uma cadeia de chamadas segura threads e conexões enquanto espera, e a lentidão de um serviço se propaga para o checkout (o problema tratado na Aula 02). Além disso, o Pix já é assíncrono por natureza: a confirmação chega por webhook.
  > - O cliente recebe `202 Accepted` com o `orderId` e acompanha o status pelo modelo de leitura (seção 3.1).
  >
  > **Como evito o "serviço Deus":**
  > - O orquestrador decide só a **sequência** e **o que desfazer**, nunca a regra de negócio. O Estoque decide se há unidade; o Pagamento decide antifraude, gateway e prazos. O orquestrador só reage a "sucesso" ou "falha".
  > - Os comandos expressam intenção de negócio (`ReservarEstoque(orderId, sku, qtd)`), não operações de CRUD sobre os dados dos outros serviços.
  > - Cada participante é dono do seu banco e implementa a própria compensação (`LiberarReserva`, `EstornarPagamento`). O orquestrador nunca lê nem escreve no banco de outro serviço.
  > - Um orquestrador por processo de negócio (checkout, devolução, troca), e não um orquestrador central para tudo.
  > - Contratos de mensagem versionados (Avro ou JSON Schema no Schema Registry), para que cada participante evolua sem deploy conjunto.

---

## 2. Máquina de Estados, Compensação e Confiabilidade

### 2.1 Máquina de Estados e Fluxos Ponta a Ponta

- **Estados do Pedido:** Defina a máquina de estados explícita do ciclo de vida da transação no seu modelo (incluindo os estados iniciais, intermediários, de compensação e finais).
  > **Decisão de ordem dos passos: reservar o estoque antes de cobrar.** O pagamento é o passo mais difícil de desfazer (envolve dinheiro de terceiros), então ele vem depois do passo reversível e barato, a reserva com prazo da Aula 03. Isso elimina a maior parte do incidente, mas não todo: com Pix, o cliente pode pagar o QR Code depois que a reserva expirou e as unidades foram vendidas a outro cliente. Para esse caso existe a compensação.
  >
  > ```text
  > CRIADO → RESERVANDO_ESTOQUE → AGUARDANDO_PAGAMENTO → CONFIRMANDO_ESTOQUE → CONCLUIDO
  >               │                        │                       │
  >               ▼                        ▼                       ▼
  >           CANCELADO            LIBERANDO_ESTOQUE       ESTORNANDO_PAGAMENTO
  >                                        │                       │
  >                                        ▼                       ▼
  >                                    CANCELADO          CANCELADO_ESTORNADO
  > ```
  >
  > - **Inicial:** `CRIADO`: pedido gravado com os itens e o valor revalidados.
  > - **Intermediários:** `RESERVANDO_ESTOQUE` (comando enviado ao Estoque); `AGUARDANDO_PAGAMENTO` (unidade reservada por 10 min e QR Code Pix emitido); `CONFIRMANDO_ESTOQUE` (pagamento aprovado; a reserva vira baixa definitiva).
  > - **De compensação:** `LIBERANDO_ESTOQUE` (pagamento recusado ou QR Code expirado; devolve a unidade); `ESTORNANDO_PAGAMENTO` (dinheiro recebido, mas o estoque não foi confirmado; devolve o Pix).
  > - **Finais:** `CONCLUIDO` (sucesso); `CANCELADO` (nenhum dinheiro retido); `CANCELADO_ESTORNADO` (dinheiro devolvido ao cliente).
  > - **Implementação:** um `enum` com o mapa de transições permitidas. Toda mudança passa por `transicionar(evento)`, que rejeita transições inválidas. Um evento repetido para um estado já alcançado é ignorado (idempotência, como na Aula 03). Cada estado intermediário tem um **prazo**; um job busca sagas vencidas com um índice parcial `ON saga_instance (prazo) WHERE estado NOT IN ('CONCLUIDO', 'CANCELADO', 'CANCELADO_ESTORNADO')` e reenvia o comando ou inicia a compensação.
- **Detalhamento dos Fluxos:**
  - **Caminho Feliz:** Descreva o fluxo de ponta a ponta da transação concluída com sucesso entre Checkout, Pagamento e Estoque.
  - **Caminho de Falha & Compensação:** Descreva o passo a passo a partir da falha na reserva de estoque após o pagamento aprovado. Como a ação compensatória (estorno) é disparada e executada sem intervenção humana?
  > **Caminho feliz**
  > 1. **Checkout:** recebe `POST /checkout` com `Idempotency-Key` (Aula 02). Na mesma transação local, grava o pedido (`CRIADO`), a `saga_instance` e o comando `ReservarEstoque` na tabela outbox (seção 2.2). Responde `202 Accepted` com o `orderId`. Estado: `RESERVANDO_ESTOQUE`.
  > 2. **Estoque:** executa o `UPDATE` condicional da Aula 03 (`disponivel >= :qtd`), cria a reserva com prazo de 10 min e responde `EstoqueReservado`.
  > 3. **Checkout:** passa para `AGUARDANDO_PAGAMENTO` e envia `CobrarPagamento`.
  > 4. **Pagamento:** gera a cobrança Pix (com a própria chave de idempotência) e devolve o QR Code ao cliente. Quando o webhook do PSP confirma o pagamento, grava `PAGO` e responde `PagamentoAprovado`.
  > 5. **Checkout:** passa para `CONFIRMANDO_ESTOQUE` e envia `ConfirmarReserva`.
  > 6. **Estoque:** converte a reserva em baixa definitiva e responde `ReservaConfirmada`.
  > 7. **Checkout:** passa para `CONCLUIDO` e publica `PedidoConcluido`, consumido por logística, notificação e pelo modelo de leitura.
  >
  > **Caminho de falha: estoque rejeitado após o Pix aprovado**
  > 1. O cliente paga o Pix no minuto 12; a reserva expirou no minuto 10 e o job da Aula 03 devolveu a unidade, que já foi vendida.
  > 2. **Pagamento** grava `PAGO` e responde `PagamentoAprovado`. **Checkout** passa para `CONFIRMANDO_ESTOQUE` e envia `ConfirmarReserva`.
  > 3. **Estoque** encontra a reserva expirada, tenta reservar de novo com o mesmo `UPDATE` condicional e obtém 0 linhas afetadas. Responde `EstoqueIndisponivel`. É uma falha de negócio definitiva, então não há retry.
  > 4. **Checkout**, na mesma transação local: passa para `ESTORNANDO_PAGAMENTO` e grava na outbox o comando `EstornarPagamento(orderId, valor, idEstorno = "estorno-" + orderId)`.
  > 5. **Pagamento** registra o estorno (com `UNIQUE (order_id)`) e chama a devolução na API Pix do PSP (`PUT /pix/{e2eid}/devolucao/{idEstorno}`, em que o `id` da devolução já funciona como chave de idempotência). Ao receber a confirmação, responde `PagamentoEstornado`.
  > 6. **Checkout** passa para `CANCELADO_ESTORNADO` e publica `PedidoCancelado`. O cliente é notificado ("produto esgotado, valor devolvido") e "Meus Pedidos" é atualizado.
  >
  > **Sem intervenção humana:** cada passo é disparado por um evento (resposta do participante) ou por um timer (prazo vencido). O caminho de compensação está codificado na máquina de estados, e não depende de alguém perceber o problema.

### 2.2 Resiliência da Compensação e Consistência

- **E se a compensação falhar ou oscilar?** A chamada ou evento de estorno pode sofrer timeout ou instabilidade transitória. Como seu design garante que o cliente não fique no prejuízo se a tentativa de compensação falhar ou for reprocessada?
  > **Princípio:** a compensação pode falhar por motivo técnico, mas **nunca por regra de negócio**. O Pagamento sempre aceita um estorno de um pagamento que ele mesmo aprovou. Por isso ela pode ser repetida até concluir.
  >
  > - **Idempotência (reprocessar nunca estorna duas vezes):** o `idEstorno` é determinístico (`estorno-{orderId}`). No Pagamento, a tabela `estornos` tem `UNIQUE (order_id)`; um comando repetido devolve o resultado já existente. No PSP, o mesmo `id` de devolução não gera uma segunda devolução.
  > - **Timeout ambíguo (como na pergunta 1 da Aula 02):** não assumo sucesso nem falha. Antes de repetir, o Pagamento consulta o status da devolução pelo `idEstorno` (`GET /pix/{e2eid}/devolucao/{idEstorno}`).
  > - **Retries sem travar a partição:** `@RetryableTopic` do Spring Kafka com backoff exponencial (ex.: 1 min, 5 min, 30 min), em tópicos de retry separados. Assim, um estorno com problema não bloqueia os outros pedidos da mesma partição. A chamada ao PSP tem TimeLimiter e Circuit Breaker (Resilience4j, Aula 02).
  > - **Estado durável:** o orquestrador grava `ESTORNANDO_PAGAMENTO` antes de enviar o comando. Se o serviço cair, o job de prazos encontra a saga vencida e reenvia o comando, o que é seguro por causa da idempotência.
  > - **DLT não é cemitério:** esgotados os retries, a mensagem vai para a Dead Letter Topic, e um consumidor a reprocessa automaticamente em intervalos maiores (ex.: a cada hora).
  > - **Reconciliação:** um job diário compara o relatório de liquidação do PSP com os pedidos. Qualquer Pix recebido sem pedido `CONCLUIDO` e sem estorno concluído dispara `EstornarPagamento`. É a rede de segurança caso uma mensagem se perca por um bug.
  > - **Observabilidade e último recurso:** métricas de sagas em `ESTORNANDO_PAGAMENTO` há mais de 15 min e do tamanho da DLT, com alerta. Se após 24 h o estorno ainda não concluiu (ex.: PSP fora do ar), o plantão é acionado com o histórico completo da saga. O cliente é informado de que o estorno está em processamento, com prazo. O estado nunca se perde.
- **Atomicidade entre Estado e Mensageria (Dual Write):** Se a sua arquitetura envolve publicação de eventos ou mensagens entre passos, como você garante que a persistência no banco local e o despacho da mensagem ocorram sem risco de perda silenciosa se a rede ou o broker falharem entre as duas operações?
  > **Transactional Outbox.** O serviço nunca publica direto no Kafka dentro da regra de negócio.
  >
  > - **Mesma transação local:** a mudança de estado e o evento são gravados juntos no PostgreSQL. Ou os dois são gravados, ou nenhum.
  >   ```sql
  >   BEGIN;
  >   UPDATE saga_instance SET estado = 'ESTORNANDO_PAGAMENTO', version = version + 1
  >    WHERE order_id = :id AND version = :v;
  >   INSERT INTO outbox (id, aggregate_id, type, payload)
  >   VALUES (:uuidv7, :id, 'EstornarPagamento', :json);
  >   COMMIT;
  >   ```
  > - **Publicação (relay):** o **Debezium** lê o WAL do PostgreSQL (CDC) e publica a linha da outbox no Kafka com o *Outbox Event Router*. O código do serviço nem conhece o Kafka. Alternativa sem Debezium: um publicador agendado que lê a outbox com `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 500`, publica e marca como enviado (ou o *Event Publication Registry* do Spring Modulith).
  > - **Se o broker cair:** os eventos ficam na outbox e são publicados quando ele voltar. Atraso, sim; perda, não.
  > - **Entrega "pelo menos uma vez" gera duplicatas:** cada consumidor é idempotente com uma tabela **inbox** (`processed_messages`, `message_id` como chave primária), inserida na mesma transação do efeito de negócio. Uma mensagem repetida viola a chave e é ignorada. A máquina de estados é uma segunda barreira, porque rejeita transições já feitas.
  > - **Ordem:** chave da mensagem = `orderId` (mesma partição por pedido).
  > - **Manutenção:** a outbox é particionada por dia e as partições antigas são removidas, para não crescer sem limite.
  > - **O que evito:** `@Transactional` com `kafkaTemplate.send()` no mesmo método (continua sendo dual write: o commit pode falhar depois do envio) e `@TransactionalEventListener(AFTER_COMMIT)` (se o processo cair logo após o commit, o evento se perde). Transações do Kafka cobrem só o Kafka, não o PostgreSQL.

---

## 3. Desacoplamento com CQRS e Estratégia de Testes

### 3.1 Consultas em Alta Vazão com CQRS

- **O Problema de Leitura:** Durante o pico de vendas, milhões de usuários atualizam a página de "Meus Pedidos" a cada poucos segundos.
- **Desenho da Solução:** Como você separa a responsabilidade de escrita (Command) da consulta de status (Query)? Como o modelo de leitura é alimentado e como sua arquitetura lida com a consistência eventual (o delay entre o pagamento e a visualização do status)?
  > **Lado de escrita (Command):** `POST /checkout` e os eventos da saga gravam no PostgreSQL do Checkout, com modelo normalizado e otimizado para transações. Nenhuma consulta de "Meus Pedidos" chega a esse banco, então o pico de leituras não disputa conexões com o checkout (o problema do HikariCP da Aula 03).
  >
  > **Lado de leitura (Query):** um serviço separado, `pedidos-query`, com uma visão desnormalizada `meus_pedidos` em banco próprio: uma linha por pedido, já com status, itens, valor e linha do tempo, com chave `(customer_id, created_at DESC)`. A consulta vira uma leitura simples por cliente, paginada, sem `JOIN`. Os status de pedidos em andamento ficam também em cache no Redis.
  >
  > **Como o modelo de leitura é alimentado:**
  > - Um projetor consome do Kafka os mesmos eventos que saem da outbox (`PedidoCriado`, `PagamentoAprovado`, `PedidoConcluido`, `PedidoCancelado`...) e atualiza a visão.
  > - **Idempotente e à prova de desordem:** cada evento carrega a `version` do pedido, e a atualização só aplica eventos mais novos:
  >   ```sql
  >   UPDATE meus_pedidos SET status = :status, version = :v
  >    WHERE order_id = :id AND version < :v;
  >   ```
  > - **Reconstruível:** se a visão corromper ou mudar de formato, ela é recriada reprocessando os eventos (tópico com retenção longa ou compactado).
  >
  > **Como lido com a consistência eventual:**
  > - **Status honesto:** a visão mostra estados de transição ("Aguardando pagamento", "Pagamento em processamento", "Estorno em andamento") em vez de esconder o atraso.
  > - **Ler a própria escrita:** a resposta `202` do checkout já traz o pedido com o status inicial, e o front-end o exibe na hora, sem esperar a projeção.
  > - **Menos polling:** mudanças de status são enviadas por SSE (Server-Sent Events) para quem está com a página aberta. Quem faz polling usa `ETag`/`If-None-Match`, e a maioria das requisições recebe `304 Not Modified`, sem consultar o banco.
  > - **Atraso monitorado:** métrica de *consumer lag* e de `agora - evento.created_at`, com meta (ex.: p99 < 2 s) e alerta. Se o lado de leitura cair, o checkout continua funcionando.
  > - **Limite claro:** o modelo de leitura serve só para exibição. Decisões sobre preço, estoque ou dinheiro são sempre tomadas no lado de escrita (mesma regra da Aula 03).

### 3.2 Estratégia de Testes da Transação Distribuída

- **Validação de Resiliência:** Como comprovar em ambiente de testes e pipeline que a Saga funciona de verdade?
- **Plano de Testes:** Descreva como você estruturaria o teste automatizado para simular a rejeição na reserva de estoque após a cobrança e validar que a compensação automática transiciona o pedido para cancelado com sucesso.
  > **1. Testes unitários da máquina de estados (a cada commit)**
  > - `@ParameterizedTest` (JUnit 5) cobrindo cada par (estado, evento): transições válidas, transições inválidas rejeitadas e eventos repetidos ignorados. Ex.: `CONFIRMANDO_ESTOQUE` + `EstoqueIndisponivel` → `ESTORNANDO_PAGAMENTO`. São testes rápidos, sem Spring.
  >
  > **2. Teste de integração da Saga (no pipeline de cada PR)**
  > - `@SpringBootTest` com **Testcontainers** (PostgreSQL e Kafka, via `@ServiceConnection`) e **WireMock** no lugar do PSP Pix. Debezium pode ser substituído pelo publicador agendado da outbox no teste.
  > - Um `Clock` injetável permite avançar o tempo e expirar a reserva sem esperar 10 minutos de verdade.
  > - **Roteiro do cenário do incidente:**
  >   1. *Arrange:* SKU com 1 unidade; criar o pedido e esperar `AGUARDANDO_PAGAMENTO`.
  >   2. Avançar o `Clock` além do prazo da reserva, rodar o job de expiração e vender a unidade para outro pedido.
  >   3. *Act:* simular o webhook de Pix aprovado.
  >   4. *Assert* (com Awaitility, pois o fluxo é assíncrono): pedido em `CANCELADO_ESTORNADO`; exatamente **uma** chamada de devolução ao PSP; estoque nunca negativo; eventos da outbox na ordem esperada; "Meus Pedidos" mostrando o cancelamento.
  >   ```java
  >   @Test
  >   void estornaPixQuandoEstoqueFalhaAposPagamento() {
  >       var orderId = checkout.criarPedido("TV-55", 1);
  >       await().until(() -> saga.estado(orderId) == AGUARDANDO_PAGAMENTO);
  >
  >       clock.avancar(Duration.ofMinutes(11));
  >       estoque.expirarReservas();
  >       checkout.criarPedido("TV-55", 1);          // outro cliente leva a unidade
  >
  >       pixWebhook.confirmar(orderId);
  >
  >       await().atMost(Duration.ofSeconds(15)).untilAsserted(() ->
  >           assertThat(saga.estado(orderId)).isEqualTo(CANCELADO_ESTORNADO));
  >       wireMock.verify(1, putRequestedFor(urlMatching("/pix/.*/devolucao/estorno-" + orderId)));
  >       assertThat(estoque.disponivel("TV-55")).isZero();
  >   }
  >   ```
  >
  > **3. Variações de resiliência (mesma base de teste)**
  > - **PSP instável:** WireMock responde `503` duas vezes e depois `200` (*scenarios*). Esperado: estado final correto e uma única devolução efetiva.
  > - **Timeout ambíguo:** resposta com atraso maior que o TimeLimiter. Esperado: o Pagamento consulta o status da devolução antes de repetir.
  > - **Mensagem duplicada:** publicar `EstoqueIndisponivel` duas vezes. Esperado: um único estorno (inbox + máquina de estados).
  > - **Queda do serviço no meio da saga:** parar o contexto Spring logo após gravar `ESTORNANDO_PAGAMENTO` e subir de novo. Esperado: o job de prazos retoma e conclui.
  > - **Broker fora do ar:** pausar o container do Kafka. Esperado: eventos acumulados na outbox e publicados ao despausar, sem perda.
  >
  > **4. Testes de contrato:** Spring Cloud Contract ou Pact para as mensagens entre o orquestrador e os participantes, evitando que uma mudança de schema quebre a saga em produção.
  >
  > **5. Staging (execução noturna):** teste de carga (Gatling ou k6) próximo de 5.000 TPS, com injeção de falhas (Toxiproxy para latência e quedas). Validação por invariantes: total de Pix recebidos = pedidos concluídos + estornos concluídos; nenhuma saga parada além do prazo; estoque nunca negativo.
