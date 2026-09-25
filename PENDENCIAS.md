# Pendências — ChainTracker (infra)

Atualizado: 2026-09-25

---

## Contexto rápido

Repositório de **infraestrutura e documentação** do ChainTracker. Não tem código
de aplicação — o que vive aqui é o `docker-compose.yaml` que sobe o ambiente de
desenvolvimento, o `realm-export.json` do Keycloak e o `ARQUITETURA.md`.

**Os serviços de aplicação moram em repositórios separados:**

| Repositório | Porta | Papel |
|---|---|---|
| `identity-service` | 8080 | Autenticação (casca sobre o Keycloak) |
| `assetMarketDataService` | 8082 | Cotações (CoinGecko, Alchemy) |
| `portfolioService` | 8087 | Carteiras e posições |

Cada um tem o próprio `PENDENCIAS.md`.

**O que o compose sobe:**

| Serviço | Imagem | Porta | Observação |
|---|---|---|---|
| `identity-service-db` | postgres:17 | 5432 | Também serve de banco do Keycloak |
| `assetData-service-db` | postgres:17 | 5433 | |
| `portfolioService-db` | postgres:17 | 5434 | |
| `kafka` | apache/kafka:4.3.1 | 9092 | KRaft, nó único |
| `kafka-ui` | kafbat/kafka-ui | 8083 | |
| `keycloak` | quay.io/keycloak/keycloak | 8081 | `start-dev` |
| `redis` | redis:latest | 6379 | **Compartilhado** entre os serviços |

---

## 🔴 Bloqueadores

### 1. O realm do Keycloak não é reproduzível a partir do repositório

O `realm-export.json` está versionado, mas **nada o usa**. O serviço do Keycloak
no `docker-compose.yaml:88-102` roda `command: start-dev`, sem `--import-realm`,
e não há volume montando o arquivo.

**Consequência:** o realm `ChainTracker` — com os clients `identity-service` e os
usuários de teste — foi configurado **à mão** na interface do Keycloak. Quem
clonar o repositório sobe o ambiente e não tem realm nenhum; e o
`realm-export.json` é um retrato que envelhece a cada mudança feita na UI, sem
ninguém perceber.

Isso é o que torna o client secret do `identity-service` tão chato de resolver:
não há uma fonte de verdade para o valor.

**Fix:**

1. Montar o arquivo e importar na subida:
   ```yaml
   volumes:
     - ./realm-export.json:/opt/keycloak/data/import/realm-export.json:ro
   command: start-dev --import-realm
   ```
2. Reexportar o realm sempre que a configuração mudar, e **commitar o export**
   como parte da mudança.
3. Lembrar que o Keycloak mascara secrets no export (`"secret": "**********"`),
   então o valor real precisa vir de variável de ambiente — não do arquivo.

### 2. Credenciais de desenvolvimento hardcoded

O `docker-compose.yaml` tem, em texto puro: a senha dos três Postgres
(`POSTGRES_PASSWORD`, linhas 11, 21 e 32), a senha do banco do Keycloak
(`KC_DB_PASSWORD:97`), a senha do admin do Keycloak (`KC_BOOTSTRAP_ADMIN_PASSWORD:99`)
e a senha do Redis (`--requirepass`, linha 110).

São credenciais **locais de desenvolvimento**, e o repositório é privado — o risco
é menor do que o das API keys do `assetMarketDataService`. Mas o hábito é o mesmo
que produziu o secret vazado no `identity-service`, e o dia em que alguém subir
isso num servidor, a senha vai junto.

**Fix:** um `.env` na raiz (coberto pelo `.gitignore`) e `${VAR}` no compose, com
um `.env.example` versionado mostrando quais variáveis existem.

---

## 🟡 Importante

### 3. `depends_on` sem healthcheck nos bancos

O Keycloak declara `depends_on: identity-service-db` (`:101-102`), mas na forma
de lista — que espera só o **container iniciar**, não o Postgres **aceitar
conexão**. Os três serviços de Postgres também não têm `healthcheck`.

**Consequência:** em subida a frio (`docker compose up` com volumes vazios), o
Keycloak tenta conectar num Postgres que ainda está inicializando e morre com
erro de conexão. No segundo `up` funciona, o que faz parecer intermitente.

O `kafka-ui` faz certo (`:84-86`, `condition: service_healthy`) — é o padrão a
seguir.

**Fix:** `healthcheck` com `pg_isready` nos três Postgres, e trocar o
`depends_on` do Keycloak para a forma longa com `condition: service_healthy`.

### 4. Redis compartilhado — é um contrato, não um detalhe

O `redis` (`:103-111`) é um container só, e os três serviços apontam para ele na
mesma porta e **mesmo database index** (nenhum define `spring.data.redis.database`,
então todos usam o DB 0).

Isso é intencional e funciona: o `assetMarketDataService` escreve `coin:{id}` e o
`portfolioService` **lê a mesma chave** para não precisar de uma chamada HTTP por
posição. Mas significa que:

- Nada no compilador garante que os dois lados concordem no formato do JSON.
  Mudar o `CoinMarketResponse` de um lado quebra o outro em silêncio.
- Qualquer serviço pode sobrescrever a chave de qualquer outro. Não há namespace
  por serviço.
- Um `FLUSHALL` para depurar o cache de um serviço limpa o de todos.

**Fix (quando incomodar):** prefixar as chaves por serviço (`asset:coin:{id}`) ou
dar um database index por serviço (`spring.data.redis.database`). Nada disso é
urgente com três serviços; vira urgente no quarto.

### 5. Keycloak em modo `start-dev`

`:100` — `command: start-dev`. Modo de desenvolvimento: banco em H2 por padrão
(aqui está em Postgres, o que é melhor), sem cache de cluster, e **aceita HTTP
sem TLS**. Correto para agora, mas é uma linha que não pode ir para produção.

### 6. Postgres sem política de restart

`redis`, `kafka` e `kafka-ui` têm `restart: unless-stopped`; os três Postgres e o
Keycloak **não têm**. Depois de reiniciar a máquina, os bancos não voltam sozinhos
e os serviços de aplicação sobem contra um banco que não existe.

---

## 🔵 Limpeza

### 7. Não existe `.gitignore` na raiz

O repositório não tem `.gitignore`, e o `.idea/` aparecia como não rastreado.
Boas práticas de IDE não devem ser versionadas — o `.idea/.gitignore` que está
**rastreado** (`git ls-files` mostra ele) é justamente o oposto disso: cobre
`workspace.xml` mas deixa `misc.xml`, `modules.xml` e `dbnavigator.xml` soltos.

**Fix:** um `.gitignore` na raiz com `.idea/` e `*.iml`, e remover o
`.idea/.gitignore` do versionamento.

### 8. `version: '3.8'` é obsoleto

`docker-compose.yaml:1`. O Compose v2 ignora a chave e emite aviso. Pode sair.

### 9. `keycloak` sem `container_name`

Todos os outros serviços têm `container_name`; o Keycloak não, então o container
nasce como `chaintracker-keycloak-1` e o nome muda se o diretório for renomeado.
Inconsistência pequena, mas atrapalha script que referencie o container.

### 10. `ARQUITETURA.md` descreve mais do que existe

O documento (678 linhas) descreve Ledger como fonte de verdade, projeção de
Portfolio, alertas e ingestão de múltiplas blockchains e corretoras. O que existe
são três serviços e nenhum Ledger.

Não é um erro — é o alvo. Vale manter uma seção de "o que já existe vs. o que é
planejado" para quem ler não achar que o Ledger está implementado.

---

## 🧪 DX e testes

### 11. Sem script de subida

Não há `Makefile` nem script de conveniência. Subir o ambiente exige lembrar de
`docker compose up -d` na raiz e depois rodar cada serviço na IDE. Um `make up`
(ou `scripts/dev.sh`) que sobe o compose, espera os healthchecks e avisa quando
está pronto economiza isso a cada reset de máquina.

### 12. Sem Dockerfile de aplicação

Nenhum dos três serviços tem `Dockerfile`. Hoje todos rodam pela IDE, o que é
adequado para desenvolvimento — mas não há caminho para rodar o conjunto todo
containerizado.

---

## ✅ Já resolvido (não refazer)

- `KAFKA_NUM_PARTITIONS` reduzido de 3 para 1: com um broker só, partição extra
  não traz paralelismo e só espalha mensagem.
- `KC_DB_URL` corrigido para apontar para `identity-service-db` (era `postgres`,
  que não resolvia).
- Volumes nomeados para os três Postgres, o Kafka e o Redis — os dados sobrevivem
  a `docker compose down`.
- Healthcheck do Kafka testando o listener interno (`29092`), que é o correto
  para checagem de dentro do container.
