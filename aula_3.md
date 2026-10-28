# Template de System Design: Aula 03 - Banco de Dados em Larga Escala

> **Contexto do Cenário: Checkout na Black Friday**
>
> Sua equipe está preparando a persistência da API de Checkout para o pico de tráfego da Black Friday:
>
> 1. **Armazenamento:** Transações financeiras de pedidos exigem garantias ACID rigorosas e consistência imediata, enquanto dados de catálogo e sessões de usuários demandam alta vazão de leitura e tolerância a partição.
> 2. **Concorrência:** A atualização de status do pedido tem baixa disputa por registro (apenas o comprador interage), mas a **reserva de estoque** sofre altíssima contenção com centenas de clientes disputando as últimas unidades do mesmo SKU promocional.
> 3. **Incidente em Produção:** Em testes de carga, o tempo de resposta p99 subiu para 4.8s e o pool HikariCP saturou devido a consultas lentas de pedidos (`WHERE customer_id = ?` e `WHERE order_status = 'processing'`).
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Adicione este bloco ao repositório contínuo do seu projeto.

---

## 1. Decisão de Armazenamento e Chave Primária (PK)

### 1.1 Motor de Banco de Dados e Paradigma de Persistência

- **A Escolha para o Checkout:** ( ) Relacional Puro (PostgreSQL) (x) Persistência Poliglota / Híbrida
- **Justificativa Arquitetural:** Avalie o Teorema CAP e o modelo de consistência (ACID vs BASE). Por que a sua escolha atende aos requisitos do Checkout sem comprometer a integridade financeira?
  > **Dados financeiros ficam no PostgreSQL; o resto vai para o banco mais adequado a cada uso.**
  >
  > - **Teorema CAP:** quando há uma falha de rede entre servidores (partição), o sistema precisa escolher entre consistência e disponibilidade. Para pedido, pagamento e baixa de estoque, a escolha é **consistência (CP)**: é melhor recusar um pedido por alguns segundos do que vender um item sem estoque ou cobrar duas vezes.
  > - **ACID no núcleo:** pedido, itens e reserva de estoque são gravados **na mesma transação** no PostgreSQL (um único servidor primário de escrita). Ou tudo é gravado, ou nada é. O pagamento é registrado depois, em outra transação curta, porque depende da resposta do parceiro externo e não pode segurar locks enquanto espera (seção 2.2). Restrições do banco (`UNIQUE` na `idempotency_key`, `CHECK (disponivel >= 0)`, chaves estrangeiras) são a última barreira contra dados inválidos.
  > - **BASE na periferia:** catálogo e sessões aceitam **consistência eventual** (o dado pode ficar alguns segundos desatualizado) em troca de alta vazão de leitura e disponibilidade (AP). Ficam em cache (Redis) e em réplicas de leitura.
  > - **Por que não compromete a integridade:** nenhum dado eventual decide dinheiro. O preço e o estoque exibidos vêm do cache, mas no momento do checkout são **revalidados no PostgreSQL** dentro da transação.
  > - **Por que não Relacional Puro:** colocar sessões, carrinhos e consultas de catálogo no mesmo PostgreSQL consumiria conexões e I/O justamente do fluxo financeiro, no pico da Black Friday.
- **Além do Modelo Documental (Ecossistema NoSQL):** A persistência poliglota vai muito além da dicotomia "Relacional vs MongoDB". Considerando as necessidades complementares de uma plataforma de e-commerce (como carrinhos efêmeros, busca textual de produtos, telemetria/logs, auditoria temporal ou análise de conexões/fraude), cite e comente pelo menos outros dois tipos/famílias de bancos NoSQL que poderiam coexistir na arquitetura, indicando para quais casos de uso específicos cada um seria a ferramenta ideal e por quê.
  > - **Chave-valor em memória (Redis):** carrinhos efêmeros, sessões, cache de catálogo e chaves de idempotência. Latência abaixo de 1 ms, expiração automática por TTL (o carrinho abandonado some sozinho) e operações atômicas como `INCR`/`DECRBY`, úteis como "porteiro" do estoque (seção 2.2).
  > - **Motor de busca / índice invertido (Elasticsearch ou OpenSearch):** busca textual de produtos, filtros por categoria e preço (facetas), tolerância a erros de digitação e ordenação por relevância. Um `LIKE '%termo%'` no PostgreSQL não escala para isso. O índice é alimentado por eventos ou CDC (captura de mudanças, ex.: Debezium), e alguns segundos de atraso são aceitáveis.
  > - **Colunar / séries temporais (Cassandra, ClickHouse ou InfluxDB):** telemetria, logs, cliques e trilha de auditoria. São dados que só recebem inserções (append-only), em volume muito alto, consultados por janela de tempo ("eventos do SKU X entre 0h e 1h"). Esses bancos escrevem em sequência e comprimem bem, sem disputar recursos com o banco transacional.
  > - **Grafo (Neo4j):** análise de fraude. Encontrar contas ligadas pelo mesmo cartão, dispositivo ou endereço exige percorrer relações de vários níveis, o que em SQL viraria muitos `JOIN`s recursivos e lentos.

### 1.2 Estratégia da Chave Primária

- **A Escolha:** ( ) Sequence / Identity ( ) UUIDv4 (x) UUIDv7 / TSID
- **Impacto no Armazenamento:** Por que essa estratégia foi escolhida considerando inserções concorrentes e o impacto de fragmentação na árvore B-Tree (page splits)?
  > **UUIDv7 (tipo nativo `uuid` do PostgreSQL), gerado na aplicação.**
  >
  > - **UUIDv4 é descartado:** é totalmente aleatório, então cada inserção cai numa página aleatória do índice B-Tree. Com páginas cheias, o banco precisa dividi-las (**page splits**), o índice fica fragmentado, mais páginas precisam ficar em memória e o volume de WAL (log de escrita) aumenta. Sob milhares de inserções concorrentes na Black Friday, isso degrada as escritas.
  > - **Sequence / Identity tem limitações:** é ordenada e compacta (8 bytes), mas o ID só existe depois do `INSERT` (com `IDENTITY`, o Hibernate também desativa inserções em lote). Expõe o volume de vendas e permite enumerar pedidos (`/orders/1001`, `/orders/1002`), e dificulta sharding ou várias regiões no futuro.
  > - **Por que UUIDv7:** os primeiros 48 bits são um timestamp em milissegundos, então os IDs são **crescentes no tempo**. As inserções caem sempre no final da B-Tree, como numa sequence, com poucos page splits e índice compacto. Ao mesmo tempo, a aplicação gera o ID sem consultar o banco (útil para usar o ID em eventos e na idempotência antes do `INSERT`), e a parte aleatória impede enumerar pedidos. O PostgreSQL 18 já gera UUIDv7 nativamente com `uuidv7()`.
  > - **Trade-off aceito:** ocupa 16 bytes em vez de 8 e revela o horário de criação do pedido, o que não é sensível neste caso. Se o espaço for crítico, o **TSID** (8 bytes, `bigint`) traz o mesmo benefício de ordenação.

---

## 2. A Dualidade da Concorrência: Locks e Mitigações

Analise e defina a abordagem de concorrência para os dois fluxos críticos do negócio:

### 2.1 Mudança de Status do Pagamento (Baixa contenção por registro)

- **Abordagem de Lock:** (x) Lock Otimista (`@Version`) ou ( ) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Por que essa abordagem é a mais adequada para este cenário específico?
  > - **Conflito é raro:** só o comprador interage com o próprio pedido. Uma colisão real só acontece em casos pontuais, como o webhook do gateway de pagamento e o job de reconciliação atualizando o mesmo pedido ao mesmo tempo.
  > - **Como funciona:** a entidade tem uma coluna `@Version`. O Hibernate executa `UPDATE ... SET status = ?, version = version + 1 WHERE id = ? AND version = ?`. Se nenhuma linha for afetada, outra transação alterou o pedido antes, e é lançada uma `OptimisticLockException`.
  > - **Protege o pool de conexões:** nenhuma linha fica travada enquanto a transação roda, então nenhuma conexão do HikariCP fica parada esperando lock. Um lock pessimista aqui pagaria esse custo em 100% das requisições para evitar um conflito que quase nunca ocorre.
  > - **Tratamento do conflito:** a aplicação relê o pedido e reaplica a mudança só se a transição ainda for válida na máquina de estados (ex.: `PENDENTE → PAGO`). Se o pedido já estiver `PAGO`, a operação é tratada como já concluída (idempotente).

### 2.2 Controle de Estoque Concorrente (Alta contenção no mesmo registro)

- **Abordagem de Lock:** ( ) Lock Otimista (`@Version`) ou (x) Lock Pessimista (`SELECT FOR UPDATE`)?
- **Justificativa:** Centenas de usuários disputando os últimos itens na Black Friday. Como você garante que o saldo nunca fique negativo sem derrubar o banco?
  > - **Por que não otimista:** com centenas de clientes lendo a mesma versão do registro, só um vence e os outros recebem `OptimisticLockException`. Os retries geram uma tempestade de novas tentativas que aumenta a carga justamente no pior momento, e a maioria dos clientes vê erro mesmo havendo estoque.
  > - **Pessimista na forma mais curta possível:** o lock fica a cargo do próprio banco, num `UPDATE` condicional e atômico. O PostgreSQL trava a linha só durante o comando (milissegundos) e enfileira os concorrentes:
  >   ```sql
  >   UPDATE estoque
  >      SET disponivel = disponivel - :qtd
  >    WHERE sku = :sku AND disponivel >= :qtd;
  >   ```
  >   Se 1 linha for afetada, a unidade está reservada; se 0, o produto esgotou e o cliente recebe a resposta na hora. Quando a regra exige ler antes de decidir, uso `SELECT ... FOR UPDATE` (`@Lock(LockModeType.PESSIMISTIC_WRITE)` no JPA) em uma transação curta.
  > - **Saldo nunca negativo:** garantido pela condição `disponivel >= :qtd` e, como última barreira, por `CHECK (disponivel >= 0)` na tabela.
  > - **Sem derrubar o banco:**
  >   - **Nada de chamada externa dentro do lock:** o estoque é **reservado** com prazo (ex.: 10 min). O pagamento acontece depois, fora da transação; se falhar ou expirar, um job devolve a unidade.
  >   - **Porteiro no Redis:** um contador atômico (`DECRBY`) por SKU promocional filtra a demanda. Quando ele chega a zero, a API responde "esgotado" sem tocar no PostgreSQL. Só quem tem chance real de comprar chega ao banco, que continua sendo a fonte da verdade.
  >   - **Concorrência limitada:** um Bulkhead (semáforo) no fluxo de estoque impede que ele ocupe todas as conexões do pool.

### 2.3 Riscos e Mitigações

- **Análise Crítica:** Quais os riscos operacionais das abordagens de lock que você escolheu (ex: excesso de `OptimisticLockException` frustrando o cliente vs `ConnectionTimeoutException` e *deadlocks* enfileirando threads no Lock Pessimista)? Como sua arquitetura mitiga esses riscos?
  > **Lock otimista (status do pagamento)**
  > - **Risco:** se a disputa for maior que o previsto (ex.: webhooks duplicados do gateway), as `OptimisticLockException` se multiplicam e o cliente vê erro.
  > - **Mitigação:** retry limitado (2 a 3 tentativas) com backoff exponencial e jitter (Resilience4j), aplicado só a transições idempotentes; métrica da taxa de conflitos com alerta se passar de ~1%, sinal de que a premissa de baixa disputa deixou de valer.
  >
  > **Lock pessimista (estoque)**
  > - **Risco 1: fila de espera esgotando o pool.** Transações esperando o lock seguram conexões; o HikariCP satura e **todo** o checkout passa a falhar com `ConnectionTimeoutException`, inclusive fluxos que nem tocam o estoque.
  >   - Transações curtas, sem I/O externo dentro.
  >   - `SET LOCAL lock_timeout = '2s'` e `statement_timeout` no PostgreSQL, e `connectionTimeout` baixo no HikariCP: é melhor falhar rápido do que enfileirar.
  >   - Bulkhead dedicado ao estoque com limite menor que o tamanho do pool, para sobrar conexões para o resto do sistema.
  > - **Risco 2: deadlocks.** Um pedido com vários SKUs pode travar as linhas em ordem diferente de outro pedido, e um espera pelo outro para sempre.
  >   - Travar sempre na mesma ordem (por `sku` crescente, ex.: `SELECT ... ORDER BY sku FOR UPDATE`).
  >   - O PostgreSQL detecta o deadlock e aborta uma das transações; a aplicação repete essa transação uma vez.
  > - **Risco 3: linha "quente".** Um único SKU concentra toda a disputa.
  >   - O porteiro no Redis reduz quantas requisições chegam ao banco. Se ainda não bastar, o estoque do SKU pode ser dividido em N linhas (ex.: 10 lotes), espalhando os locks.
  > - **Risco 4: reservas órfãs.** Unidades presas por clientes que abandonaram o pagamento.
  >   - Reserva com prazo de expiração e job que devolve as unidades.
  >
  > **Experiência do cliente e observabilidade:** respostas claras ("esgotado", ou `429`/`503` com `Retry-After`) em vez de timeouts longos, fila virtual se o pico exigir, e monitoramento de `pg_locks`, eventos de espera e conexões pendentes no HikariCP.

---

## 3. Troubleshooting do Incidente e Gestão de Índices

### 3.1 Investigação de Incidente em Produção

- **Cenário:** Em testes de carga para a Black Friday, o tempo de resposta p99 do Checkout subiu repentinamente para 4.8s e o pool de conexões (HikariCP) saturou. Os logs apontam que a consulta de pedidos mais frequente (`WHERE customer_id = ? AND order_status = 'processing'`) está demorando segundos para responder.
- **Plano de Diagnóstico:** Qual é o seu processo técnico passo a passo para investigar a causa raiz dessa lentidão diretamente no banco de dados? Quais comandos, métricas ou planos você analisaria para comprovar a hipótese antes de propor qualquer mudança?
  > **Hipótese inicial:** a consulta não tem índice adequado e faz leitura completa da tabela (Seq Scan). Cada execução segura uma conexão por segundos, o pool satura e o p99 sobe. Aumentar o pool só esconderia o problema e colocaria mais carga no banco.
  >
  > 1. **Confirmar o sintoma no HikariCP:** comparar `hikaricp_connections_active`, `hikaricp_connections_pending` e `hikaricp_connections_usage` (tempo que cada conexão fica emprestada). Tempo de uso alto indica consulta lenta, não pool pequeno.
  > 2. **Achar as consultas mais caras com `pg_stat_statements`:** ordenar por `total_exec_time` e `mean_exec_time`, olhando `calls`, `rows` e `shared_blks_read` (blocos lidos do disco), para confirmar que é essa consulta que domina o tempo do banco.
  > 3. **Descartar lock como causa com `pg_stat_activity`:** ver o que está rodando agora, há quanto tempo e o `wait_event_type`. Se for `Lock`, o problema é disputa (por exemplo, com o estoque), não índice; `pg_blocking_pids()` e `pg_locks` mostram quem bloqueia quem. Se for `IO` ou CPU, reforça a hipótese do Seq Scan.
  > 4. **Ler o plano de execução real:**
  >    ```sql
  >    EXPLAIN (ANALYZE, BUFFERS)
  >    SELECT * FROM orders WHERE customer_id = :id AND order_status = 'processing';
  >    ```
  >    Procurar `Seq Scan on orders`, um `Rows Removed by Filter` muito alto, grande diferença entre linhas estimadas e reais, e muitos buffers lidos do disco. Testar com um cliente com muitos pedidos e com um cliente comum.
  > 5. **Checar índices e estatísticas da tabela:** `\d orders` para os índices existentes; `pg_stat_user_tables` para `seq_scan` vs `idx_scan`, `n_dead_tup` (linhas mortas, sinal de bloat) e a data do último `autovacuum`/`analyze`. Estatísticas desatualizadas podem levar o planejador a escolher um plano ruim.
  > 6. **Medir a seletividade:** `SELECT order_status, count(*) FROM orders GROUP BY 1;` mostra que `processing` é uma fração pequena da tabela, o que orienta o tipo de índice (seção 3.2).
  > 7. **Verificar a aplicação:** logs SQL e estatísticas do Hibernate para descartar N+1 (a mesma consulta disparada em loop) ou consultas sem paginação.
  > 8. **Comprovar antes de mudar:** em staging com volume de dados parecido com o de produção, criar o índice proposto, repetir o `EXPLAIN` (esperado: `Index Scan` e tempo em milissegundos) e rodar o teste de carga de novo, comparando p99 e conexões pendentes no HikariCP.

### 3.2 Decisão de Engenharia: A Abordagem de Índices

- **Dilema no Pull Request:** Para solucionar a lentidão, um desenvolvedor da equipe abriu um PR criando índices individuais para todas as colunas da tabela de pedidos que aparecem em filtros de consulta.
- **Seu Posicionamento:** Como responsável técnico, você aprovaria esse PR? Qual é a sua avaliação crítica sobre essa estratégia e quais impactos colaterais sistêmicos devem ser ponderados antes de aprovar novos índices?
  > **Não aprovaria o PR como está.** Pediria mudanças.
  >
  > **Problemas da estratégia "um índice por coluna":**
  > - **Não resolve bem a consulta:** para `customer_id = ? AND order_status = ?`, o PostgreSQL usaria um índice e filtraria o resto, ou combinaria dois índices (`BitmapAnd`), o que é mais lento que um índice composto. Um índice só em `order_status` tem poucos valores distintos e muitas vezes é ignorado pelo planejador.
  > - **Deixa as escritas mais lentas:** todo `INSERT` e `UPDATE` precisa atualizar todos os índices. Na Black Friday o checkout é dominado por escritas. Como `order_status` muda várias vezes por pedido, indexar essa coluna também impede as atualizações HOT do PostgreSQL (que evitam reescrever índices), gerando mais WAL e mais atraso nas réplicas.
  > - **Consome memória e disco:** os índices disputam o `shared_buffers` com os dados mais acessados, aumentam o bloat e deixam o `VACUUM` e os backups mais demorados.
  > - **Risco na criação:** `CREATE INDEX` sem `CONCURRENTLY` bloqueia as escritas na tabela de pedidos enquanto o índice é construído.
  >
  > **Proposta:** índices guiados pelas consultas reais (`pg_stat_statements` + `EXPLAIN`), não pelas colunas.
  > ```sql
  > -- Atende customer_id = ? AND order_status = ? e também customer_id = ? sozinho (prefixo à esquerda)
  > CREATE INDEX CONCURRENTLY idx_orders_customer_status ON orders (customer_id, order_status);
  >
  > -- Índice parcial: só os pedidos em processamento, pequeno e barato de manter
  > CREATE INDEX CONCURRENTLY idx_orders_processing ON orders (created_at) WHERE order_status = 'processing';
  > ```
  > - `customer_id` vem primeiro porque é muito seletivo e porque o índice também atende a consulta só por cliente.
  > - O índice parcial cobre a fila de `processing`, que é uma pequena fração da tabela.
  > - **Custo assumido:** os dois índices também envolvem `order_status` (o composto como coluna, o parcial no `WHERE`), então as mudanças de status também deixam de ser atualizações HOT. Aceito esse custo porque são só 2 índices escolhidos pelas consultas reais, e não um por coluna, e porque eles transformam uma leitura de segundos em milissegundos. Para limitar o impacto, uso `fillfactor = 90` na tabela, para que as atualizações que não mexem em colunas indexadas continuem HOT, e acompanho `n_tup_hot_upd` em `pg_stat_user_tables` depois do deploy.
  >
  > **Condições para aprovar:**
  > - `EXPLAIN (ANALYZE, BUFFERS)` antes e depois anexado ao PR, mais o resultado do teste de carga.
  > - Migration com `CREATE INDEX CONCURRENTLY`, executada fora de transação e fora do horário de pico.
  > - Depois do deploy, acompanhar `pg_stat_user_indexes` e remover índices com `idx_scan = 0`, que só têm custo.
