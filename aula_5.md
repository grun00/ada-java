# Template de System Design: Aula 05 - Containers, CI/CD & Deploy sem Downtime

> **Contexto do Cenário: O Deploy que Quebrou o Checkout e o Rollback Impossível**
>
> Sua equipe está reestruturando a engenharia de release do serviço de **Checkout** após um incidente severo na Black Friday:
>
> 1. **O Incidente:** Para subir uma otimização urgente na Saga, o pipeline de produção recompilou a branch `main`, sobrescreveu a tag `checkout:latest` e iniciou a atualização. No boot da primeira instância nova, o **Flyway** renomeou a coluna `status_pagamento` para `estado_saga`.
> 2. **O Efeito em Cadeia:** Os 80% de servidores que ainda rodavam a versão anterior passaram a falhar imediatamente (`column "status_pagamento" does not exist`). Ao tentar acionar o rollback puxando `:latest`, a equipe descobriu que a imagem anterior havia sido sobrescrita e, mesmo restaurando um backup do binário antigo, ele continuou falhando porque o banco já estava alterado.
> 3. **A Carga:** 5.000 TPS ativos com múltiplas réplicas de aplicação conectadas ao mesmo banco PostgreSQL.
>
> **Instruções:** Responda às decisões arquiteturais abaixo fundamentando suas escolhas estritamente com base nos requisitos e restrições deste cenário. Documente suas decisões no repositório contínuo do seu projeto através de commits atômicos e bem descritivos.

---

## 1. Empacotamento, Superfície de Ataque e Promoção Imutável

### 1.1 Anatomia da Imagem e Redução de Superfície de Ataque

- **Construção vs Execução:** Como você estruturará o empacotamento do serviço de Checkout para produção? Especifique o que deve ser usado na etapa de compilação, o que deve (e o que não deve) entrar na imagem final de execução, com qual permissão de usuário o processo rodará e qual o trade-off da sua escolha de imagem base para a segurança e para o diagnóstico em produção.
  > **Resumo:** build multi-stage. A JDK e o Maven ficam só no estágio de compilação; a imagem final leva apenas a JRE e a aplicação, roda como usuário não-root e usa base Distroless.
  >
  > **Estágio 1: compilação (descartado):**
  > - Base `eclipse-temurin:25-jdk` (mesma versão LTS do projeto), fixada por digest (`@sha256:...`) e atualizada automaticamente pelo Renovate/Dependabot via PR.
  > - Copio primeiro `pom.xml`, `mvnw` e `.mvn/`, rodo `dependency:go-offline` e só depois copio `src/`. Assim, a camada de dependências fica em cache e só é refeita quando o `pom.xml` muda (`RUN --mount=type=cache,target=/root/.m2` acelera ainda mais).
  > - Gero o jar e extraio as camadas do Spring Boot (`java -Djarmode=tools -jar app.jar extract --layers --launcher`): `dependencies`, `spring-boot-loader`, `snapshot-dependencies` e `application`. Uma correção de código altera só a última camada, de poucos KB, e o push/pull no deploy fica rápido.
  > - Os testes rodam em etapa própria da esteira (seção 3.1), antes do `docker build`. Por isso o `-DskipTests` no Dockerfile é aceitável: a imagem só é construída depois que os testes passaram.
  >
  > **Estágio 2: execução (imagem final):**
  > - Base `gcr.io/distroless/java25-debian13:nonroot` (ou a variante `java21-debian12` do guia, se o projeto ainda estiver no Java 21), também fixada por digest.
  > - **Entra:** apenas a JRE, as camadas extraídas da aplicação e os certificados CA. Labels OCI (`org.opencontainers.image.revision`, `.source`, `.version`) apontam para o commit de origem.
  > - **Não entra:** JDK, Maven, `~/.m2`, código-fonte, `.git`, testes, `shell`, `apt`, `curl`, ferramentas de rede e, principalmente, **nenhum segredo** nem `application-prod.yml` com credenciais. O `.dockerignore` reforça isso.
  > - **Usuário:** `nonroot` (UID 65532). No Kubernetes, completo com `runAsNonRoot: true`, `readOnlyRootFilesystem: true` (com um `emptyDir` em `/tmp`), `allowPrivilegeEscalation: false` e `capabilities.drop: [ALL]`. Se alguém explorar uma falha na aplicação, não consegue instalar ferramentas nem escrever no sistema de arquivos.
  > - **JVM no container:** `ENTRYPOINT` em formato exec (`["java", ...]`), para o processo Java ser o PID 1 e receber o `SIGTERM`; `server.shutdown=graceful` para terminar as requisições em andamento; `-XX:MaxRAMPercentage=75` para respeitar o limite de memória do pod.
  >
  > **Trade-off da Distroless:**
  > - **Segurança:** muito menos pacotes de sistema operacional significa menos CVEs para corrigir e nenhum shell para um invasor usar. O guia mostra isso comparando o Trivy da `eclipse-temurin:jdk` com a Distroless.
  > - **Diagnóstico:** não dá para fazer `kubectl exec ... sh`. Compenso com observabilidade de fora para dentro (logs estruturados, métricas Micrometer/Prometheus, tracing OpenTelemetry, Actuator) e, quando for inevitável, com `kubectl debug` usando um **container efêmero** (ex.: imagem `:debug` ou uma imagem com JDK para `jcmd`/JFR) que compartilha o namespace de processos do pod, sem alterar a imagem de produção. Esse acesso é registrado (seção 3.1).
  > - **Alternativa:** `./mvnw spring-boot:build-image` (Paketo) gera imagem não-root, em camadas, reprodutível e com SBOM, sem Dockerfile. É uma boa opção se o time preferir não manter o Dockerfile; a base dele (Ubuntu Jammy "tiny") é um pouco maior que a Distroless.

### 1.2 Rastreabilidade, Identidade do Artefato e Parametrização

- **Da Homologação à Produção:** Como sua esteira garante que o binário executado em produção é exatamente o mesmo que passou nos testes de homologação? Explique como as configurações mudam entre ambientes (dev, staging, prod) sem recompilar o artefato e como você identifica a versão no registry para que um rollback nunca puxe um binário sobrescrito por engano.
  > **Princípio: build once, deploy many.** A imagem é construída **uma única vez**, no merge para `main`. Homologação e produção recebem exatamente o mesmo artefato; nada é recompilado no caminho. Foi a recompilação da `main` que levou para produção um código diferente do testado no incidente.
  >
  > **Identidade do artefato:**
  > - **Tag imutável e rastreável:** `checkout:1.8.3-a1b2c3d` (versão semântica + SHA curto do commit). A tag `:latest` é proibida em qualquer manifesto.
  > - **Imutabilidade no registry:** tags imutáveis habilitadas (ECR `IMMUTABLE`, Harbor ou Artifact Registry). Se alguém tentar sobrescrever uma tag, o push falha.
  > - **Deploy por digest:** após o push, a esteira captura o `RepoDigest` (`checkout@sha256:...`), e os manifestos (Helm/Kustomize no repositório GitOps) referenciam o digest. A tag serve para pessoas lerem; o digest garante que o conteúdo é o mesmo byte a byte.
  > - **Promoção:** passar de staging para produção é só um commit no repositório GitOps trocando o digest no ambiente `prod`, ou uma cópia da imagem para o registry de produção com `crane copy`, que preserva o digest. Nunca um novo `docker build`.
  > - **Assinatura e proveniência:** a imagem é assinada com **Cosign** (Sigstore) e recebe uma atestação SLSA de proveniência (qual commit, qual workflow) e o SBOM. Em produção, um admission controller (Kyverno ou Sigstore policy-controller) só aceita imagens assinadas pela nossa esteira.
  > - **Rastreio em execução:** o `git-commit-id-maven-plugin` expõe o commit em `/actuator/info`, e o mesmo valor vai nos logs e nas métricas (`app_version`). Durante o canary, fica claro qual versão gerou cada erro.
  >
  > **Configuração por ambiente sem recompilar (12-Factor):**
  > - O mesmo jar lê a configuração do ambiente: `SPRING_PROFILES_ACTIVE`, variáveis de ambiente e `ConfigMap` montado (`spring.config.import=optional:configtree:/etc/config/`). O código não tem perfis de build.
  > - **Segredos** (senha do banco, chaves do PSP) vêm do Vault ou do Secrets Manager via External Secrets Operator, montados em tempo de execução. Nunca na imagem, nunca em variáveis de build.
  > - Diferenças entre ambientes ficam no repositório GitOps (`values-staging.yaml`, `values-prod.yaml`), revisadas por PR como qualquer código.
  >
  > **Rollback:** reverter o commit do GitOps para o digest anterior. A política de retenção do registry nunca apaga digests que estiveram em produção nos últimos N releases (ou 90 dias). Assim, o rollback sempre encontra o binário original, e não uma imagem reconstruída ou sobrescrita.

---

## 2. Estratégia de Rollout e Migração de Banco sem Downtime

### 2.1 O Modelo de Rollout em Produção

- **A Escolha:** (x) Canary Release (Liberação Gradual por Percentual)  ( ) Blue-Green Deployment (Alternância de Ambientes)
- **Justificativa e Mitigação das Desvantagens do seu Modelo:**
  - *Se você escolheu Canary Release:* Como o tráfego evoluirá sob 5.000 TPS, quais sinais definirão o avanço ou a reversão (rollback) automática e como lidar com o tempo em que duas versões diferentes da API atendem clientes simultaneamente?
  - *Se você escolheu Blue-Green Deployment:* Como você gerenciará o custo de infraestrutura duplicada no pico da Black Friday, o aquecimento das novas instâncias (conexões/cache) antes da virada de chave e o risco de virar 100% do tráfego de uma só vez?
  > **Por que Canary:** a 5.000 TPS, o que mais importa é limitar o **raio de impacto**. Com 1% do tráfego, um defeito afeta ~50 transações por segundo, e não todas. O Blue-Green exigiria dobrar a infraestrutura justamente no pico e viraria 100% do tráfego de uma vez. Além disso, como v1 e v2 compartilham o mesmo PostgreSQL, o Blue-Green não elimina a convivência de versões no banco: a migração precisa ser retrocompatível de qualquer forma (seção 2.2).
  >
  > **Ferramenta:** **Argo Rollouts** (ou Flagger) controlando o peso do tráfego no Istio / Gateway API. O rollout é declarativo, versionado no GitOps e com análise automática.
  >
  > **Evolução do tráfego:**
  > | Passo | Peso v2 | TPS aprox. | Pausa | Objetivo |
  > |---|---|---|---|---|
  > | 1 | 1% | ~50 | 10 min | Detectar erros grosseiros (boot, SQL, serialização) |
  > | 2 | 5% | ~250 | 10 min | Validar latência e métricas da Saga com amostra estatística |
  > | 3 | 20% | ~1.000 | 15 min | Validar pool de conexões, GC e lag do Kafka sob carga real |
  > | 4 | 50% | ~2.500 | 15 min | Comparação direta canary × estável |
  > | 5 | 100% | 5.000 | — | Promoção; v1 fica escalada para baixo, pronta para voltar |
  >
  > **Sinais de avanço e rollback automático** (`AnalysisTemplate` com consultas ao Prometheus, comparando canary com a versão estável no mesmo intervalo):
  > - **Técnicos:** taxa de 5xx do canary no máximo 0,5 ponto percentual acima da estável; p99 até 1,2× o da estável; conexões pendentes no HikariCP perto de zero; pausas de GC; reinícios de pod; erros de SQL (`column does not exist` encerra o rollout na hora).
  > - **De negócio (da Aula 04):** taxa de compensação da Saga, pedidos parados no mesmo estado além do prazo, lag do consumidor Kafka, mensagens na DLQ e taxa de aprovação do checkout.
  > - Duas medições ruins seguidas → `abort` automático: o peso volta a 0% em segundos, sem esperar alguém.
  >
  > **Duas versões atendendo ao mesmo tempo:**
  > - **API:** somente mudanças aditivas (campos novos opcionais, nada removido ou renomeado). Os clientes ignoram campos desconhecidos (*tolerant reader*, `FAIL_ON_UNKNOWN_PROPERTIES=false` no Jackson).
  > - **Mensagens Kafka e Outbox:** schemas no Schema Registry com compatibilidade `FULL`, para que v1 consuma o que v2 publica e vice-versa. A v2 não emite estado novo da Saga que a v1 não conheça até que a v1 seja totalmente retirada.
  > - **Idempotência:** as chaves continuam no Redis/PostgreSQL compartilhados (Aula 02). Um retry do cliente que cair na outra versão não gera cobrança dupla.
  > - **Desacoplar deploy de release:** comportamento novo atrás de *feature flag* (OpenFeature/Unleash), desligado por padrão. O canary valida o binário; a flag libera a funcionalidade depois, também gradualmente.
  > - **Aquecimento:** a `readinessProbe` só marca o pod como pronto depois do pool do HikariCP inicializado e de um warm-up do JIT. No Java 25, o **AOT Cache** do Projeto Leyden (JEPs 483/514/515) reduz o tempo de boot e de aquecimento, evitando picos de latência a cada passo.

### 2.2 Evolução de Schema Sem Downtime e Plano de Rollback

- **Passo a Passo da Migração:** Descreva em etapas sequenciais como evoluir a coluna `status_pagamento` para `estado_saga` no PostgreSQL garantindo que as instâncias da versão antiga (`v1`) e da versão nova (`v2`) operem juntas sem erros de SQL.
  > **Regras gerais (o que causou o incidente):**
  > - **Nunca renomear coluna em uma única etapa.** `RENAME COLUMN` é uma mudança destrutiva para qualquer instância que ainda use o nome antigo. Uso **Expand and Contract** (*Parallel Change*), dividido em várias releases.
  > - **Migração fora do boot da aplicação.** Nos pods, `spring.flyway.enabled=false`. O Flyway roda como um `Job` do Kubernetes (hook *PreSync* do Argo CD), uma única vez, com um usuário de banco que tem permissão de DDL. A aplicação usa um usuário só com DML (seção 3.1).
  > - **Toda migração tem `SET lock_timeout = '3s'`**, para que um `ALTER TABLE` esperando lock não faça fila e trave as 5.000 TPS atrás dele. Se falhar, o Job tenta de novo depois.
  >
  > **Release 1: Expand (só banco, compatível com v1)**
  > ```sql
  > -- V20__expand_add_estado_saga.sql
  > SET lock_timeout = '3s';
  > ALTER TABLE pedido ADD COLUMN estado_saga VARCHAR(30) NULL;
  > ALTER TABLE pedido ALTER COLUMN status_pagamento DROP NOT NULL;
  > ```
  > - No PostgreSQL 11+, adicionar uma coluna nula sem `DEFAULT` altera só o catálogo: é instantâneo, sem reescrever a tabela.
  > - Crio um *trigger* `BEFORE INSERT OR UPDATE` que copia o valor de uma coluna para a outra quando só uma for preenchida. Assim, tudo o que a v1 grava (só em `status_pagamento`) também aparece em `estado_saga`.
  > - A v1 continua funcionando sem mudança nenhuma: o Hibernate lista as colunas explicitamente no SQL e ignora a coluna nova.
  >
  > **Release 2: Migrate (aplicação v2 com escrita dupla + backfill)**
  > - A **v2 grava nas duas colunas** e lê `estado_saga` (com `COALESCE(estado_saga, status_pagamento)` até o backfill terminar).
  > - **Backfill em lotes** por um job separado: `UPDATE pedido SET estado_saga = status_pagamento WHERE id BETWEEN ? AND ? AND estado_saga IS NULL`, em lotes de 5 mil linhas, com pausa entre eles e fora do pico. Isso evita transações longas, picos de WAL e atraso nas réplicas de leitura (Aula 03).
  > - **Verificação:** `SELECT count(*) FROM pedido WHERE estado_saga IS DISTINCT FROM status_pagamento` precisa retornar 0.
  > - **Restrição sem travar:** `ADD CONSTRAINT ck_estado_saga CHECK (estado_saga IS NOT NULL) NOT VALID` e depois `VALIDATE CONSTRAINT`, que não bloqueia escritas. Índices na coluna nova com `CREATE INDEX CONCURRENTLY`, em script com `executeInTransaction=false` (arquivo `.conf` do Flyway).
  > - v1 e v2 convivem durante o canary sem erro de SQL: as duas colunas existem e estão sempre sincronizadas.
  >
  > **Release 3: v3 usa só `estado_saga`**
  > - Só depois de a v1 sair de todos os ambientes. A v3 lê e grava apenas `estado_saga`; o *trigger* continua mantendo `status_pagamento`, porque a v2 ainda é o alvo de rollback.
  >
  > **Release 4: Contract (destrutiva, em janela própria)**
  > ```sql
  > -- V24__contract_drop_status_pagamento.sql
  > SET lock_timeout = '3s';
  > DROP TRIGGER trg_sync_estado_saga ON pedido;
  > ALTER TABLE pedido DROP COLUMN status_pagamento;
  > ```
  > - Executada só quando nenhuma versão que dependa da coluna antiga puder voltar (fim da janela de rollback, e **nunca durante o congelamento de mudanças da Black Friday**). O linter de migração bloqueia `DROP`/`RENAME` em qualquer PR que não seja marcado como fase *contract* (seção 3.1).
- **Garantia de Rollback Imediato:** Caso a versão `v2` apresente degradação grave de performance com 20% do tráfego na Black Friday, qual é o procedimento exato de rollback e por que o seu desenho de banco de dados garante que a versão `v1` voltará a assumir 100% da carga sem quebrar nem perder dados gravados pela `v2`?
  > **Procedimento exato (cenário: v2 com 20% do tráfego e p99 degradado):**
  > 1. **Automático:** a análise do Argo Rollouts detecta o p99 acima de 1,2× o da v1 em duas medições seguidas e executa `abort`. O peso do canary volta a 0% em segundos, e a v1, que já atendia os 80% restantes, assume 100%.
  > 2. **Manual, se necessário:** `kubectl argo rollouts abort checkout` (ou o botão no painel). A pessoa de plantão não precisa de aprovação para reverter (seção 3.1).
  > 3. **Capacidade:** o HPA da v1 escala de acordo com a CPU e o RPS. No pico da Black Friday, a v1 é mantida com folga mínima (`minReplicas`) durante todo o canary, para absorver os 20% de volta sem esperar novos pods.
  > 4. **Fixar o estado:** reverter o commit do GitOps para o digest da v1 e pausar a esteira de deploy do Checkout até o post-mortem.
  > 5. **Não mexer no banco.** Nenhuma migração é desfeita: não existe *down migration* em produção. O schema da Release 1 é um superconjunto que atende v1 e v2.
  >
  > **Por que a v1 volta sem quebrar e sem perder dados:**
  > - **Schema compatível:** `status_pagamento` continua existindo. A v1 nunca vê erro de SQL, porque nada que ela usa foi removido ou renomeado.
  > - **Dados escritos pela v2 estão na coluna antiga:** a v2 faz escrita dupla e o *trigger* garante a sincronização mesmo em caminhos esquecidos. Todo pedido criado ou atualizado pela v2 tem `status_pagamento` preenchido, e a v1 consegue continuar a Saga desses pedidos.
  > - **Mensagens em trânsito:** eventos da Outbox e comandos da Saga publicados pela v2 seguem schemas `FULL` compatíveis, então os consumidores v1 os processam. A deduplicação por inbox (Aula 04) evita processamento duplicado na troca de versão.
  > - **Flyway não bloqueia a v1:** a migração não roda no boot dos pods. Mesmo que rodasse, o padrão `ignoreMigrationPatterns=*:future` faz a v1 aceitar um histórico com versões mais novas que as dela.
  > - **Binário original:** a v1 é puxada pelo digest, que é imutável e está retido no registry (seção 1.2). Esse é o rollback que falhou no incidente.

---

## 3. Guardrails de Segurança, Governança e Validação da Release

### 3.1 Gates de Segurança e Guardrails no Processo de Entrega

- **Guardrails do Commit ao Deploy:** Quais portões automatizados de qualidade e segurança devem proteger a sua esteira ao longo do fluxo de desenvolvimento (desde o Pull Request até a publicação e promoção da imagem)?
  > **No Pull Request (bloqueiam o merge via *required status checks*):**
  > - Branch `main` protegida: sem push direto, no mínimo 1 aprovação e `CODEOWNERS`, que exige aprovação de quem cuida do banco para qualquer arquivo em `db/migration/`.
  > - Build e testes unitários (`./mvnw -B verify`) e testes de integração com **Testcontainers** (PostgreSQL e Kafka reais).
  > - Análise estática (Error Prone/SpotBugs, Sonar com *quality gate* de cobertura no código novo) e **SAST** (CodeQL ou Semgrep).
  > - **Secret scanning** (gitleaks / GitHub push protection): commit com credencial é recusado.
  > - **SCA** das dependências (Dependabot, OWASP Dependency-Check) e checagem de licenças.
  > - **Linter de migração** (ex.: `squawk` para PostgreSQL): bloqueia `RENAME COLUMN`, `DROP` fora de PR de fase *contract*, `SET NOT NULL` direto em tabela grande, índice sem `CONCURRENTLY` e migração sem `lock_timeout`. Teria barrado o incidente ainda no PR.
  > - Checagem de compatibilidade dos schemas de mensagem no Schema Registry e testes de contrato da API.
  > - `hadolint` no Dockerfile (base fixada por digest, sem `USER root`, sem `latest`).
  >
  > **No merge para `main` (build e publicação):**
  > - Build único da imagem, **SBOM** CycloneDX (`cyclonedx:makeAggregateBom` + `trivy image --format cyclonedx`).
  > - **Trivy** na imagem: falha com CVE `CRITICAL` que tenha correção disponível (`--ignore-unfixed`), com exceções aceitas registradas em `.trivyignore` com prazo e justificativa.
  > - Testes da imagem e da migração (seção 3.2).
  > - Push com tag imutável, assinatura Cosign e atestações (SBOM e proveniência SLSA).
  >
  > **Na promoção (staging → produção):**
  > - Deploy automático em staging pelo digest, smoke e E2E, teste de carga (Gatling/k6) e ensaio do canary.
  > - Aprovação obrigatória no *environment* `production` do GitHub Actions, com janela de deploy e regra de congelamento da Black Friday.
  > - **Admission controller** (Kyverno/Gatekeeper) no cluster de produção: só aceita imagens do registry oficial, assinadas, referenciadas por digest, com `runAsNonRoot`, `readOnlyRootFilesystem` e limites de CPU/memória.
  > - Reescaneamento diário das imagens em execução: CVE nova em imagem já em produção abre alerta e PR de atualização da base.
- **Governança e Controle em Produção:** Quais *guardrails* de processo e arquitetura impedem que alterações manuais não rastreadas cheguem à produção e como equilibrar segurança rigorosa com agilidade de resposta durante incidentes críticos sem perder a trilha de auditoria?
  > **Guardrails contra alterações manuais não rastreadas:**
  > - **GitOps com Argo CD e *self-heal*:** o estado do cluster é o que está no Git. Um `kubectl edit` manual é detectado como *drift* e revertido automaticamente; a única forma de mudar produção é um PR revisado.
  > - **RBAC mínimo:** pessoas têm acesso somente leitura ao cluster de produção. Quem aplica mudanças é a conta de serviço do Argo CD.
  > - **Banco com separação de privilégios:** o usuário da aplicação só tem DML (`SELECT/INSERT/UPDATE`). O DDL é exclusivo do usuário do Job de migração, cuja credencial vive no Vault e só é entregue à esteira. Ninguém executa `ALTER TABLE` à mão, e o Flyway valida o *checksum* de cada migração já aplicada.
  > - **Trilha de auditoria:** histórico do Git e do Argo CD, *audit log* do Kubernetes, CloudTrail e `pgaudit` no PostgreSQL. Cada mudança responde "quem, quando, o quê e aprovado por quem".
  >
  > **Agilidade em incidentes sem perder auditoria:**
  > - **Rollback é sempre liberado:** reverter para um digest que já esteve em produção não exige aprovação extra, porque o artefato já passou por todos os gates. A aprovação é exigida para ir adiante, não para voltar.
  > - **Kill switch por feature flag:** desligar uma funcionalidade sem deploy, com cada alteração de flag registrada.
  > - **Hotfix pela esteira normal:** a correção urgente passa pelos mesmos gates de segurança, rodando em paralelo. Pulam-se só etapas demoradas sem relação com segurança (ex.: teste de carga longo), com registro explícito da exceção.
  > - ***Break-glass*:** acesso elevado temporário (ex.: 1 h), concedido sob demanda com MFA e justificativa vinculada ao incidente, sessão gravada (Teleport ou AWS SSM) e alerta automático para o time de segurança. Depois do incidente, toda mudança manual vira PR em até 24 h e entra no post-mortem.
  > - Métricas DORA (frequência de deploy, *lead time*, taxa de falha e tempo de recuperação) mostram se os guardrails estão protegendo sem travar o time.

### 3.2 Estratégia de Testes da Imagem e da Migração

- **Validação Pré-Tráfego:** Passar nos testes unitários durante o build do código não prova que a imagem final do container sobe nem que a migração de banco é retrocompatível.
- **Plano de Testes na Pipeline:** Descreva como você testará automaticamente a imagem construída e validará a coexistência da migração de banco de dados na esteira antes de liberar o deploy para o tráfego real de clientes.
  > **Objetivo:** antes de qualquer cliente, provar três coisas sobre o **mesmo digest** que irá para produção: (1) a imagem sobe com as restrições de produção, (2) a migração é aplicável sem travar, e (3) v1 e v2 convivem e a v1 sobrevive ao rollback.
  >
  > **Etapa 1: estrutura da imagem**
  > - `container-structure-test` ou checagens simples: usuário `65532` (`docker inspect --format '{{.Config.User}}'`), ausência de shell (`docker run --entrypoint sh` **deve falhar**), labels OCI com o commit e limite de tamanho da imagem.
  >
  > **Etapa 2: smoke test com restrições de produção**
  > - Subo a imagem recém-construída com Testcontainers (`GenericContainer` pelo digest) ao lado de um PostgreSQL da **mesma versão major de produção**, com `--read-only`, `--cap-drop ALL`, `--user 65532` e limite de memória.
  > - Valido `/actuator/health/readiness` = 200 e um checkout de ponta a ponta. Em seguida, envio `SIGTERM` com uma requisição em andamento e verifico que ela termina (*graceful shutdown*).
  >
  > **Etapa 3: migração isolada**
  > - `flyway validate` e `migrate` em banco vazio e em um banco restaurado de um snapshot anonimizado de produção (versão N → N+1), medindo duração e locks com `lock_timeout` ativo. Migração que estoura o tempo reprova a esteira.
  >
  > **Etapa 4: coexistência v1 × v2 e rollback (o teste principal)**
  > - Banco migrado para o schema novo; sobem **duas** imagens apontando para ele: a v1 (digest atual de produção, puxado do registry) e a v2 (digest candidato).
  > ```java
  > @Test
  > void v1EV2ConvivemNoSchemaExpandidoESobrevivemAoRollback() {
  >     flyway.migrate(); // aplica V20__expand_add_estado_saga
  >     var v1 = checkoutContainer(DIGEST_PRODUCAO);
  >     var v2 = checkoutContainer(DIGEST_CANDIDATO);
  >     Startables.deepStart(v1, v2).join();
  >
  >     var pedidoV1 = criarPedido(v1);           // v1 grava só status_pagamento
  >     var pedidoV2 = criarPedido(v2);           // v2 grava as duas colunas
  >     avancarSaga(v2, pedidoV1);                // v2 atualiza pedido criado pela v1
  >
  >     assertThat(jdbc.queryForObject("""
  >         SELECT count(*) FROM pedido
  >         WHERE estado_saga IS DISTINCT FROM status_pagamento""", Long.class)).isZero();
  >
  >     v2.stop();                                // simula o rollback
  >     assertThat(consultarStatus(v1, pedidoV2)).isEqualTo("PAGO");
  >     assertThat(logs(v1)).doesNotContain("does not exist");
  > }
  > ```
  > - O teste falha se a v2 depender de algo que a v1 não tem, ou se a migração quebrar a v1. É exatamente o cenário do incidente, reproduzido na esteira.
  >
  > **Etapa 5: ensaio do canary em staging**
  > - Deploy pelo Argo Rollouts em staging com carga sintética (k6/Gatling) e um build propositalmente defeituoso (flag que injeta erros 5xx), para provar que a análise automática executa o `abort` e devolve o tráfego para a v1. Só então a promoção para produção fica liberada.
