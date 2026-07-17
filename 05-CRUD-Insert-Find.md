# CRUD: inserir, encontrar e construir filtros

## Objetivos do capítulo

- Inserir um ou vários documentos e interpretar os resultados.
- Construir filtros MQL com igualdade, comparação, lógica e arrays.
- Distinguir `find()` de `findOne()`.
- Compreender igualdade por tipo, `null` versus missing e dot notation.
- Escolher entre cursor, materialização e iteração.

---

## Conceitos Fundamentais

### Vocabulário de CRUD e queries

**CRUD** agrupa quatro intenções fundamentais: **Create**, **Read**, **Update** e **Delete**. Este capítulo cobre Create através de inserts e Read através de queries.

| Conceito      | Definição                                                                                               |
| ------------- | ------------------------------------------------------------------------------------------------------- |
| Write         | operação que pretende criar, alterar ou eliminar dados                                                  |
| Read          | operação que consulta dados sem os modificar                                                            |
| Insert        | write que acrescenta um novo documento a uma collection                                                 |
| Query         | pedido de leitura composto por filtro e, opcionalmente, projection, sort, limit e outras opções        |
| Filter        | documento MQL que descreve as condições de seleção                                                      |
| Predicate     | condição dentro do filtro, como igualdade ou range                                                      |
| Match         | documento que satisfaz o filtro                                                                         |
| Result object | objeto devolvido por um write com acknowledgement, identificadores ou contadores                       |
| Cursor        | objeto stateful que permite configurar e consumir zero ou mais resultados por batches                  |
| Batch         | grupo de documentos transferido numa resposta inicial ou num `getMore`                                 |
| Materializar  | carregar os resultados restantes numa estrutura como um array                                          |

Um filtro não contém os resultados: contém apenas a descrição das condições. Do mesmo modo, um cursor não é um array; representa uma leitura que pode produzir resultados progressivamente.

### O que é um insert?

Um insert cria um documento novo na collection. O servidor valida regras aplicáveis, verifica constraints como unique indexes e persiste o documento e as respetivas index keys. O retorno normal é um result object, não o documento inserido.

Se `_id` não for fornecido, o driver atribui normalmente um `ObjectId` antes de enviar o documento. Por isso, a aplicação pode consultar `result.insertedId` e o próprio objeto enviado pode já conter `_id`.

### O que é uma query?

Uma query de leitura pede ao servidor documentos que satisfazem um filtro. O filtro pode ser vazio, usar igualdade ou combinar predicates através de operadores. Projection, sort e limit alteram a forma, a ordem ou a quantidade do resultado, mas não substituem o filtro.

No driver, a cardinalidade pretendida determina a API:

```javascript
const insertResult = await collection.insertOne(document);
const cursor = collection.find(filter);
const documentOrNull = await collection.findOne(filter);
```

- `insertResult` contém metadata do write;
- `cursor` ainda tem de ser consumido;
- `documentOrNull` é já um valor final único.

### insertOne versus insertMany

`insertOne(document, options)` insere um documento. Se `_id` estiver ausente, o driver atribui normalmente um `ObjectId` antes do envio. `insertMany(documents, options)` envia vários documentos e, por defeito, é ordered: para no primeiro erro. Com `ordered: false`, o servidor pode continuar operações independentes e a ordem de execução não deve ser assumida.

| Método       | Input     | Resultado normal                                       |
| ------------ | --------- | ------------------------------------------------------ |
| `insertOne`  | documento | `InsertOneResult` com `insertedId`                     |
| `insertMany` | array     | `InsertManyResult` com `insertedCount` e `insertedIds` |

Um acknowledgement não significa que o documento satisfaz regras de negócio que não foram validadas. Unique indexes, schema validation e write concern são mecanismos distintos.

### find versus findOne

| Método                     | Devolve          | Ausência     | Cardinalidade |
| -------------------------- | ---------------- | ------------ | ------------- |
| `find(filter, options)`    | `FindCursor`     | cursor vazio | zero a muitos |
| `findOne(filter, options)` | documento/`null` | `null`       | zero ou um    |

`findOne()` não significa que o filtro identifica univocamente um documento; apenas devolve um match. Se a regra exige unicidade, criar unique index.

### Estrutura de filtros

Um filter document combina condições ao mesmo nível com AND implícito:

```javascript
const filter = {
    year: { $gte: 2000 },
    genres: "Drama",
};
```

Operadores essenciais:

| Família    | Operadores                                                |
| ---------- | --------------------------------------------------------- |
| comparação | `$eq`, `$ne`, `$gt`, `$gte`, `$lt`, `$lte`, `$in`, `$nin` |
| lógica     | `$and`, `$or`, `$nor`, `$not`                             |
| elemento   | `$exists`, `$type`                                        |
| array      | `$all`, `$elemMatch`, `$size`                             |
| avaliação  | `$regex`, `$expr`, `$text` quando existe text index       |

`$in` corresponde a qualquer valor da lista. `$all` exige que um array contenha todos. `$elemMatch` exige que um mesmo elemento satisfaça todas as condições.

### null e campo ausente

`{ field: null }` corresponde normalmente a documentos em que `field` é `null` **ou não existe**. Para exigir `null` explícito:

```javascript
const explicitNullFilter = {
    field: {
        $type: 10,
    },
};
```

Para exigir existência:

```javascript
const existingFieldFilter = {
    field: {
        $exists: true,
    },
};
```

O código numérico 10 é BSON Null; a forma por alias, quando aceite no contexto, é mais legível.

> **Importante para o exame:** `find()` devolve sempre um cursor, mesmo quando nenhum documento corresponde; `findOne()` devolve documento ou `null`. Em arrays de subdocumentos, usa `$elemMatch` quando várias condições têm de ser satisfeitas pelo mesmo elemento.

---

## Funcionamento Interno

### Insert

O driver serializa cada documento para BSON. O servidor valida limite de tamanho, `_id`, validators e constraints de índices. Depois escreve no storage engine e atualiza todos os índices afetados. O write concern determina quando responde com acknowledgement.

Cada documento individual é atómico. `insertMany()` não torna automaticamente o batch numa transação: num batch ordered podem existir inserts confirmados antes de um erro. Para all-or-nothing multi-documento, é necessária transação, se justificada.

### Query

O query planner cria candidate plans com base no filtro, sort, projection e índices. Pode escolher `COLLSCAN` ou `IXSCAN` e cachear informação do plano. Um cursor devolve o primeiro batch e um identificador de cursor server-side quando existem mais resultados. Pedidos `getMore` obtêm batches seguintes.

### Type bracketing

Comparações query como `$gt` aplicam normalmente type bracketing: comparam campos cujo tipo BSON corresponde ao tipo do operando, com equivalência entre tipos numéricos. A string `"2020"` não é o número `2020`.

### Arrays

Igualdade a um scalar sobre um campo array corresponde se algum elemento for igual. Igualdade a um array exige conteúdo e ordem exatos. Dot notation percorre arrays de subdocumentos, mas várias condições sem `$elemMatch` podem ser satisfeitas por elementos diferentes.

---

## Sintaxe

### Insert

```javascript
const oneResult = await collection.insertOne(document, {
    writeConcern: { w: "majority" },
});

const manyResult = await collection.insertMany(documents, {
    ordered: false,
});
```

### Find

```javascript
const cursor = collection.find(filter, {
    projection,
    sort,
    limit,
    maxTimeMS,
    hint,
    collation,
});

const document = await collection.findOne(filter, {
    projection,
    sort,
    maxTimeMS,
});
```

`sort` em `findOne()` escolhe que match devolver quando existem vários; não cria unicidade.

### Filtros frequentes

```javascript
const exampleFilters = [
    { year: 1999 },
    { year: { $gte: 1990, $lt: 2000 } },
    { genres: { $in: ["Drama", "Comedy"] } },
    { $or: [{ rated: "G" }, { "imdb.rating": { $gte: 8.5 } }] },
    { cast: { $all: ["Keanu Reeves", "Laurence Fishburne"] } },
    { "awards.wins": { $exists: true, $gt: 0 } },
    {
        comments: {
            $elemMatch: {
                author: "Ana",
                approved: true,
            },
        },
    },
];
```

---

## Exemplos

### Exemplo 1 — insertOne idempotente por unique key

```javascript
/**
 * @file Insere um utilizador cuja unicidade deve ser garantida por índice.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const users = client.db("certification_lab").collection("users");
    await users.createIndex({ email: 1 }, { unique: true });

    const result = await users.insertOne({
        email: "ana@example.com",
        displayName: "Ana",
        roles: ["reader"],
        createdAt: new Date(),
    });

    console.log(result.insertedId);
} finally {
    await client.close();
}
```

Resultado: um `insertedId`; uma segunda inserção com o mesmo email falha com duplicate key.

### Exemplo 2 — insertMany unordered

```javascript
/**
 * @file Insere eventos independentes e permite continuar após erros isolados.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const events = client.db("certification_lab").collection("events");
    const documents = [
        { eventId: "evt-101", type: "view", occurredAt: new Date() },
        { eventId: "evt-102", type: "click", occurredAt: new Date() },
        { eventId: "evt-103", type: "purchase", occurredAt: new Date() },
    ];

    const result = await events.insertMany(documents, { ordered: false });
    console.log({
        insertedCount: result.insertedCount,
        insertedIds: result.insertedIds,
    });
} finally {
    await client.close();
}
```

Resultado: acknowledgement por documento inserido. Perante erro parcial, `insertMany()` lança; o `catch` deve inspecionar o bulk write error se a aplicação precisar dos sucessos.

### Exemplo 3 — filtros compostos e cursor

```javascript
/**
 * @file Encontra filmes com condições escalares, nested e de array.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const filter = {
        year: { $gte: 2000, $lt: 2010 },
        genres: { $all: ["Action", "Sci-Fi"] },
        "imdb.rating": { $gte: 7.5 },
        $or: [{ rated: "PG-13" }, { rated: "R" }],
    };

    const cursor = movies
        .find(filter)
        .project({ _id: 0, title: 1, year: 1, genres: 1, "imdb.rating": 1 })
        .sort({ "imdb.rating": -1, title: 1 })
        .limit(25);

    for await (const movie of cursor) {
        console.log(movie);
    }
} finally {
    await client.close();
}
```

Resultado: até 25 filmes que satisfazem todas as condições de topo e uma alternativa de rating.

### Exemplo 4 — `$elemMatch` evita combinação entre elementos

```javascript
/**
 * @file Encontra restaurantes com uma mesma avaliação recente e alta.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const restaurants = client
        .db("sample_restaurants")
        .collection("restaurants");

    const result = await restaurants.findOne(
        {
            grades: {
                $elemMatch: {
                    grade: "A",
                    score: { $lte: 5 },
                },
            },
        },
        {
            projection: {
                _id: 0,
                name: 1,
                borough: 1,
                grades: { $elemMatch: { grade: "A", score: { $lte: 5 } } },
            },
        },
    );

    console.dir(result, { depth: null });
} finally {
    await client.close();
}
```

Resultado: documento ou `null`; a projection `$elemMatch` devolve apenas o primeiro elemento correspondente.

---

## Explicação linha a linha

### Exemplo 1

1. Cria um unique index em `email`; é ele que garante unicidade concorrente.
2. O documento omite `_id` e o driver gera-o.
3. `insertOne()` aguarda acknowledgement.
4. `insertedId` identifica o documento, mas não contém o documento completo.

### Exemplo 2

1. Cada evento tem uma chave de domínio.
2. `ordered: false` permite continuar após falhas independentes.
3. O result object existe no caminho de sucesso total.
4. “Unordered” não significa “sem acknowledgement” nem “atómico”.

### Exemplo 3

1. O range de ano é `[2000, 2010)`.
2. `$all` exige ambos os géneros no array.
3. Dot notation acede ao subdocumento `imdb`.
4. `$or` é combinado com AND implícito das outras chaves.
5. Projection, sort e limit configuram o cursor antes da iteração.
6. `for await` processa batches progressivamente.

### Exemplo 4

1. Query `$elemMatch` exige grade A e score ≤ 5 no mesmo elemento.
2. Projection `$elemMatch` é um operador diferente com objetivo de projeção.
3. Só o primeiro elemento projetado é devolvido.
4. `findOne()` devolve um restaurante ou `null`.

---

## Casos Reais

- **Registo:** unique email evita race entre “verificar” e “inserir”.
- **Telemetria:** unordered batch aceita eventos independentes, com reconciliação dos erros.
- **Catálogo:** filtros combinam ranges, tags e atributos nested.
- **Reservas:** filtros com estado esperado suportam decisões concorrentes.
- **Moderação:** `$elemMatch` procura um subdocumento que satisfaz várias condições.

---

## Performance

`insertMany()` reduz round trips relativamente a muitos `insertOne()`, mas batches enormes aumentam latência e custo de retry. Cada índice aumenta o custo de insert. Indexar apenas queries/constraints justificadas.

Um filtro seletivo com índice reduz keys/docs examined. `$ne`, `$nin` e condições pouco seletivas podem usar índice sem grande benefício. Regex ancorada pode usar um índice adequado em alguns casos; regex sem prefixo tende a ser dispendiosa. Collation do índice tem de ser compatível com a query.

Iteração por cursor mantém memória aproximadamente proporcional ao batch/processamento, não ao total. `toArray()` é O(n) em memória no cliente.

---

## Armadilhas comuns

- **Tratar `insertMany` como transação:** pode haver sucesso parcial.
- **Não criar unique index:** uma verificação prévia tem race condition.
- **Confundir `insertedId` com documento inserido.**
- **Esperar array de `find()`:** é cursor.
- **Esperar cursor de `findOne()`:** é documento ou `null`.
- **Usar `{ field: null }` para exigir null explícito:** também pode corresponder a missing.
- **Comparar ObjectId com string.**
- **Usar `$in` quando se pretende todos os elementos:** usar `$all`.
- **Omitir `$elemMatch` em condições do mesmo elemento.**
- **Materializar resultados ilimitados.**
- **Usar regex proveniente do utilizador sem limites:** risco de consumo excessivo.

---

## O que costuma aparecer no exame

- `insertOne` versus `insertMany`.
- Ordered versus unordered e erro parcial.
- `find` versus `findOne`.
- AND implícito e `$or` explícito.
- `$in`, `$all`, `$elemMatch` e `$size`.
- Dot notation.
- `null` versus missing.
- Igualdade de array versus igualdade de elemento.
- Cursor e `toArray()`.
- Unique index como garantia concorrente.

---

## Resumo

`insertOne` e `insertMany` devolvem result objects; batches não são transações. `find` devolve cursor e `findOne` documento/`null`. Filtros combinam operadores com AND implícito, respeitam tipos BSON e têm semântica específica para arrays. `$elemMatch` liga várias condições ao mesmo elemento. Unique indexes garantem unicidade; projeção, limites e iteração progressiva controlam custo.

Fontes oficiais: [insert no driver](https://www.mongodb.com/docs/drivers/node/current/crud/insert/), [find no driver](https://www.mongodb.com/docs/drivers/node/current/crud/query/retrieve/), [query predicates](https://www.mongodb.com/docs/manual/reference/mql/query-predicates/), [arrays](https://www.mongodb.com/docs/manual/reference/mql/query-predicates/arrays/) e [cursor](https://www.mongodb.com/docs/drivers/node/current/crud/query/cursor/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — `find()` devolve sempre `FindCursor`; `findOne()` devolve documento ou `null`. `insertMany()` reduz round trips, mas não cria atomicidade global: um erro pode ocorrer depois de alguns inserts já terem sido confirmados.

> ### ARMADILHA
>
> Em `{ scores: { $gte: 80, $lt: 90 } }`, elementos diferentes do array podem satisfazer cada condição. Só `$elemMatch` exige que o mesmo elemento cumpra ambas. A sintaxe sem `$elemMatch` não é equivalente.

> ### DICA DE MEMORIZAÇÃO
>
> **find = futuro conjunto; findOne = valor agora.** E **elemMatch = mesmo elemento**.

> ### COMPARAÇÃO
>
> Cardinalidade pedida e tipo de retorno determinam o método.

| Método         |  Cardinalidade | Retorno            | Atenção                      |
| -------------- | -------------: | ------------------ | ---------------------------- |
| `insertOne()`  |      um insert | `InsertOneResult`  | `insertedId`                 |
| `insertMany()` | vários inserts | `InsertManyResult` | partial failure possível     |
| `find()`       |  zero a muitos | `FindCursor`       | só executa/consome ao iterar |
| `findOne()`    |     zero ou um | documento/`null`   | não devolve cursor           |

> **Ligação entre capítulos:** projections e limits são aprofundados no capítulo 07; lifecycle de cursor no 08; índices para filtros no 09.

### Mapa mental de leitura

```text
Preciso de um documento?
  |-- sim --> findOne(filter) -> document | null
  `-- não --> find(filter) -> cursor
                          |
                          +-> project / sort / limit
                          `-> next / for await / toArray
```

### Mini desafio

Prevê se as duas queries seguintes são equivalentes quando `scores` é um array e justifica sem as executar:

```javascript
const a = { scores: { $gte: 80, $lt: 90 } };
const b = { scores: { $elemMatch: { $gte: 80, $lt: 90 } } };
```

---

## Resumo Rápido

- `insertOne()` e `insertMany()` devolvem result objects.
- Batch de inserts não é transação.
- `find()` devolve cursor; `findOne()` devolve documento/`null`.
- Filtros respeitam tipo BSON e AND implícito.
- `$elemMatch` liga condições ao mesmo elemento.
- Projection, limit e iteração controlam payload e memória.

---

## Checklist

- [ ] Escolho `insertOne()` ou `insertMany()` pela cardinalidade.
- [ ] Interpreto `acknowledged`, `insertedId` e `insertedCount`.
- [ ] Distingo batch ordered de transação.
- [ ] Distingo `find()` de `findOne()`.
- [ ] Construo filtros com tipos BSON corretos.
- [ ] Sei quando usar `$elemMatch`.
- [ ] Distingo `null` de field ausente.
- [ ] Itero cursor sem materializar resultados ilimitados.

---

## Perguntas para confirmar conhecimentos

1. Que tipo devolve `insertOne()` e onde aparece o novo identificador?
2. Que diferença existe entre `ordered: true` e `ordered: false` em `insertMany()`?
3. Porque `insertMany()` não equivale a uma transação?
4. Que tipo devolve `find()` mesmo sem correspondências?
5. Como verificas a ausência de resultado em `findOne()`?
6. Como funciona o AND implícito num filtro MongoDB?
7. Quando duas condições sobre um array exigem `$elemMatch`?
8. Qual é a diferença entre igualdade a scalar e igualdade ao array completo?
9. Porque `{ field: null }` pode corresponder a mais do que BSON Null?
10. Quando `toArray()` representa um risco de memória?
