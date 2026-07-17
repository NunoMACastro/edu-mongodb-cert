# Resumo final para o exame MongoDB Associate Developer (Node.js)

## Objetivos do capítulo

- Consolidar os capítulos numa revisão técnica única.
- Consultar operadores, stages, tipos, índices, sharding e métodos em tabelas.
- Comparar APIs frequentemente confundidas.
- Rever decisões de performance, consistência e segurança.
- Usar uma checklist e os 110 conceitos essenciais antes do exame.

---

## Conceitos Fundamentais

### Como usar este resumo

Este ficheiro é deliberadamente uma ferramenta de **recuperação e decisão**, não a primeira explicação da matéria. As tabelas seguintes condensam sintaxe e diferenças que já devem ter sido construídas nos capítulos pedagógicos. Se uma linha parecer surgir “do nada”, não deve ser memorizada isoladamente: deve regressar-se ao capítulo que explica o problema, o modelo mental, a anatomia da API e os exemplos progressivos.

Percurso recomendado:

1. estudar o capítulo completo e executar os exemplos;
2. responder às perguntas desse capítulo sem consultar o texto;
3. usar este resumo para ligar conceitos e localizar fraquezas;
4. regressar ao capítulo sempre que não se conseguir explicar **porquê** uma opção está correta.

| Se a dúvida é sobre...                       | Explicação de base                                                                 |
| -------------------------------------------- | ---------------------------------------------------------------------------------- |
| documentos, BSON, embedding e referências   | [02 — Modelo documental](02-Document-Model.md)                                     |
| URI, autenticação e `mongosh`                | [03 — Connection strings e shell](03-Connection-Strings-e-MongoShell.md)           |
| client, pool, timeouts e lifecycle           | [04 — Ligação a partir de Node.js](04-Connecting-From-NodeJS.md)                   |
| inserts, reads e filtros                     | [05 — CRUD: insert e find](05-CRUD-Insert-Find.md)                                 |
| updates, replacements e deletes              | [06 — CRUD: mutações](06-CRUD-Update-Replace-Delete.md)                            |
| sort, projection, limit, count e paginação   | [07 — Resultados e paginação](07-CRUD-Sort-Limit-Projection.md)                    |
| repositories, bulk e compound operations     | [08 — CRUD no Node.js Driver](08-CRUD-NodeJS.md)                                   |
| key patterns, criação e desenho de índices   | [09 — Índices](09-Indexes.md)                                                      |
| shard keys, routing e distribuição           | [10 — Sharding](10-Sharding.md)                                                    |
| stages, expressions e accumulators           | [11 — Aggregation Framework](11-Aggregation.md)                                    |
| cursor e builders de aggregation em Node.js  | [12 — Aggregation no Node.js Driver](12-Aggregation-NodeJS.md)                     |
| sessions, callbacks e retries transacionais  | [13 — Transações](13-Transactions.md)                                              |
| analyzers, search index, operadores e score  | [14 — MongoDB Search](14-Atlas-Search.md)                                          |

### Resumo de todos os capítulos

| Capítulo                 | Ideia central                                       | Detalhe examinável                        |
| ------------------------ | --------------------------------------------------- | ----------------------------------------- |
| 00 — Introdução          | driver, server, Atlas e shell são camadas distintas | tipos de retorno e baseline atual         |
| 01 — Atlas               | control plane, data plane, identidade e rede        | Atlas user ≠ database user                |
| 02 — Document Model      | modelar pelos access patterns                       | embedding, references, BSON e atomicidade |
| 03 — Connection Strings  | URI descreve discovery e autenticação               | SRV, `authSource` e percent-encoding      |
| 04 — Connecting Node.js  | reutilizar um `MongoClient`                         | pools, server selection e timeouts        |
| 05 — Insert/Find         | writes e filtros MQL                                | cursor, arrays, `null` e tipos            |
| 06 — Update/Delete       | expressar a alteração no servidor                   | update ≠ replace; contadores              |
| 07 — Result Modifiers    | ordenar/projetar/paginar                            | projection válida e sort determinístico   |
| 08 — CRUD Node.js        | cada método tem retorno próprio                     | compound operations e bulk                |
| 09 — Indexes             | menos read work por custo de write/storage          | prefixos, ESR, multikey e explain         |
| 10 — Sharding            | distribuir dados e workload horizontalmente          | shard key, targeting e scatter-gather     |
| 11 — Aggregation         | pipeline ordered de transformações                  | stages, expressions, accumulators         |
| 12 — Aggregation Node.js | `aggregate()` devolve cursor                        | batch, BSON e construção segura           |
| 13 — Transactions        | ACID multi-document quando necessário               | session, retries e callback repetível     |
| 14 — Search              | índice invertido e relevância                       | mappings, analyzers e `compound`          |

### Modelo mental de uma operação

1. **Input:** validar tipo, dimensão e autorização.
2. **MQL:** construir filtro/stages com allowlist.
3. **Driver:** escolher método e options; entender o retorno.
4. **Topology/pool:** selecionar servidor e ligação.
5. **Routing:** num sharded cluster, determinar os shards participantes.
6. **Planner:** escolher `COLLSCAN`/índice/pipeline plan em cada executor.
7. **Storage:** ler/escrever BSON e atualizar índices.
8. **Consistency:** aplicar concerns/session/transaction.
9. **Output:** projetar, limitar, iterar e não expor dados.
10. **Evidence:** verificar result object, fan-out e `explain`.

### Diferenças entre métodos e conceitos frequentemente confundidos

| A                  | B                          | Diferença decisiva                               |
| ------------------ | -------------------------- | ------------------------------------------------ |
| `find()`           | `findOne()`                | cursor zero-muitos vs documento/`null`           |
| `updateOne()`      | `replaceOne()`             | alteração parcial vs substituição total          |
| `updateOne()`      | `findOneAndUpdate()`       | `UpdateResult` vs documento atómico              |
| `deleteOne()`      | `findOneAndDelete()`       | `DeleteResult` vs documento eliminado            |
| `insertMany()`     | transação                  | batch não é all-or-nothing                       |
| `$set` update      | `$set` stage               | update operator vs aggregation stage             |
| `$push`            | `$addToSet`                | acrescenta sempre vs evita nova duplicação exata |
| `$in`              | `$all`                     | qualquer valor vs todos no array                 |
| `$elemMatch` query | `$elemMatch` projection    | escolhe documento vs limita array devolvido      |
| `$match`           | `find()` filter            | stage de pipeline vs query direta                |
| `aggregate()`      | `find()`                   | transformação multi-stage vs read direta         |
| `countDocuments()` | `estimatedDocumentCount()` | filtro exato vs estimativa global                |
| `skip` pagination  | keyset pagination          | offset crescente vs range por última chave       |
| projection         | schema                     | output da query vs forma persistida              |
| embedding          | referencing                | coesão/atomicidade vs independência/partilha     |
| replica set        | sharded cluster            | redundância vs distribuição                      |
| shard key          | query index                | onde procurar vs como procurar localmente        |
| targeted query     | scatter-gather             | shard(s) relevante(s) vs broadcast               |
| primary shard      | replica set primary        | local de unsharded data vs membro que aceita write |
| replication        | backup                     | disponibilidade atual vs recuperação histórica   |
| Atlas user         | database user              | gestão Atlas vs autenticação ao server           |
| default database   | `authSource`               | handle inicial vs origem de autenticação         |
| `mongodb://`       | `mongodb+srv://`           | seed list explícita vs DNS discovery             |
| read preference    | read concern               | routing vs garantia de leitura                   |
| write concern      | transação                  | acknowledgement vs atomicidade multi-write       |
| Stable API         | driver version             | API server declarada vs package Node             |
| B-tree index       | search index               | filtro/sort vs full-text relevance               |
| text index         | MongoDB Search             | `$text` básico vs Search rico                    |
| `$search`          | `$searchMeta`              | documentos vs metadata                           |
| `must`             | `filter` Search            | obrigatório com score vs obrigatório sem score   |
| `batchSize`        | `limit`                    | tamanho de batch vs total máximo                 |
| `matchedCount`     | `modifiedCount`            | encontrou vs alterou                             |
| unique index       | check-then-insert          | constraint concorrente vs race                   |

> **Importante para o exame:** usa estas distinções para explicar o comportamento, não para decorar frases isoladas. As notas deste material são prioridades de estudo baseadas no syllabus e na documentação pública, não perguntas reais nem conteúdo confidencial do exame.

---

## Funcionamento Interno

### Caminho end-to-end

```text
Node.js object
  → BSON serialization
  → server selection
  → pool checkout
  → command over TLS
  → mongos routing quando a topologia é sharded
  → authz + query planning
  → index/storage execution
  → BSON response/batches
  → JavaScript values or cursor
```

O `MongoClient` mantém discovery, monitoring e pools. Uma operação seleciona um servidor, obtém uma ligação e envia um comando. Num sharded cluster, `mongos` usa a shard key e a metadata de ranges para escolher os shards participantes. Cada shard planeia localmente; resultados multi-shard podem exigir merge. Índices B-tree mantêm keys ordenadas; multikey produz keys para arrays. Cursors transferem batches e conservam estado até serem esgotados/fechados.

Writes atualizam documento e índices. Cada write single-document é atómico. Uma transação cria uma snapshot/coordenação multi-documento; `withTransaction` pode repetir callback. Search usa um índice derivado/invertido e atribui relevância, com sincronização não necessariamente imediata.

### Quatro limites de custo

- **Cardinalidade:** quantos documentos entram e saem em cada passo.
- **Working set:** dados/índices que cabem em cache.
- **Round trips:** número e latência das interações.
- **Coordenação:** locks, replication, transactions e shards.

Otimizar sem medir um destes eixos tende a trocar um problema por outro.

---

## Sintaxe

### Tabela geral de operadores de query

| Família    | Operadores principais                                 | Uso                        |
| ---------- | ----------------------------------------------------- | -------------------------- |
| comparação | `$eq $ne $gt $gte $lt $lte $in $nin`                  | valores/ranges/listas      |
| lógica     | `$and $or $nor $not`                                  | combinar condições         |
| elemento   | `$exists $type`                                       | presença e tipo            |
| array      | `$all $elemMatch $size`                               | conteúdo de arrays         |
| avaliação  | `$expr $regex $text $mod`                             | expressions/padrões/text   |
| geospatial | `$geoWithin $geoIntersects $near $nearSphere`         | localização                |
| bitwise    | `$bitsAllClear $bitsAllSet $bitsAnyClear $bitsAnySet` | bit masks                  |
| misc.      | `$jsonSchema`                                         | validação/match por schema |

### Tabela de comparison operators

| Operador | Corresponde quando                          |
| -------- | ------------------------------------------- |
| `$eq`    | valor igual, incluindo tipo/semântica BSON  |
| `$ne`    | valor diferente; pode incluir campo ausente |
| `$gt`    | maior                                       |
| `$gte`   | maior ou igual                              |
| `$lt`    | menor                                       |
| `$lte`   | menor ou igual                              |
| `$in`    | igual a qualquer elemento da lista          |
| `$nin`   | não igual a nenhum; pode incluir missing    |

### Tabela de logical operators

| Operador      | Semântica                                        |
| ------------- | ------------------------------------------------ |
| AND implícito | todas as chaves de topo têm de passar            |
| `$and`        | todas as expressões; útil para chave repetida    |
| `$or`         | pelo menos uma expressão                         |
| `$nor`        | nenhuma expressão                                |
| `$not`        | nega a expressão do field, com atenção a missing |

### Tabela de array operators

| Operador/contexto       | Semântica                                   |
| ----------------------- | ------------------------------------------- |
| igualdade scalar        | array contém o scalar                       |
| igualdade array         | array completo, incluindo ordem             |
| `$all`                  | contém todos os valores pedidos             |
| `$elemMatch` query      | um mesmo elemento cumpre todas as condições |
| `$size`                 | tamanho exato                               |
| dot notation            | percorre subdocumentos/arrays               |
| `$elemMatch` projection | devolve primeiro elemento correspondente    |
| `$slice` projection     | devolve subset do array                     |
| `$` update              | primeiro elemento correspondente            |
| `$[]` update            | todos os elementos                          |
| `$[id]` update          | elementos que passam `arrayFilters`         |

### Tabela de update operators

| Operador       | Efeito                      | Atenção                               |
| -------------- | --------------------------- | ------------------------------------- |
| `$set`         | atribui field/path          | preserva o resto                      |
| `$unset`       | remove field                | valor fornecido é irrelevante         |
| `$inc`         | soma número                 | falha em tipo incompatível            |
| `$mul`         | multiplica                  | semântica numérica                    |
| `$min`         | substitui se novo for menor | comparação BSON                       |
| `$max`         | substitui se novo for maior | comparação BSON                       |
| `$rename`      | renomeia field              | não usar em arrays                    |
| `$currentDate` | Date/Timestamp atual        | relógio do servidor                   |
| `$setOnInsert` | só no ramo insert de upsert | inicialização                         |
| `$push`        | acrescenta ao array         | aceita `$each/$slice/$sort/$position` |
| `$addToSet`    | acrescenta se não existir   | igualdade exata                       |
| `$pop`         | remove primeiro/último      | `-1`/`1`                              |
| `$pull`        | remove matches              | condição sobre elementos              |
| `$pullAll`     | remove valores listados     | igualdade                             |
| `$bit`         | operação bitwise            | integer fields                        |

### Tabela de aggregation stages

| Stage                       | Função                         | Performance/semântica            |
| --------------------------- | ------------------------------ | -------------------------------- |
| `$match`                    | filtra                         | cedo pode usar índice            |
| `$project`                  | inclui/exclui/calcula          | muda forma                       |
| `$set/$addFields`           | adiciona/substitui             | nomes equivalentes               |
| `$unset`                    | remove fields                  | projection simplificada          |
| `$replaceWith/$replaceRoot` | troca root                     | exige documento                  |
| `$unwind`                   | expande array                  | multiplica cardinalidade         |
| `$group`                    | agrupa/acumula                 | blocking                         |
| `$sort`                     | ordena                         | blocking sem índice              |
| `$limit`                    | limita                         | combina bem com sort             |
| `$skip`                     | descarta prefixo               | caro em offsets grandes          |
| `$count`                    | conta input                    | emite documento                  |
| `$sortByCount`              | group+count+sort               | atalho                           |
| `$lookup`                   | join à foreign collection      | output array; indexar foreign    |
| `$unionWith`                | concatena collection/pipeline  | pode aumentar muito input        |
| `$facet`                    | subpipelines paralelas lógicas | um documento com arrays          |
| `$bucket`                   | intervalos explícitos          | boundaries ordered               |
| `$bucketAuto`               | intervalos automáticos         | distribuição aproximada          |
| `$sample`                   | amostra                        | custo depende da posição/tamanho |
| `$setWindowFields`          | cálculos em janelas            | sort/partition e memória         |
| `$geoNear`                  | proximidade                    | primeiro stage e geo index       |
| `$search`                   | documentos full-text           | primeiro stage; search index     |
| `$searchMeta`               | metadata search                | primeiro stage                   |
| `$out`                      | substitui/escreve output       | altera dados                     |
| `$merge`                    | merge configurável             | altera dados                     |

### Accumulators e expressions frequentes

| Operador               | Contexto               | Resultado                  |
| ---------------------- | ---------------------- | -------------------------- |
| `$sum`                 | group/expression       | soma ou contagem com `1`   |
| `$avg`                 | group/expression       | média                      |
| `$min/$max`            | group/expression       | extremo                    |
| `$first/$last`         | group/window           | depende de ordem           |
| `$push`                | group                  | array com todos            |
| `$addToSet`            | group                  | array de valores únicos    |
| `$count` accumulator   | group/window suportado | contagem                   |
| `$multiply/$divide`    | expression             | aritmética                 |
| `$concat`              | expression             | strings                    |
| `$cond/$switch`        | expression             | condição                   |
| `$dateToString`        | expression             | formatação de Date         |
| `$ifNull`              | expression             | fallback para null/missing |
| `$map/$filter/$reduce` | expression             | transformação de arrays    |

### Tabela de index types

| Tipo         | Chave/exemplo               | Serve para               |
| ------------ | --------------------------- | ------------------------ |
| `_id`        | `{ _id: 1 }`                | identidade unique        |
| single-field | `{ email: 1 }`              | filtro/sort simples      |
| compound     | `{ status: 1, date: -1 }`   | access pattern combinado |
| multikey     | field array                 | elementos de array       |
| unique       | option `unique`             | constraint               |
| partial      | `partialFilterExpression`   | subset controlado        |
| sparse       | option `sparse`             | omitir missing           |
| TTL          | Date + `expireAfterSeconds` | expiração assíncrona     |
| text         | `{ field: "text" }`         | `$text` clássico         |
| 2dsphere     | GeoJSON                     | geospatial esférico      |
| 2d           | coordenadas legacy/plano    | geospatial planar        |
| hashed       | `{ key: "hashed" }`         | equality/hashed sharding |
| wildcard     | `{ "attrs.$**": 1 }`        | paths variáveis          |
| clustered    | opção de collection         | ordem de armazenamento   |
| Search       | search index definition     | full-text relevance      |

### Tabela de sharding

| Conceito | Decide/representa | Sinal principal |
| -------- | ----------------- | --------------- |
| replica set | redundância do mesmo dataset | high availability |
| sharded cluster | distribuição horizontal | vários shards |
| shard | parte dos dados sharded | normalmente um replica set |
| `mongos` | routing e merge | interface da aplicação ao cluster |
| config servers | metadata de ranges e topologia | não guardam o dataset como backup |
| shard key | distribuição e routing | cardinalidade, frequência, monotonicidade |
| ranged | preserva proximidade de valores | favorece range queries |
| hashed | distribui pelo hash | favorece equality e dispersão de writes |
| targeted query | consulta shard(s) relevante(s) | filtro contém informação da shard key |
| scatter-gather | consulta todos os shards | filtro não permite targeting suficiente |
| balancer | migra ranges | corrige desequilíbrio com custo operacional |
| refinement | acrescenta sufixo à shard key | preserva a key existente |
| resharding | redefine ou redistribui a collection | operação administrativa planeada |

### Tabela de BSON types

| Tipo BSON  | Representação Node comum | Nota                    |
| ---------- | ------------------------ | ----------------------- |
| Double     | `number`                 | IEEE-754                |
| String     | `string`                 | UTF-8                   |
| Object     | object                   | ordem preservada        |
| Array      | array                    | ordered                 |
| Binary     | `Binary/Uint8Array`      | subtypes                |
| Undefined  | legado                   | evitar                  |
| ObjectId   | `ObjectId`               | 12 bytes                |
| Boolean    | `boolean`                | true/false              |
| Date       | `Date`                   | UTC milliseconds        |
| Null       | `null`                   | distinto de missing     |
| Regex      | `RegExp/BSONRegExp`      | padrão                  |
| JavaScript | `Code`                   | uso raro                |
| Int32      | `Int32/number`           | 32-bit                  |
| Timestamp  | `Timestamp`              | interno/oplog, não Date |
| Int64/Long | `Long/BigInt options`    | atenção a safe integer  |
| Decimal128 | `Decimal128`             | base-10                 |
| MinKey     | `MinKey`                 | sentinel mínimo         |
| MaxKey     | `MaxKey`                 | sentinel máximo         |

Ordem simplificada de sort BSON: MinKey, Null, Numbers, String/Symbol, Object, Array, BinData, ObjectId, Boolean, Date, Timestamp, Regex, Code, Code with scope, MaxKey.

### Tabela de métodos de collection/mongosh

| Método                      | Devolve/efeito         |
| --------------------------- | ---------------------- |
| `insertOne/insertMany`      | result de insert       |
| `find`                      | cursor                 |
| `findOne`                   | documento/`null`       |
| `updateOne/updateMany`      | update result          |
| `replaceOne`                | replacement result     |
| `deleteOne/deleteMany`      | delete result          |
| `findOneAndUpdate`          | documento antes/depois |
| `aggregate`                 | cursor                 |
| `countDocuments`            | contagem por filtro    |
| `estimatedDocumentCount`    | estimativa total       |
| `distinct`                  | valores distintos      |
| `createIndex/createIndexes` | nome(s)                |
| `listIndexes`               | cursor                 |
| `dropIndex`                 | remove índice          |
| `bulkWrite`                 | bulk result            |

### Tabela de métodos do Node Driver

| Objeto          | Método/propriedade          | Função/retorno           |
| --------------- | --------------------------- | ------------------------ |
| `MongoClient`   | constructor                 | configura client/pools   |
| `MongoClient`   | `connect()`                 | handshake inicial        |
| `MongoClient`   | `db(name)`                  | handle `Db`              |
| `MongoClient`   | `startSession()`            | `ClientSession`          |
| `MongoClient`   | `close()`                   | fecha pools/monitoring   |
| `Db`            | `collection(name)`          | handle `Collection`      |
| `Db`            | `command()`                 | comando raw deliberado   |
| `Db`            | `listCollections()`         | cursor                   |
| `Collection`    | `insertOne/Many()`          | `Insert*Result`          |
| `Collection`    | `find()`                    | `FindCursor`             |
| `Collection`    | `findOne()`                 | documento/`null`         |
| `Collection`    | `updateOne/Many()`          | `UpdateResult`           |
| `Collection`    | `replaceOne()`              | `UpdateResult`           |
| `Collection`    | `deleteOne/Many()`          | `DeleteResult`           |
| `Collection`    | `findOneAnd*()`             | documento/`null` default |
| `Collection`    | `aggregate()`               | `AggregationCursor`      |
| `Collection`    | `bulkWrite()`               | `BulkWriteResult`        |
| `Collection`    | `countDocuments()`          | number                   |
| `Collection`    | `estimatedDocumentCount()`  | number                   |
| `Collection`    | `createIndex(es)()`         | nome(s)                  |
| `Collection`    | `listIndexes()`             | cursor                   |
| `Collection`    | `createSearchIndex()`       | nome submetido           |
| `Cursor`        | `project/sort/skip/limit()` | configura cursor         |
| `Cursor`        | `next()`                    | próximo documento/`null` |
| `Cursor`        | `toArray()`                 | array de restantes       |
| `Cursor`        | async iterator              | iteração por batches     |
| `Cursor`        | `close()`                   | liberta cursor           |
| `ClientSession` | `withTransaction()`         | convenient transaction   |
| `ClientSession` | `startTransaction()`        | Core API                 |
| `ClientSession` | `commitTransaction()`       | commit                   |
| `ClientSession` | `abortTransaction()`        | abort                    |
| `ClientSession` | `endSession()`              | fecha session            |

### Sintaxe mínima que deve sair de memória

```javascript
const document = await collection.findOne(filter, { projection });

const documents = await collection
    .find(filter)
    .project(projection)
    .sort(sort)
    .limit(limit)
    .toArray();

const updateResult = await collection.updateOne(
    filter,
    { $set: changes },
    { upsert: false },
);

const pipelineResults = await collection
    .aggregate(pipeline, { maxTimeMS: 5_000 })
    .toArray();
```

---

## Exemplos

### Exemplo 1 — escolher a API pelo resultado necessário

```javascript
/**
 * @file Atualiza estado e devolve o documento final de forma atómica.
 */
import { MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);
const orderId = new ObjectId(process.env.ORDER_ID);

try {
    const orders = client.db("shop").collection("orders");
    const order = await orders.findOneAndUpdate(
        { _id: orderId, status: "pending" },
        {
            $set: {
                status: "paid",
                paidAt: new Date(),
            },
        },
        {
            returnDocument: "after",
            projection: {
                _id: 1,
                status: 1,
                paidAt: 1,
            },
        },
    );

    console.log(order);
} finally {
    await client.close();
}
```

Resultado: documento final ou `null`; `updateOne()` não serviria para devolver o documento.

### Exemplo 2 — query, índice e evidence

```javascript
/**
 * @file Cria um índice e mede o plano do access pattern correspondente.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const posts = client.db("blog").collection("posts");
    await posts.createIndex(
        { status: 1, createdAt: -1, _id: -1 },
        { name: "status_createdAt_id" },
    );

    const plan = await posts
        .find(
            { status: "published" },
            { projection: { title: 1, slug: 1, createdAt: 1 } },
        )
        .sort({ createdAt: -1, _id: -1 })
        .limit(25)
        .explain("executionStats");

    console.log({
        returned: plan.executionStats.nReturned,
        keysExamined: plan.executionStats.totalKeysExamined,
        documentsExamined: plan.executionStats.totalDocsExamined,
    });
} finally {
    await client.close();
}
```

Resultado: métricas para decidir se o índice reduz trabalho como esperado.

### Exemplo 3 — aggregation com filtro cedo

```javascript
/**
 * @file Resume vendas online por localização.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const sales = client.db("sample_supplies").collection("sales");
    const pipeline = [
        { $match: { purchaseMethod: "Online" } },
        { $unwind: "$items" },
        {
            $group: {
                _id: "$storeLocation",
                units: { $sum: "$items.quantity" },
                revenue: {
                    $sum: {
                        $multiply: ["$items.price", "$items.quantity"],
                    },
                },
            },
        },
        { $sort: { revenue: -1, _id: 1 } },
    ];

    for await (const row of sales.aggregate(pipeline)) {
        console.log(row);
    }
} finally {
    await client.close();
}
```

Resultado: uma linha agregada por localização, processada por cursor.

---

## Explicação linha a linha

### Exemplo 1

1. O filtro inclui id e estado esperado.
2. `findOneAndUpdate` faz read+write single-document atómico.
3. `$set` preserva outros fields.
4. `returnDocument: "after"` escolhe o estado novo.
5. Projection reduz dados.
6. O retorno é documento ou `null`, não `UpdateResult`.

### Exemplo 2

1. Equality `status` precede os fields de sort.
2. `_id` torna a ordem total.
3. A query repete o access pattern do índice.
4. Explain mede uma execução.
5. Ratios keys/docs/returned mostram trabalho.
6. Existência do índice não é prova suficiente.

### Exemplo 3

1. `$match` reduz o input antes de unwind.
2. `$unwind` muda a unidade de venda para linha.
3. `$group` soma unidades e receita.
4. `$sort` ordena grupos.
5. `aggregate` devolve cursor.
6. Async iteration evita array intermédio.

---

## Casos Reais

| Caso                  | Modelo/API                   | Índice/garantia                             |
| --------------------- | ---------------------------- | ------------------------------------------- |
| registo de utilizador | `insertOne`                  | unique email normalizado                    |
| catálogo              | `find` + projection + keyset | filtro/sort compound                        |
| carrinho              | embedded items + `$[id]`     | atomicidade single-document                 |
| checkout              | transação curta              | unique/idempotency + majority               |
| SaaS de grande escala | sharding por access pattern | shard key tenant-scoped + índices locais     |
| relatório             | aggregation cursor           | match inicial/index                         |
| pesquisa              | `$search compound`           | static search mapping                       |
| sessões               | TTL                          | autorização não depende da remoção imediata |
| outbox                | transaction                  | unique event id/consumer idempotente        |

---

## Performance

### Checklist de análise

1. O filtro reduz cardinalidade cedo?
2. O índice suporta equality, sort e range na ordem certa?
3. `totalDocsExamined`/`totalKeysExamined` estão proporcionais a `nReturned`?
4. Quantos shards recebem a operação e porquê?
5. Cada shard usa um índice eficiente localmente?
6. A projection reduz payload ou cobre a query?
7. Existe sort blocking?
8. `skip` é profundo?
9. `$unwind` multiplica quantas vezes?
10. `$group/$facet/$sort` podem fazer spill?
11. O cursor é iterado ou materializado?
12. Quantos round trips e processos/pools existem?
13. O write atualiza quantos índices?
14. A transação cruza shards ou permanece longa?
15. Search mapping indexa apenas o necessário?

Complexidades são orientações:

- scan: O(n);
- B-tree lookup: aproximadamente O(log n + k);
- sort em memória: aproximadamente O(n log n);
- hash/group: aproximadamente O(n) esperado, com memória por grupos;
- `toArray()`: O(k) memória no cliente.

O plano, os dados e o ambiente determinam a latência real.

---

## Armadilhas comuns

### 55 erros que devo evitar

1. Criar `MongoClient` por request.
2. Guardar a URI/credentials no Git.
3. Comparar ObjectId com string.
4. Guardar dinheiro como Number sem política.
5. Esperar array de `find()`.
6. Usar `toArray()` sem limite.
7. Esquecer `await`.
8. Usar check-then-insert para unicidade.
9. Confundir `matchedCount` e `modifiedCount`.
10. Usar replacement para patch.
11. Fazer `updateMany({})`/`deleteMany({})` sem intenção.
12. Usar `$push` quando se quer set.
13. Omitir `$elemMatch` para condições do mesmo elemento.
14. Misturar inclusion/exclusion em projection.
15. Paginar com sort não determinístico.
16. Criar índices para todos os fields.
17. Ignorar ordem/prefixos de compound index.
18. Concluir performance só por `IXSCAN`.
19. Contar documentos depois de unwind como se fossem pais.
20. Usar `$first` sem sort.
21. Esperar objeto de `$lookup`.
22. Omitir session numa operação de transação.
23. Usar `Promise.all` numa transação.
24. Fazer side effect externo dentro de `withTransaction`.
25. Confundir search index e B-tree index.
26. Confundir a default database da URI com `authSource`.
27. Tentar autenticar a aplicação com um Atlas user.
28. Pensar que Network Access concede autorização sobre dados.
29. Chamar `connect()` ou criar um client em cada função CRUD.
30. Fechar o client partilhado dentro de um repository.
31. Tratar `serverSelectionTimeoutMS` como timeout total da query.
32. Confundir projection com schema persistido.
33. Confundir `batchSize()` com `limit()`.
34. Usar `estimatedDocumentCount()` para uma contagem filtrada.
35. Esperar que `updateOne()` devolva o documento atualizado.
36. Usar `result.value` numa compound operation sem metadata no driver atual.
37. Assumir que `insertMany()` é all-or-nothing.
38. Interpretar ordered batch como rollback de operações anteriores.
39. Tratar `bulkWrite()` como uma transação global.
40. Ver unique index apenas como otimização e ignorar a constraint.
41. Esperar remoção TTL exatamente no instante de expiração.
42. Confundir compound index com multikey index.
43. Ignorar a restrição de vários fields array num compound multikey index.
44. Assumir que `allowDiskUse` remove o limite de `$facet`.
45. Esperar que `$group` preserve fields não acumulados.
46. Esquecer que `$lookup` devolve um array.
47. Esperar um array diretamente de `aggregate()` no driver.
48. Assumir que a callback de `withTransaction()` executa uma única vez.
49. Esperar consistência imediata de um search index após write.
50. Usar regex não ancorada como substituto geral de MongoDB Search.
51. Shardear para esconder uma query sem índice ou um modelo inadequado.
52. Escolher shard key apenas por ter alta cardinalidade.
53. Omitir a shard key das queries críticas e ignorar scatter-gather.
54. Confundir primary shard com o primary de um replica set.
55. Assumir que balancer, migrations e resharding não consomem recursos.

---

## O que costuma aparecer no exame

### Checklist final antes do exame

#### Ambiente e ligação

- [ ] Sei interpretar `mongodb://` e `mongodb+srv://`.
- [ ] Sei explicar percent-encoding, `authSource` e default database.
- [ ] Distingo Atlas user de database user.
- [ ] Sei por que se reutiliza `MongoClient`.
- [ ] Distingo selection, pool, connect, socket e server execution timeouts.
- [ ] Sei o que Stable API resolve e o que não resolve.

#### Modelo e CRUD

- [ ] Decido embedding versus references por access patterns.
- [ ] Conheço BSON, `ObjectId`, Date e `Decimal128`.
- [ ] Sei o limite BSON de documento.
- [ ] Distingo `find`/`findOne` e os retornos.
- [ ] Sei AND implícito, `$or`, ranges e dot notation.
- [ ] Distingo `$in`, `$all` e `$elemMatch`.
- [ ] Sei `null` versus missing.
- [ ] Distingo update parcial, replacement e compound operation.
- [ ] Interpreto todos os result counters.
- [ ] Sei upsert e `$setOnInsert`.
- [ ] Sei positional array operators.
- [ ] Distingo ordered/unordered e partial failure.

#### Resultados e índices

- [ ] Construo projection válida.
- [ ] Defino sort determinístico.
- [ ] Distingo skip e keyset pagination.
- [ ] Distingo contagem exata e estimada.
- [ ] Sei single, compound e multikey.
- [ ] Sei prefixos, direções e ESR.
- [ ] Explico covered query.
- [ ] Leio `COLLSCAN`, `IXSCAN`, `FETCH` e métricas.
- [ ] Conheço unique, partial e TTL.

#### Sharding

- [ ] Distingo replicação, vertical scaling e sharding.
- [ ] Explico shards, `mongos`, config servers e balancer.
- [ ] Avalio cardinalidade, frequência e monotonicidade de uma shard key.
- [ ] Comparo ranged, hashed e zoned sharding.
- [ ] Prevejo targeted queries e scatter-gather.
- [ ] Separo routing global de uso de índice local.
- [ ] Distingo primary shard de replica set primary.
- [ ] Reconheço o custo de migrations e operações cross-shard.

#### Aggregation, transactions e Search

- [ ] Distingo stage, expression e accumulator.
- [ ] Sei o efeito de `$unwind` e `$group`.
- [ ] Sei que `$lookup` devolve array.
- [ ] Identifico blocking stages e limites de memória.
- [ ] Sei que `aggregate()` devolve cursor.
- [ ] Sei quando uma transação é desnecessária.
- [ ] Passo a session a todas as operações.
- [ ] Evito paralelismo e side effects na callback.
- [ ] Distingo database index, text index e search index.
- [ ] Sei mappings, analyzers, `$search`, `$searchMeta` e compound clauses.

### Estratégia na pergunta

1. Sublinhar o verbo: inserir, devolver, modificar, contar, agrupar, pesquisar.
2. Identificar cardinalidade: um ou muitos.
3. Identificar tipo de retorno exigido.
4. Verificar tipo BSON dos operands.
5. Procurar a fronteira de atomicidade.
6. Procurar opção silenciosa: projection, ordered, upsert, returnDocument, session.
7. Eliminar sintaxe antiga/callbacks.
8. Avaliar o índice e custo apenas depois da semântica.

---

## Resumo

### Top 110 conceitos que devo saber

1. MongoDB guarda documentos BSON, não texto JSON.
2. Cada documento tem `_id` único e indexado.
3. `ObjectId` tem 12 bytes e não é um segredo.
4. String hexadecimal e ObjectId são tipos diferentes.
5. BSON Date representa um instante UTC em milissegundos.
6. `Decimal128` é adequado a números decimais exatos.
7. Um documento BSON tem limite de 16 MiB.
8. Flexible schema não significa ausência de invariantes.
9. Schema validation protege writes de qualquer cliente.
10. Modelar começa pelos access patterns.
11. Embedding favorece leitura conjunta.
12. Embedding permite atomicidade single-document.
13. Referencing favorece entidades independentes/partilhadas.
14. Arrays sem limite são um anti-pattern frequente.
15. Todo write single-document é atómico.
16. Atlas user e database user são identidades diferentes.
17. IP access list e database credentials são condições distintas.
18. Replica set replica o mesmo dataset.
19. Sharding distribui o dataset.
20. Replicação não substitui backup.
21. `mongodb+srv://` usa DNS discovery.
22. `mongodb://` lista seeds explicitamente.
23. Credenciais URI exigem percent-encoding.
24. Default database e `authSource` são conceitos diferentes.
25. TLS protege a ligação em trânsito.
26. A URI deve vir de um secret/configuração externa.
27. `MongoClient` gere topology, monitoring e pools.
28. Uma aplicação reutiliza um client por configuração.
29. `db()` e `collection()` criam handles baratos.
30. `connect()` força handshake inicial.
31. `serverSelectionTimeoutMS` não é timeout total da query.
32. `waitQueueTimeoutMS` limita espera no pool.
33. `maxTimeMS` limita execução server-side.
34. Read preference decide routing.
35. Read concern decide garantia da leitura.
36. Write concern decide acknowledgement.
37. Stable API não é a versão do driver.
38. `insertOne()` devolve `InsertOneResult`.
39. `insertMany()` não é uma transação.
40. Ordered batch para no primeiro erro.
41. Unordered batch pode continuar operações independentes.
42. Unique index resolve unicidade concorrente.
43. `find()` devolve `FindCursor`.
44. `findOne()` devolve documento ou `null`.
45. Um filter combina chaves com AND implícito.
46. `$in` pede qualquer valor da lista.
47. `$all` pede todos os valores no array.
48. `$elemMatch` liga condições ao mesmo elemento.
49. Igualdade scalar pode corresponder a elemento de array.
50. Igualdade array considera conteúdo e ordem.
51. `{ field: null }` pode incluir field ausente.
52. Query comparisons respeitam tipos BSON/type bracketing.
53. Dot notation acede a fields embedded.
54. Projection inclusiva e exclusiva não se misturam, salvo `_id`.
55. `_id` aparece por defeito numa projection inclusiva.
56. `sort({ field: 1 })` é ascendente.
57. `sort({ field: -1 })` é descendente.
58. Um sort estável precisa de tiebreaker.
59. Limit restringe total; batch size não.
60. Skip profundo percorre resultados descartados.
61. Keyset pagination usa a última chave.
62. `countDocuments()` aceita filtro exato.
63. `estimatedDocumentCount()` estima o total.
64. `updateOne()` devolve `UpdateResult`, não documento.
65. `$set` preserva fields não mencionados.
66. `replaceOne()` apaga fields omitidos.
67. `matchedCount` e `modifiedCount` medem coisas diferentes.
68. `$inc` faz incremento no servidor.
69. `upsert` faz update-or-insert.
70. `$setOnInsert` só atua no insert do upsert.
71. `$push` pode criar duplicados.
72. `$addToSet` evita uma nova duplicação exata.
73. `$[id]` usa `arrayFilters`.
74. `deleteOne()` elimina no máximo um match.
75. `findOneAndUpdate()` pode devolver o documento after.
76. Compound operations devolvem documento por defeito no driver atual.
77. `bulkWrite()` reduz round trips, sem atomicidade global.
78. Cursor deve ser configurado antes de iterar.
79. `toArray()` usa memória proporcional ao output.
80. Async iteration processa batches progressivamente.
81. Índices B-tree trocam write/storage por read performance.
82. A ordem de compound index define prefixos.
83. ESR significa Equality, Sort, Range.
84. Um índice pode suportar o sort e o inverso completo.
85. Índice de array torna-se multikey.
86. Compound multikey tem restrição de múltiplos fields array.
87. Covered query pode ter `totalDocsExamined: 0`.
88. `IXSCAN` sozinho não prova eficiência.
89. TTL remove documentos de forma assíncrona.
90. Sharding distribui horizontalmente os dados e o workload.
91. Cada shard é normalmente um replica set.
92. `mongos` faz routing; config servers guardam metadata.
93. A shard key determina distribuição e targeting.
94. Cardinalidade alta não compensa frequência ou monotonicidade inadequadas.
95. Ranged preserva localidade; hashed favorece distribuição.
96. Uma targeted query reduz o número de shards participantes.
97. Scatter-gather consulta vários shards e exige merge dos resultados.
98. Shard key decide onde procurar; índice decide como procurar localmente.
99. Balancer e migrations corrigem distribuição com custo operacional.
100. Aggregation pipeline é uma sequência ordered de stages.
101. `$match` cedo pode usar índice e reduzir cardinalidade.
102. `$unwind` emite um documento por elemento.
103. `$group._id` é a chave de grupo.
104. `$sort` e `$group` podem ser blocking.
105. `aggregate()` devolve `AggregationCursor`.
106. Transações exigem session e todas as operações recebem-na.
107. `withTransaction()` pode repetir a callback.
108. Não há operações paralelas numa única transação do driver.
109. `$search` usa search index e deve iniciar a pipeline.
110. `$searchMeta` devolve metadata; `compound.filter` não pontua.

### Regra final

Para qualquer pergunta, identificar **método → input → tipo BSON → cardinalidade → retorno → atomicidade → routing → índice → custo**. Se estes nove pontos estiverem claros, a maior parte das alternativas erradas torna-se visível. Em deployments não sharded, o passo de routing reduz-se à seleção normal da topologia.

Fontes oficiais de revisão: [Learning Path](https://learn.mongodb.com/learning-paths/mongodb-nodejs-developer-path), [guia do Node.js Driver](https://www.mongodb.com/docs/drivers/node/current/), [MongoDB Manual](https://www.mongodb.com/docs/manual/), [MQL reference](https://www.mongodb.com/docs/manual/reference/mql/), [Sharding](https://www.mongodb.com/docs/manual/sharding/), [Aggregation](https://www.mongodb.com/docs/manual/aggregation/), [Transactions](https://www.mongodb.com/docs/manual/core/transactions/) e [MongoDB Search](https://www.mongodb.com/docs/search/).

---

## Preparação final orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — As alternativas erradas são frequentemente corretas noutro contexto. Elimina-as pelo verbo pedido, cardinalidade, tipo BSON, retorno, atomicidade e pré-condições. Só depois compara performance.

> ### ARMADILHA
>
> Não escolhas um método apenas porque contém o verbo certo. `updateOne()` modifica um documento mas não o devolve; `findOneAndUpdate()` pode devolvê-lo. `find()` encontra dados, mas devolve cursor. `insertMany()` insere vários, mas não garante rollback global.

> ### DICA DE MEMORIZAÇÃO
>
> **Método → Input → Tipo → Quantos → Retorno → Atomicidade → Routing → Índice → Custo.** Esta é a checklist de nove passos para qualquer snippet.

> ### COMPARAÇÃO
>
> Usa o sinal da pergunta para escolher primeiro a família de solução.

| Sinal na pergunta               | Família provável            | Confirma antes de escolher          |
| ------------------------------- | --------------------------- | ----------------------------------- |
| “devolver um documento”         | `findOne()`/`findOneAnd*()` | read simples ou read+write?         |
| “alterar campos”                | `updateOne/Many()`          | um ou muitos? precisa do documento? |
| “substituir”                    | `replaceOne()`              | replacement completo intencional?   |
| “agrupar/calcular/juntar”       | `aggregate()`               | cardinalidade e blocking stages?    |
| “garantir unicidade”            | unique index                | filtro e normalização corretos?     |
| “distribuir horizontalmente”    | sharding                    | shard key e queries ficam alinhadas? |
| “all-or-nothing em vários docs” | transação                   | o modelo pode evitar?               |
| “relevância/autocomplete”       | MongoDB Search              | mapping e search index existem?     |

> **Ligação entre capítulos:** este resumo é um índice de recuperação. Quando uma distinção falhar, regressa ao capítulo especializado em vez de decorar apenas a linha da tabela.

### Mapa mental global

```text
MongoDB Associate Developer (Node.js)
|
+-- Modelo
|   +-- BSON / ObjectId / Date / Decimal128
|   +-- embedding vs referencing
|   `-- atomicidade single-document
|
+-- Acesso
|   +-- Atlas / URI / TLS / auth / roles
|   `-- MongoClient / topology / pool / timeouts
|
+-- Operações
|   +-- Insert / Find / Update / Replace / Delete
|   +-- cursor / projection / sort / pagination
|   `-- result objects / compound / bulk
|
+-- Performance
|   +-- single / compound / multikey / unique / TTL
|   +-- explain / cardinalidade / memória / round trips
|   `-- sharding / shard key / targeted / scatter-gather
|
`-- Coordenação e pesquisa
    +-- aggregation stages / expressions / accumulators
    +-- sessions / transactions / retries
    `-- MongoDB Search / mappings / analyzers / score
```

### Fluxograma de decisão global

```text
Qual é o verbo da pergunta?
|
+-- ler ------> um? findOne : muitos? find -> cursor
+-- inserir ---> um? insertOne : muitos? insertMany
+-- alterar ---> patch? update : replacement? replace
|               precisa do documento? findOneAnd*
+-- eliminar --> um? deleteOne : muitos? deleteMany
+-- calcular --> aggregate -> controlar cardinalidade/memória
+-- distribuir horizontalmente --> shard key alinha com o workload?
|                                   |-- não --> rever modelo/queries
|                                   `-- sim --> testar distribuição + targeting
+-- pesquisar relevância --> $search -> confirmar search index
`-- coordenar vários docs --> modelo resolve?
                              |-- sim --> write single-document
                              `-- não --> transação curta

Depois: tipo BSON -> retorno -> atomicidade -> routing -> índice -> custo
```

### Mini desafio final

Para cada linha, identifica tipo de retorno, fronteira de atomicidade, índice relevante e principal armadilha, sem executar:

```javascript
const a = users.find({ roles: "admin" });
const b = await users.updateOne({ _id }, { $set: { active: true } });
const c = await users.findOneAndUpdate(
    { _id },
    { $inc: { loginCount: 1 } },
    { returnDocument: "after" },
);
```

---

## O que rever na véspera do exame

### Bloco 1 — retornos e cardinalidade

- `find()` → cursor; `findOne()` → documento/`null`.
- inserts/updates/deletes → result objects.
- compound operations → documento/`null` por defeito.
- `aggregate()` → `AggregationCursor`.
- `limit` não é `batchSize`.

### Bloco 2 — distinções que eliminam alternativas

- String hexadecimal não é `ObjectId`.
- `$set` não é replacement.
- `$push` não é `$addToSet`.
- `$in` não é `$all`.
- projection não é schema nem cobertura automática.
- compound não é multikey.
- B-tree index não é search index.

### Bloco 3 — performance e segurança

- Um `MongoClient` partilhado por configuração.
- Filtrar cedo e limitar payload/cardinalidade.
- ESR como heurística; `explain` como evidência.
- Nunca inferir eficiência apenas de `IXSCAN`.
- Shard key decide routing; o índice local decide o plano dentro do shard.
- Targeted query reduz fan-out; scatter-gather consulta vários shards.
- Não aceitar filtros ou pipelines arbitrários do utilizador.
- Credenciais fora do código e roles mínimas.

### Bloco 4 — aggregation e transactions

- `$unwind` multiplica; `$group` resume; `$lookup` devolve array.
- Sort antes de accumulators dependentes de ordem.
- `$facet` não faz spill para disco.
- Mesma session em todas as operações da transação.
- Sem `Promise.all()` e sem side effects externos na callback.

### Plano de 30 minutos

| Minutos | Revisão                              |
| ------: | ------------------------------------ |
|     0–5 | tipos BSON e retornos dos métodos    |
|    5–10 | CRUD, contadores e arrays            |
|   10–15 | projection, sort, paginação e counts |
|   15–20 | indexes, sharding, targeting e explain |
|   20–25 | aggregation, cursor e memória         |
|   25–30 | transactions, Search e 55 armadilhas  |

---

## Resumo Rápido

- Começar pelo verbo, cardinalidade e retorno.
- Confirmar tipo BSON e fronteira de atomicidade.
- Distinguir patch, replacement e compound operation.
- Tratar cursor, índice e memória como partes da resposta.
- Separar shard routing do plano de execução local.
- Modelar antes de transacionar; medir antes de otimizar.
- Eliminar alternativas pelo detalhe técnico, não por familiaridade.

---

## Checklist

- [ ] Sei de memória os retornos dos métodos principais.
- [ ] Distingo todas as comparações da tabela inicial.
- [ ] Reconheço os query e update operators essenciais.
- [ ] Prevejo cardinalidade de uma pipeline.
- [ ] Desenho e avalio compound/multikey indexes.
- [ ] Avalio shard keys e prevejo targeted versus scatter-gather.
- [ ] Interpreto contadores e `executionStats`.
- [ ] Decido quando não usar uma transação.
- [ ] Distingo database, text e search indexes.
- [ ] Consigo explicar os 55 erros sem consultar a solução.
- [ ] Aplico os nove passos a código desconhecido.

---

## Perguntas para confirmar conhecimentos

1. Que nove passos deves aplicar a qualquer pergunta com código?
2. Que métodos devolvem cursor e quais devolvem documento/`null`?
3. Que result objects e contadores precisas de reconhecer?
4. Como distingues update parcial, replacement e compound operation?
5. Que diferenças existem entre `$push`, `$addToSet`, `$in` e `$all`?
6. Como distinguem projection, schema e covered query?
7. Como escolhes a ordem de um compound index e verificas a decisão?
8. Que restrições próprias têm multikey indexes?
9. Como avalias uma shard key e distingues targeting de execução indexada local?
10. Como a ordem de stages altera cardinalidade e memória?
11. Quando deves iterar um cursor em vez de usar `toArray()`?
12. Que condições justificam uma transação multi-documento?
13. Que regras tornam uma callback de transação segura para retries?
14. Como diferem B-tree, text index e MongoDB Search?
15. Que cinco armadilhas tens maior probabilidade de cometer sob pressão?
16. Como explicarias por que as alternativas erradas de uma pergunta parecem plausíveis?
