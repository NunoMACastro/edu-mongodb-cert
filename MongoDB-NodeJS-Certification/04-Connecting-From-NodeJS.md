# Ligação a MongoDB a partir de Node.js

## Objetivos do capítulo

- Configurar e reutilizar `MongoClient` no driver 7.x.
- Compreender discovery, server selection e connection pools.
- Diferenciar os timeouts de ligação e operação.
- Aplicar Stable API, TLS, secrets e encerramento correto.
- Diagnosticar erros sem criar retry loops perigosos.

---

## Conceitos Fundamentais

### Um cliente por processo

`MongoClient` representa mais do que um socket: contém visão da topologia, monitoring e pools por servidor. Numa aplicação server-side de longa duração, deve normalmente existir uma instância partilhada por URI/configuração. `db()` e `collection()` são handles baratos e podem ser obtidos sem criar novos clientes.

Criar um cliente por request:

- repete DNS, TLS, handshake e autenticação;
- aumenta sockets e latência;
- causa connection storms;
- perde a capacidade do pool absorver concorrência.

Scripts e CLIs de curta duração devem fechar o cliente num `finally`. Um servidor fecha-o no graceful shutdown, não no fim de cada request.

### Ligação explícita versus implícita

`new MongoClient()` não abre imediatamente todas as ligações. `await client.connect()` força a ligação inicial/handshake. Uma operação também pode iniciar ligação implicitamente. Usar `connect()` no startup é útil para falhar cedo; não é obrigatório antes de cada operação.

### Pool

Cada servidor da topologia tem um pool. Opções importantes:

| Opção | Default documentado | Função |
|---|---:|---|
| `maxPoolSize` | 100 | máximo de ligações em uso + disponíveis por pool |
| `minPoolSize` | 0 | mínimo mantido |
| `maxConnecting` | 2 | criações concorrentes |
| `maxIdleTimeMS` | 0 | idade idle ilimitada |
| `waitQueueTimeoutMS` | 0 | espera ilimitada por checkout |

`maxPoolSize` limita ligações usadas por operações. Além do pool, cada `MongoClient` abre até duas ligações de monitorização por servidor da topologia; estas não entram nesse limite.

Não dimensionar `maxPoolSize` isoladamente: processos × pools × membros pode gerar muitas ligações. Um pool enorme não torna o servidor mais rápido; pode apenas aumentar concorrência e filas a jusante.

### Stable API

A Stable API permite declarar uma versão de API do servidor. `strict` ajuda a rejeitar comandos fora da API declarada; `deprecationErrors` revela uso de funcionalidades descontinuadas nessa API. Não congela o query planner, performance, versão do driver nem todas as opções.

### Concerns e preferences

- **read preference** decide para que membros podem ir reads.
- **read concern** define garantias de isolamento/consistência da leitura.
- **write concern** define acknowledgement/durabilidade exigidos ao write.

`readPreference: "secondary"` não é um botão genérico de performance: pode devolver dados com replication lag e requer índices nos secondaries, que normalmente são os mesmos do replica set.

> **Importante para o exame:** reutiliza um `MongoClient` por configuração, porque cada client gere discovery e pools próprios. `serverSelectionTimeoutMS`, `connectTimeoutMS`, `socketTimeoutMS` e `waitQueueTimeoutMS` limitam fases diferentes e não são sinónimos.

---

## Funcionamento Interno

### SDAM e seleção

O driver implementa Server Discovery and Monitoring. Monitoring connections observam membros e atualizam o estado. Uma operação:

1. determina se é read ou write;
2. filtra servidores por topologia/read preference;
3. aplica tags, staleness e latency window;
4. escolhe um candidato;
5. faz checkout de uma ligação do pool;
6. executa o comando e devolve a ligação.

Uma eleição pode deixar temporariamente sem primary. Retryable writes/reads cobrem operações elegíveis segundo regras do driver, mas não substituem retries de negócio ou idempotency keys.

### Fases e timeouts

~~~text
server selection → pool checkout → socket/connect → comando no servidor → resposta
~~~

- `serverSelectionTimeoutMS`: encontrar servidor.
- `waitQueueTimeoutMS`: obter ligação do pool.
- `connectTimeoutMS`: estabelecer socket.
- `socketTimeoutMS`: inatividade do socket.
- `timeoutMS`/CSOT quando suportado: orçamento cliente para a operação.
- `maxTimeMS`: limite server-side para a execução de uma operação.

O nome “connection timeout” é ambíguo; no diagnóstico, identificar a fase.

### Serialização e promoção de tipos

O driver converte JavaScript para BSON. Tipos BSON sem equivalente seguro usam classes do package `mongodb`/`bson`. Numeric promotion pode converter Int32 para Number; Long fora do safe integer range exige cuidado e opções/tipos apropriados.

---

## Sintaxe

### Cliente de produção

~~~javascript
import { MongoClient, ServerApiVersion } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI, {
  appName: "orders-api",
  maxPoolSize: 50,
  minPoolSize: 0,
  maxConnecting: 2,
  waitQueueTimeoutMS: 2_000,
  serverSelectionTimeoutMS: 5_000,
  connectTimeoutMS: 10_000,
  retryReads: true,
  retryWrites: true,
  serverApi: {
    version: ServerApiVersion.v1,
    strict: true,
    deprecationErrors: true
  }
});
~~~

Os valores são exemplos, não defaults universais para qualquer workload. Devem derivar de SLO, concorrência, número de processos e capacidade do deployment.

### Selecionar database e collection

~~~javascript
const database = client.db("sample_mflix");
const movies = database.collection("movies");
~~~

Isto cria handles. A primeira operação realiza I/O se ainda não houver ligação.

### Encerrar

~~~javascript
await client.close();
~~~

No driver 7.4+, resource management com `await using` está estável em runtimes compatíveis, mas `try/finally` continua explícito e amplamente compreendido. Não misturar ciclos de vida: quem recebe um client partilhado não o deve fechar.

---

## Exemplos

### Exemplo 1 — módulo partilhado com startup e shutdown

~~~javascript
/**
 * @file Gere o único MongoClient do processo e o respetivo ciclo de vida.
 */
import { MongoClient, ServerApiVersion } from "mongodb";

const uri = process.env.MONGODB_URI;

if (!uri) {
  throw new Error("MONGODB_URI é obrigatória.");
}

export const mongoClient = new MongoClient(uri, {
  appName: "catalog-api",
  maxPoolSize: 40,
  waitQueueTimeoutMS: 2_000,
  serverSelectionTimeoutMS: 5_000,
  serverApi: {
    version: ServerApiVersion.v1,
    strict: true,
    deprecationErrors: true
  }
});

/**
 * Estabelece a ligação inicial e prova uma round trip ao servidor.
 */
export async function connectToMongoDB() {
  await mongoClient.connect();
  await mongoClient.db("admin").command({ ping: 1 });
}

/**
 * Fecha pools e monitoring durante o graceful shutdown.
 */
export async function disconnectFromMongoDB() {
  await mongoClient.close();
}
~~~

Resultado: um módulo importável que não cria clientes por request.

### Exemplo 2 — servidor HTTP com ciclo de vida correto

~~~javascript
/**
 * @file Expõe um endpoint de health sem criar clientes por pedido.
 */
import http from "node:http";
import {
  connectToMongoDB,
  disconnectFromMongoDB,
  mongoClient
} from "./mongo-client.mjs";

await connectToMongoDB();

const server = http.createServer(async (request, response) => {
  if (request.url !== "/movies/count" || request.method !== "GET") {
    response.writeHead(404).end();
    return;
  }

  try {
    const movies = mongoClient.db("sample_mflix").collection("movies");
    const count = await movies.countDocuments(
      { year: { $gte: 2000 } },
      { maxTimeMS: 2_000 }
    );

    response.writeHead(200, { "content-type": "application/json" });
    response.end(JSON.stringify({ count }));
  } catch {
    response.writeHead(503, { "content-type": "application/json" });
    response.end(JSON.stringify({ error: "database_unavailable" }));
  }
});

server.listen(3000);

let isShuttingDown = false;

/**
 * Para de aceitar pedidos, aguarda os pedidos ativos e fecha o cliente.
 *
 * @param {NodeJS.Signals} signal Signal que iniciou o encerramento.
 */
async function shutdown(signal) {
  if (isShuttingDown) {
    return;
  }

  isShuttingDown = true;

  try {
    await new Promise((resolve, reject) => {
      server.close((error) => {
        if (error) {
          reject(error);
          return;
        }

        resolve();
      });
    });
  } catch (error) {
    console.error("Falha ao fechar o servidor HTTP.", { signal, error });
    process.exitCode = 1;
  } finally {
    try {
      await disconnectFromMongoDB();
    } catch (error) {
      console.error("Falha ao fechar o MongoClient.", { signal, error });
      process.exitCode = 1;
    }
  }
}

process.once("SIGINT", () => void shutdown("SIGINT"));
process.once("SIGTERM", () => void shutdown("SIGTERM"));
~~~

Resultado: cada pedido reutiliza o pool; o shutdown deixa de aceitar novos pedidos, aguarda os pedidos ativos e só depois fecha o client.

### Exemplo 3 — separar erro de configuração de falha operacional

~~~javascript
/**
 * @file Executa um health check e preserva a causa original do erro.
 */
import { MongoClient } from "mongodb";

const uri = process.env.MONGODB_URI;

if (!uri) {
  throw new Error("Configuração inválida: MONGODB_URI ausente.");
}

const client = new MongoClient(uri, {
  appName: "diagnostic-check",
  serverSelectionTimeoutMS: 3_000
});

try {
  await client.connect();
  await client.db("admin").command({ ping: 1 });
  console.log("MongoDB disponível.");
} catch (error) {
  console.error("Falha ao ligar ou executar ping.", {
    name: error.name,
    message: error.message
  });
  process.exitCode = 1;
} finally {
  await client.close();
}
~~~

Resultado: exit code 0 em sucesso e 1 em falha, sem imprimir a URI.

---

## Explicação linha a linha

### Exemplo 1

1. O módulo valida a URI ao carregar.
2. Exporta uma única instância `MongoClient`.
3. Define limites explícitos do pool e seleção.
4. `connectToMongoDB()` força handshake e ping.
5. `disconnectFromMongoDB()` centraliza ownership do encerramento.
6. Consumers usam `mongoClient.db()`, mas não fecham o client.

### Exemplo 2

1. O startup aguarda MongoDB antes de abrir o serviço.
2. O handler valida route e method.
3. Obtém um handle da instância partilhada.
4. `countDocuments()` recebe filtro e `maxTimeMS`.
5. A resposta de erro não expõe detalhes internos.
6. `server.close()` deixa de aceitar novos pedidos.
7. Signals chamam o graceful shutdown uma única vez.

### Exemplo 3

1. Configuração ausente falha antes de qualquer rede.
2. O timeout limita a seleção no diagnóstico.
3. O `catch` preserva nome e mensagem, mas não secrets.
4. `process.exitCode` permite executar `finally`.
5. O client fecha mesmo quando `connect()` falha parcialmente.

---

## Casos Reais

- **API Node.js:** singleton por processo, pool limitado e shutdown por signals.
- **Serverless:** reutilizar client em âmbito de módulo entre invocações warm; considerar limites do provider.
- **Worker:** usar cursor/batches com o mesmo client durante a vida do worker.
- **Multi-tenant:** não criar um client por tenant se partilham deployment e credenciais; separar apenas quando a configuração o exige.
- **Health endpoints:** distinguir liveness de readiness; uma dependency lenta não deve reiniciar o processo sem política.

---

## Performance

O throughput aumenta com concorrência até CPU, storage, locks/tickets, rede ou pool saturar. A partir daí, mais ligações elevam queueing. Medir checkout latency, command latency e server metrics antes de alterar `maxPoolSize`.

`minPoolSize > 0` pode reduzir cold-start, mas mantém sockets mesmo sem carga. `maxIdleTimeMS` ajuda em ambientes com proxies/load balancers. Compressão reduz bytes e aumenta CPU; beneficia payloads compressíveis e redes caras, não automaticamente todas as operações.

Read preference secundária pode distribuir reads, mas replication lag altera frescura. Para read-after-write, usar primary ou garantias de sessão/concerns adequadas.

---

## Armadilhas comuns

- **Cliente por pedido:** causa handshake e socket churn.
- **Chamar `connect()` em cada função:** é redundante e confunde ownership.
- **Fechar client partilhado num repository:** quebra outros consumers.
- **Aumentar pool para resolver query lenta:** pode agravar saturação.
- **Usar `socketTimeoutMS` como único orçamento:** não cobre todas as fases.
- **Retry manual sem limite:** cria retry storms.
- **Retry de operação não idempotente:** pode duplicar efeitos externos.
- **Logar objeto de configuração/URI:** pode revelar password.
- **`tlsInsecure: true` em produção:** desativa verificações essenciais.
- **Ler em secondary sem aceitar staleness:** viola expectativa de frescura.
- **Confundir Stable API v1 com server v1:** não existe essa equivalência.

---

## O que costuma aparecer no exame

- Ciclo de vida de `MongoClient`.
- Pool e impacto de criar clientes repetidamente.
- `connect()` explícito versus ligação na primeira operação.
- `db()` e `collection()` como handles.
- Server selection e tipos de timeout.
- Read preference, read concern e write concern.
- Retryable reads/writes.
- Stable API.
- Encerramento com `finally`.
- Uso de environment variables e proteção de secrets.

---

## Resumo

Uma aplicação Node.js deve reutilizar um `MongoClient` por configuração, porque o client gere discovery, monitoring e pools. `connect()` é útil no startup; handles de database/collection são baratos. Timeouts correspondem a fases distintas. Pools precisam de limites alinhados com a capacidade, não máximos arbitrários. Stable API, TLS, least privilege e secrets externos reduzem risco. Retries do driver cobrem operações elegíveis, não toda a semântica da aplicação.

Fontes oficiais: [ligar com o Node.js Driver](https://www.mongodb.com/docs/drivers/node/current/connect/), [connection options](https://www.mongodb.com/docs/drivers/node/current/connect/connection-options/), [connection pools](https://www.mongodb.com/docs/drivers/node/current/connect/connection-options/connection-pools/), [Stable API](https://www.mongodb.com/docs/drivers/node/current/connect/stable-api/) e [release notes do driver](https://www.mongodb.com/docs/drivers/node/current/reference/release-notes/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — Um `MongoClient` representa topology discovery, monitoring e pools; não é apenas uma ligação. Reutilizá-lo por configuração é a regra. `db()` e `collection()` devolvem handles baratos e não justificam clients adicionais.

> ### ARMADILHA
>
> `serverSelectionTimeoutMS` não é “o timeout da query”. Pode terminar antes de existir socket escolhido. `waitQueueTimeoutMS` limita espera pelo pool; `connectTimeoutMS` limita estabelecimento de socket; `socketTimeoutMS` trata inatividade no socket; `maxTimeMS` é server-side e pertence à operação.

> ### DICA DE MEMORIZAÇÃO
>
> **Selecionar → Esperar pool → Ligar → Comunicar → Executar.** Cada fase pode ter um limite diferente.

> ### COMPARAÇÃO
>
> Escolhe o timeout pela fase que pretendes limitar.

| Opção | Fase | Sintoma típico |
|---|---|---|
| `serverSelectionTimeoutMS` | encontrar servidor elegível | topology sem candidato |
| `waitQueueTimeoutMS` | obter socket do pool | pool saturado |
| `connectTimeoutMS` | abrir ligação | TCP/TLS lento |
| `socketTimeoutMS` | I/O no socket | comunicação sem progresso |
| `maxTimeMS` | execução no servidor | comando excede prazo |

> **Ligação entre capítulos:** a URI e TLS vêm do capítulo 03; cursores e backpressure dos capítulos 07–08; sessions e transactions do capítulo 12.

### Fluxo de ciclo de vida

~~~text
Processo inicia
   |
   v
criar 1 MongoClient por configuração
   |
   v
connect/ping no startup (opcional, útil para fail-fast)
   |
   v
reutilizar db/collection + pool em todos os pedidos
   |
   v
parar listener -> aguardar pedidos -> close() no shutdown
~~~

### Mini desafio

Uma API cria `new MongoClient(uri)` dentro de cada handler e fecha-o em `finally`. Identifica pelo menos quatro custos ou riscos e propõe o ownership correto sem alterar a semântica das queries.

---

## Resumo Rápido

- Reutilizar um `MongoClient` por configuração e processo.
- Cada servidor da topologia tem pool e sockets de monitorização.
- `db()` e `collection()` são handles baratos.
- `connect()` permite falhar cedo; não se chama por request.
- Timeouts limitam fases diferentes.
- Fechar o client depois de parar e drenar o servidor.

---

## Checklist

- [ ] Sei explicar o que um `MongoClient` gere.
- [ ] Evito client por request ou repository.
- [ ] Distingo `connect()` explícito de ligação implícita.
- [ ] Distingo todos os timeouts principais.
- [ ] Dimensiono pool com processos e topologia.
- [ ] Sei o alcance de retryable reads/writes.
- [ ] Não confundo Stable API com versão do driver.
- [ ] Implemento shutdown com ownership correto.

---

## Perguntas para confirmar conhecimentos

1. Porque criar um `MongoClient` por pedido degrada a aplicação?
2. Que recursos são mantidos por cada client?
3. É obrigatório chamar `connect()` antes da primeira operação? Que vantagem pode ter?
4. Porque `db()` e `collection()` não abrem um novo pool?
5. O que limita `serverSelectionTimeoutMS`?
6. Que opção limita o tempo à espera de uma ligação livre no pool?
7. Porque aumentar `maxPoolSize` pode piorar queueing a jusante?
8. O que Stable API v1 garante e o que não garante?
9. Porque um retry do driver não torna um side effect externo idempotente?
10. Qual deve ser a ordem de encerramento entre listener HTTP e `MongoClient`?
