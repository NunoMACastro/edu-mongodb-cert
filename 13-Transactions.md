# Transações, sessões e consistência

## Objetivos do capítulo

- Saber quando uma transação multi-documento é necessária.
- Distinguir atomicidade single-document de ACID multi-documento.
- Usar sessions e `withTransaction()` no driver atual.
- Compreender retries, concerns e restrições de transações.
- Evitar side effects duplicados e transações longas.

---

## Conceitos Fundamentais

### Vocabulário das transações

| Conceito             | Definição                                                                                                  |
| -------------------- | ---------------------------------------------------------------------------------------------------------- |
| Transação            | conjunto de operações MongoDB coordenadas como uma unidade all-or-nothing                                 |
| Invariante           | regra de negócio que deve permanecer verdadeira antes e depois da operação                               |
| `ClientSession`      | contexto lógico criado pelo `MongoClient` que acompanha operações, causalidade e estado transacional     |
| Transaction callback | função assíncrona que contém as operações executadas pela Convenient Transaction API                     |
| Commit               | tentativa de tornar permanentes e visíveis as alterações da transação                                    |
| Abort                | término da transação sem aplicar as suas alterações MongoDB                                              |
| Snapshot             | visão consistente dos dados usada pelas leituras da transação conforme as garantias configuradas         |
| Concern              | configuração das garantias de leitura ou acknowledgement/durabilidade de escrita                         |
| Retry                | repetição controlada da transação ou do commit perante erros transientes elegíveis                       |

Uma session não é uma connection dedicada nem uma transação por si só. É o contexto que tem de ser passado às operações que pertencem à transação.

### O que é uma transação?

MongoDB garante atomicidade de cada write num único documento. Uma transação é apropriada quando uma invariante abrange vários documentos/collections e nenhum estado intermédio pode ser observado.

Exemplos:

- transferir saldo entre duas contas;
- criar reserva e ocupar inventário separado;
- gravar entidade e ledger que têm de concordar.

Não usar transação quando embedding permite colocar a invariante num documento, quando eventual consistency é aceitável ou quando a coordenação envolve sistemas externos que MongoDB não consegue incluir atomicamente.

### Lifecycle conceptual

```text
criar ClientSession
       |
       v
iniciar transação
       |
       v
executar operações sequenciais com { session }
       |
       +-- erro/regra falha --> abort
       |
       `-- operações válidas --> commit
                                  |
                                  `-- resultado incerto/transiente --> retry segundo o protocolo
```

Na Convenient API, `withTransaction()` coordena este lifecycle e pode voltar a executar a callback. Na Core API, a aplicação assume explicitamente `startTransaction()`, commit, abort e retry logic. Um abort descarta alterações MongoDB da transação, mas não desfaz emails, pagamentos ou outros side effects externos.

### ACID

- **Atomicity:** todas as operações commitam ou nenhuma.
- **Consistency:** constraints/invariantes mantêm-se se a aplicação as implementar corretamente.
- **Isolation:** operações da transação observam uma snapshot/isolamento conforme concerns.
- **Durability:** write concern determina acknowledgement/durabilidade.

“ACID” não valida regras de negócio automaticamente. Uma transação que debita uma conta errada continua atomicamente errada.

### Topology

Transações multi-documento exigem replica set ou sharded cluster; standalone não suporta. As versões mínimas históricas diferem para replica sets e sharded clusters, mas com o driver atual deve consultar-se sempre a compatibility matrix corrente.

### Core API versus Convenient API

| API        | Responsabilidade                                                            |
| ---------- | --------------------------------------------------------------------------- |
| Core       | `startTransaction`, commit, abort e retry logic manual                      |
| Convenient | `withTransaction` faz commit/abort e retries de erros transientes elegíveis |

Preferir `withTransaction()` salvo necessidade específica. Mesmo com a API conveniente, validar resultados dentro da callback.

### Sessions

Uma `ClientSession` pertence ao `MongoClient` que a criou. Todas as operações da transação recebem `{ session }`. Omitir a session executa a operação fora da transação, um erro especialmente perigoso porque o código “funciona”.

### Callback pode repetir

`withTransaction` pode executar a callback novamente em erros transientes. A callback não deve enviar email, cobrar uma API externa ou publicar uma mensagem sem estratégia idempotente/outbox. Writes MongoDB dentro da transação são geridos pelo protocolo; side effects externos não são revertidos.

> **Importante para o exame:** cada operação da transação recebe a mesma `session` criada pelo mesmo `MongoClient`, e as operações são aguardadas sequencialmente. O driver Node.js não suporta `Promise.all()` nem outras operações paralelas dentro de uma única transação.

---

## Funcionamento Interno

Uma session dá identidade e ordenação causal. Ao iniciar a transação, o driver associa transaction number e opções. O servidor mantém uma snapshot coerente e buffers/estado até commit ou abort.

Num replica set, commit coordena a persistência no primary e replicação segundo write concern. Num sharded cluster, vários shards podem participar e o commit exige coordenação distribuída, com maior latência.

### Erros e retries

Labels relevantes:

- `TransientTransactionError`: repetir a transação completa.
- `UnknownTransactionCommitResult`: o resultado do commit é incerto; repetir o commit segundo protocolo.

A Convenient API incorpora esta lógica. Retry loops manuais precisam de limites/política e não devem substituir o comportamento documentado.

### Lifetime e locks

Por defeito, a lifetime de transação é limitada (tipicamente menos de um minuto; configuração server/Atlas pode alterar). Transações esperam pouco tempo por locks e podem abortar em conflito. Manter a callback curta, sem input do utilizador, HTTP externo ou processamento pesado.

Transações longas retêm snapshots, aumentam pressão de cache e podem atrasar limpeza de versões. Cada documento continua sujeito ao limite BSON de 16 MiB.

### Read/write concern

Combinação típica:

```javascript
const transactionOptions = {
    readPreference: "primary",
    readConcern: { level: "snapshot" },
    writeConcern: { w: "majority" },
};
```

Escolher garantias pelo domínio. Dentro de transação, evitar alterar concerns por operação; configurar transaction options.

---

## Sintaxe

### Convenient Transaction API

Existem dois objetos com ciclos de vida diferentes:

```text
ClientSession -> começa antes da tentativa e termina no finally
transaction   -> começa/termina dentro da session e pode ter tentativas repetidas
```

A assinatura conceptual principal é:

```javascript
const callbackResult = await session.withTransaction(callback, options);
```

`callback` contém as operações que pertencem à unidade transacional. `options` define concerns, routing e limite de commit para as tentativas geridas pela Convenient API.

```javascript
const session = client.startSession();

try {
    const result = await session.withTransaction(
        async () => {
            const first = await collectionA.updateOne(filterA, updateA, {
                session,
            });

            const second = await collectionB.insertOne(documentB, { session });

            return { first, second };
        },
        {
            readPreference: "primary",
            readConcern: { level: "snapshot" },
            writeConcern: { w: "majority" },
            maxCommitTimeMS: 5_000,
        },
    );

    console.log(result);
} finally {
    await session.endSession();
}
```

Leitura por etapas:

1. `startSession()` cria uma `ClientSession`; ainda não executa a transação de negócio.
2. `withTransaction()` inicia a transação, invoca a callback e tenta commit.
3. Todas as operações participantes recebem **a mesma** `{ session }` nas options.
4. `return { first, second }` define o valor devolvido por `withTransaction()` após sucesso; não é o commit result.
5. `readPreference: "primary"` encaminha as reads transacionais para o primary.
6. `readConcern: { level: "snapshot" }` pede uma visão consistente suportada da transação.
7. `writeConcern: { w: "majority" }` configura acknowledgement do commit.
8. `maxCommitTimeMS` limita a fase de commit, não toda a execução da callback.
9. `endSession()` liberta a session tanto após sucesso como após erro.

A callback pode ser executada mais do que uma vez quando a API trata um erro transiente. Por isso, não deve enviar email, cobrar um serviço externo ou produzir outro side effect não idempotente dentro da callback sem desenho específico.

### Core API

```javascript
session.startTransaction(options);

try {
    await collection.updateOne(filter, update, { session });
    await session.commitTransaction();
} catch (error) {
    await session.abortTransaction();
    throw error;
} finally {
    await session.endSession();
}
```

Este esqueleto não implementa os retries exigidos para todos os labels; por isso a Convenient API é mais segura como default.

Na Core API, a aplicação controla `startTransaction()`, `commitTransaction()` e `abortTransaction()`. Isso também transfere para a aplicação a responsabilidade de interpretar error labels e implementar os ciclos de retry corretos. `abortTransaction()` no `catch` desfaz o trabalho MongoDB não confirmado; não desfaz side effects externos.

### Sem paralelismo dentro da transação

Não usar:

```javascript
await Promise.all([
    collectionA.updateOne(filterA, updateA, { session }),
    collectionB.updateOne(filterB, updateB, { session }),
]);
```

O Node.js Driver não suporta operações paralelas numa única transação. Executá-las sequencialmente.

`Promise.all()` tentaria usar a mesma session/transaction em operações concorrentes. A correção é aguardar a primeira operação antes de iniciar a segunda. Isto é uma restrição do contrato da transação no driver, não apenas uma preferência de estilo.

---

## Exemplos

### Exemplo 1 — transferência atómica

```javascript
/**
 * @file Transfere um valor entre contas e regista o movimento.
 */
import { Decimal128, MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);
const fromAccountId = new ObjectId(process.env.FROM_ACCOUNT_ID);
const toAccountId = new ObjectId(process.env.TO_ACCOUNT_ID);
const amount = Decimal128.fromString("50.00");
const negativeAmount = Decimal128.fromString("-50.00");
const transferId = new ObjectId();

try {
    const database = client.db("bank");
    const accounts = database.collection("accounts");
    const transfers = database.collection("transfers");
    const session = client.startSession();

    try {
        await session.withTransaction(
            async () => {
                const debit = await accounts.updateOne(
                    {
                        _id: fromAccountId,
                        balance: { $gte: amount },
                    },
                    { $inc: { balance: negativeAmount } },
                    { session },
                );

                if (debit.modifiedCount !== 1) {
                    throw new Error(
                        "Conta de origem inexistente ou saldo insuficiente.",
                    );
                }

                const credit = await accounts.updateOne(
                    { _id: toAccountId },
                    { $inc: { balance: amount } },
                    { session },
                );

                if (credit.modifiedCount !== 1) {
                    throw new Error("Conta de destino inexistente.");
                }

                await transfers.insertOne(
                    {
                        _id: transferId,
                        fromAccountId,
                        toAccountId,
                        amount,
                        createdAt: new Date(),
                    },
                    { session },
                );
            },
            {
                readPreference: "primary",
                readConcern: { level: "snapshot" },
                writeConcern: { w: "majority" },
                maxCommitTimeMS: 5_000,
            },
        );
    } finally {
        await session.endSession();
    }
} finally {
    await client.close();
}
```

Resultado: débito, crédito e ledger commitam juntos ou são abortados.

### Exemplo 2 — reservar inventário e criar encomenda

```javascript
/**
 * @file Cria encomenda apenas quando existe stock suficiente.
 */
import { MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);
const productId = new ObjectId(process.env.PRODUCT_ID);
const orderId = new ObjectId();
const quantity = 2;

try {
    const database = client.db("shop");
    const inventory = database.collection("inventory");
    const orders = database.collection("orders");
    const session = client.startSession();

    try {
        const createdOrder = await session.withTransaction(async () => {
            const stockResult = await inventory.updateOne(
                { productId, available: { $gte: quantity } },
                { $inc: { available: -quantity, reserved: quantity } },
                { session },
            );

            if (stockResult.modifiedCount !== 1) {
                throw new Error("Stock insuficiente.");
            }

            const order = {
                _id: orderId,
                status: "reserved",
                items: [{ productId, quantity }],
                createdAt: new Date(),
            };

            await orders.insertOne(order, { session });
            return order;
        });

        console.log(createdOrder);
    } finally {
        await session.endSession();
    }
} finally {
    await client.close();
}
```

Resultado: stock e encomenda mantêm-se coerentes; a callback pode repetir e não contém side effects externos.

### Exemplo 3 — padrão outbox

```javascript
/**
 * @file Grava a alteração de domínio e uma mensagem outbox na mesma transação.
 */
import { MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);
const orderId = new ObjectId(process.env.ORDER_ID);

try {
    const database = client.db("shop");
    const orders = database.collection("orders");
    const outbox = database.collection("outbox");
    const session = client.startSession();

    try {
        await session.withTransaction(async () => {
            const order = await orders.findOneAndUpdate(
                { _id: orderId, status: "reserved" },
                {
                    $set: {
                        status: "confirmed",
                        confirmedAt: new Date(),
                    },
                },
                { session, returnDocument: "after" },
            );

            if (!order) {
                throw new Error("Encomenda inexistente ou estado inválido.");
            }

            await outbox.insertOne(
                {
                    eventId: new ObjectId(),
                    type: "order.confirmed",
                    aggregateId: order._id,
                    payload: { orderId: order._id },
                    publishedAt: null,
                    createdAt: new Date(),
                },
                { session },
            );
        });
    } finally {
        await session.endSession();
    }
} finally {
    await client.close();
}
```

Resultado: a mensagem só existe se a alteração de encomenda commitar. Outro worker publica-a com idempotência.

---

## Explicação linha a linha

### Exemplo 1

1. Usa `Decimal128` para saldo e sinais explícitos.
2. Cria session a partir do mesmo client das collections.
3. O débito inclui saldo mínimo no filtro.
4. Valida `modifiedCount` antes de continuar.
5. Crédito e ledger usam a mesma session.
6. Não existe `Promise.all`.
7. Transaction options pedem snapshot/majority.
8. Session e client são fechados em blocos distintos.

### Exemplo 2

1. O filtro faz compare-and-set de stock.
2. `$inc` reserva atomicamente no documento de inventário.
3. Falha lança e causa abort.
4. A encomenda é inserida na mesma session.
5. O valor retornado pela callback é devolvido por `withTransaction` após commit.
6. Não há envio de email dentro da callback repetível.

### Exemplo 3

1. `findOneAndUpdate` valida estado e devolve o novo documento.
2. `null` aborta a regra.
3. O outbox event é inserido na mesma transação.
4. A publicação externa fica para um worker depois do commit.
5. `eventId` permite deduplicação/idempotência no consumidor.

---

## Casos Reais

- **Transferência:** saldos e ledger.
- **Reserva:** stock e encomenda.
- **Contabilidade:** lançamento e contrapartida.
- **Outbox:** estado de domínio e evento durável.
- **Migração consistente:** múltiplas collections, em batches curtos.

Não usar transação para:

- envolver uma chamada HTTP externa;
- manter um pedido aberto enquanto o utilizador decide;
- compensar um schema que podia embebedar dados coesos;
- processar milhões de documentos num único bloco.

---

## Performance

Transações acrescentam round trips, tracking de session/snapshot e commit. Cross-shard é mais caro do que single-shard/replica set. Manter poucos documentos, pouca duração e operações indexadas.

Uma query lenta dentro da transação prolonga locks/snapshot. Criar índices antes. Evitar scans e blocking aggregations. Não aumentar a lifetime como primeira correção.

Concorrência pode causar aborts; retries aumentam carga. Monitorizar taxa de abort/retry e reduzir contenção no modelo/chaves. Uma “hot account” continua um hotspot mesmo com transação correta.

---

## Armadilhas comuns

- **Usar transação para um único documento:** write já é atómico.
- **Esquecer `{ session }` numa operação:** corre fora da transação.
- **Usar session de outro `MongoClient`:** erro.
- **`Promise.all` dentro da transação:** não suportado.
- **Side effect externo na callback:** pode duplicar em retry.
- **Não validar result objects:** transação pode commitar sem cumprir a intenção.
- **Manter transação aberta durante I/O externo.**
- **Assumir suporte em standalone.**
- **Implementar retry manual incompleto com Core API.**
- **Confundir write concern com validação de negócio.**
- **Achar que abort desfaz um email/API call.**

---

## O que costuma aparecer no exame

- Atomicidade single-document.
- Quando usar multi-document transaction.
- Session e passagem de `{ session }`.
- `withTransaction` versus Core API.
- Commit, abort e retries.
- `TransientTransactionError` e `UnknownTransactionCommitResult`.
- Ausência de paralelismo numa transação.
- Replica set/sharded versus standalone.
- Read concern, write concern e read preference.
- Callback repetível e side effects.

---

## Resumo

Modelar primeiro; transacionar quando uma invariante multi-documento exige all-or-nothing. A session pertence ao client e deve acompanhar cada operação. `withTransaction` gere commit/abort e retries elegíveis, mas a callback pode repetir: sem side effects externos. Validar contadores e resultados dentro da callback. Manter transações curtas, sequenciais, indexadas e com concerns adequados.

Fontes oficiais: [transactions no driver](https://www.mongodb.com/docs/drivers/node/current/crud/transactions/), [transactions no servidor](https://www.mongodb.com/docs/manual/core/transactions/), [production considerations](https://www.mongodb.com/docs/manual/core/transactions-production-consideration/) e [causal consistency](https://www.mongodb.com/docs/manual/core/causal-consistency-read-write-concerns/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — Todas as operações de uma transação recebem a mesma `session` criada pelo mesmo `MongoClient` e executam sequencialmente. O driver Node.js não suporta `Promise.all()` dentro de uma única transação.

> ### ARMADILHA
>
> `withTransaction()` pode repetir a callback. Email, pagamento HTTP ou publicação externa dentro da callback podem acontecer mais de uma vez e não são revertidos pelo abort. Usa outbox/idempotência e mantém side effects fora da callback repetível.

> ### DICA DE MEMORIZAÇÃO
>
> **Primeiro modela; depois transaciona. Mesma session, passos em série, callback repetível.**

> ### COMPARAÇÃO
>
> A necessidade de coordenação determina o mecanismo.

| Situação                          | Mecanismo preferido      | Razão                            |
| --------------------------------- | ------------------------ | -------------------------------- |
| um documento                      | write normal             | já é atómico                     |
| vários documentos, all-or-nothing | transação                | invariante multi-documento       |
| sistema externo + MongoDB         | idempotência/outbox/saga | transação não inclui API externa |
| API conveniente                   | `withTransaction()`      | retries/commit/abort geridos     |
| controlo manual específico        | Core API                 | retry logic fica na aplicação    |

> **Ligação entre capítulos:** modelação/embedding está no capítulo 02; unique indexes no 09; sharding e coordenação cross-shard no 10; lifecycle do client no 04.

### Fluxograma: preciso de transação?

```text
A invariante cabe num único documento?
  |-- sim --> write single-document atómico
  `-- não --> estado intermédio é aceitável?
                |-- sim --> consistência eventual / processamento idempotente
                `-- não --> todos os recursos estão no mesmo deployment MongoDB?
                              |-- sim --> transação curta
                              `-- não --> outbox / saga / compensação
```

### Mini desafio

Uma callback de `withTransaction()` debita saldo, grava ledger e envia email. Identifica que operações podem permanecer na callback, qual deve sair e que padrão preserva a intenção sem assumir execução única.

---

## Resumo Rápido

- Write single-document já é atómico.
- Transação serve invariantes multi-documento all-or-nothing.
- Mesma `session` em todas as operações.
- Operações sequenciais; não usar `Promise.all()`.
- `withTransaction()` pode repetir a callback.
- Manter transações curtas, indexadas e sem I/O externo.

---

## Checklist

- [ ] Tento primeiro resolver a invariante pelo modelo.
- [ ] Sei quando uma transação é realmente necessária.
- [ ] Uso replica set ou sharded cluster compatível.
- [ ] Passo a mesma `session` a todas as operações.
- [ ] Evito paralelismo na transação.
- [ ] Valido result objects dentro da callback.
- [ ] Mantenho side effects externos fora da callback.
- [ ] Distingo Convenient API de Core API.

---

## Perguntas para confirmar conhecimentos

1. Porque um update de vários fields num documento não precisa de transação?
2. Que topologias suportam transações multi-documento?
3. O que acontece se uma operação omitir `{ session }`?
4. Porque a session tem de pertencer ao mesmo `MongoClient`?
5. Porque `Promise.all()` não é suportado numa transação do driver?
6. Em que erros `withTransaction()` pode repetir trabalho?
7. Porque side effects externos são perigosos na callback?
8. Que diferença existe entre Convenient Transaction API e Core API?
9. Como índices e duração influenciam conflitos/aborts?
10. Quando um outbox é preferível a enviar diretamente uma mensagem?
