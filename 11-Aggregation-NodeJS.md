# Aggregation com o MongoDB Node.js Driver

## Objetivos do capítulo

- Representar pipelines de forma segura e legível em JavaScript.
- Usar `AggregationCursor`, async iteration e `toArray()` conscientemente.
- Passar options como `allowDiskUse`, `maxTimeMS`, `batchSize` e `comment`.
- Trabalhar corretamente com tipos BSON e parâmetros da aplicação.
- Testar e analisar pipelines no código Node.js.

---

## Conceitos Fundamentais

No driver, uma pipeline é um array de plain objects e `collection.aggregate(pipeline, options)` devolve um `AggregationCursor`. A chamada não devolve imediatamente o array final.

```javascript
const cursor = collection.aggregate(pipeline, options);
```

Depois:

```javascript
const results = await cursor.toArray();
```

ou:

```javascript
for await (const document of cursor) {
    // processar um documento de cada vez
}
```

### aggregate versus find no driver

Usar `find()` para uma leitura direta com filtro, projection, sort e limit. Usar `aggregate()` quando é necessária transformação, grouping, join, facet, search ou cálculo. A pipeline não é uma query string: é uma estrutura de dados que deve ser construída a partir de parâmetros validados.

### Construção segura

Não aceitar stages completos enviados por um cliente HTTP sem uma política administrativa explícita. Um stage arbitrário pode:

- fazer scans/joins dispendiosos;
- expor campos não autorizados;
- escrever com `$merge`/`$out`;
- contornar regras de negócio.

Construir uma allowlist de filtros e escolher os stages no servidor. Valores do utilizador podem ocupar posições controladas, depois de validação de tipo/range.

### Reutilização e pureza

Definir pipeline builders puros facilita testes. A função recebe parâmetros já validados e devolve um novo array. Não mutar um array partilhado entre requests: um `push` condicional pode deixar stages de um pedido no seguinte.

### BSON

Os valores têm de manter o tipo do armazenamento:

- converter ids com `new ObjectId()`;
- usar `Decimal128` para decimals;
- usar `Date` para BSON Date;
- não passar datas ISO como strings se o campo é Date.

> **Importante para o exame:** `collection.aggregate()` devolve um `AggregationCursor`, não um array nem uma Promise de array. `toArray()` materializa todos os resultados restantes; `for await...of` permite processá-los progressivamente por batches.

---

## Funcionamento Interno

`aggregate()` envia o comando e obtém um cursor server-side. O primeiro batch pode vir na resposta inicial; `getMore` obtém os restantes. `batchSize` não limita o total nem é equivalente a `$limit`.

O driver serializa a pipeline para BSON. O servidor valida os stages e aplica otimizações. JavaScript não executa os `$match`/`$group` localmente; são documentos que descrevem operações server-side.

### Lifecycle do cursor

Um cursor pode terminar porque:

- foi esgotado;
- foi fechado explicitamente;
- ocorreu erro/timeout;
- o client fechou;
- o servidor o matou por regras de timeout/recursos.

Em scripts, fechar no `finally` se a iteração pode ser interrompida. Não usar o mesmo cursor em operações assíncronas concorrentes.

### Explains

Explain mostra o plano dos stages que usam o query engine e estatísticas quando pedido. Estrutura varia conforme pipeline/versão; testes não devem depender de paths internos excessivamente frágeis. Para análise humana, procurar `COLLSCAN`, `IXSCAN`, `FETCH`, sorts, keys/docs examined e cardinalidade entre stages.

---

## Sintaxe

### API principal

```javascript
const cursor = collection.aggregate(pipeline, {
    allowDiskUse: true,
    batchSize: 100,
    maxTimeMS: 10_000,
    comment: "sales-summary",
    hint: { saleDate: 1 },
});
```

- `pipeline`: array ordered de stages.
- `allowDiskUse`: permite spill para stages elegíveis.
- `batchSize`: documentos por batch, aproximadamente; não limita total.
- `maxTimeMS`: limite server-side da operação.
- `comment`: metadata de diagnóstico.
- `hint`: sugere/força índice elegível para o início; validar o ciclo de vida.

### Builder puro

```javascript
/**
 * Cria uma pipeline de receita por localização.
 */
function buildRevenuePipeline({ start, end, minimumRevenue }) {
    return [
        {
            $match: {
                saleDate: { $gte: start, $lt: end },
            },
        },
        { $unwind: "$items" },
        {
            $group: {
                _id: "$storeLocation",
                revenue: {
                    $sum: {
                        $multiply: ["$items.price", "$items.quantity"],
                    },
                },
            },
        },
        { $match: { revenue: { $gte: minimumRevenue } } },
        { $sort: { revenue: -1, _id: 1 } },
    ];
}
```

O primeiro `$match` pode usar o índice da collection. O segundo filtra um campo calculado e não pode ser movido antes do `$group`.

---

## Exemplos

### Exemplo 1 — relatório completo e limitado

```javascript
/**
 * @file Calcula as cinco lojas com maior receita num intervalo.
 */
import { Decimal128, MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const sales = client.db("sample_supplies").collection("sales");
    const start = new Date("2015-01-01T00:00:00.000Z");
    const end = new Date("2016-01-01T00:00:00.000Z");
    const pipeline = [
        { $match: { saleDate: { $gte: start, $lt: end } } },
        { $unwind: "$items" },
        {
            $group: {
                _id: "$storeLocation",
                revenue: {
                    $sum: {
                        $multiply: ["$items.price", "$items.quantity"],
                    },
                },
                units: { $sum: "$items.quantity" },
            },
        },
        {
            $match: {
                revenue: { $gte: Decimal128.fromString("10000.00") },
            },
        },
        { $sort: { revenue: -1, _id: 1 } },
        { $limit: 5 },
        {
            $project: {
                _id: 0,
                storeLocation: "$_id",
                revenue: 1,
                units: 1,
            },
        },
    ];

    const report = await sales
        .aggregate(pipeline, {
            maxTimeMS: 10_000,
            comment: "annual-store-revenue",
        })
        .toArray();

    console.dir(report, { depth: null });
} finally {
    await client.close();
}
```

Resultado: no máximo cinco lojas e valores agregados.

### Exemplo 2 — async iteration de output potencialmente grande

```javascript
/**
 * @file Emite contagens por realizador como JSON Lines.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const pipeline = [
        { $match: { directors: { $type: "array" } } },
        { $unwind: "$directors" },
        {
            $group: {
                _id: "$directors",
                movies: { $sum: 1 },
                averageRating: { $avg: "$imdb.rating" },
            },
        },
        { $sort: { movies: -1, _id: 1 } },
    ];

    const cursor = movies.aggregate(pipeline, {
        allowDiskUse: true,
        batchSize: 100,
        maxTimeMS: 30_000,
    });

    try {
        for await (const row of cursor) {
            process.stdout.write(JSON.stringify(row) + "\n");
        }
    } finally {
        await cursor.close();
    }
} finally {
    await client.close();
}
```

Resultado: uma linha por realizador sem materializar todo o resultado no processo.

### Exemplo 3 — pipeline parametrizada com paginação

```javascript
/**
 * @file Pesquisa catálogo por género com parâmetros validados.
 */
import { MongoClient } from "mongodb";

const allowedGenres = new Set([
    "Action",
    "Comedy",
    "Documentary",
    "Drama",
    "Sci-Fi",
]);

const requestedGenre = process.env.GENRE ?? "Drama";
const requestedLimit = Number.parseInt(process.env.LIMIT ?? "20", 10);

if (!allowedGenres.has(requestedGenre)) {
    throw new RangeError("Género não permitido.");
}

if (
    !Number.isInteger(requestedLimit) ||
    requestedLimit < 1 ||
    requestedLimit > 100
) {
    throw new RangeError("LIMIT deve estar entre 1 e 100.");
}

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const pipeline = [
        { $match: { genres: requestedGenre } },
        { $sort: { "imdb.rating": -1, _id: 1 } },
        { $limit: requestedLimit },
        {
            $project: {
                title: 1,
                year: 1,
                rating: "$imdb.rating",
            },
        },
    ];

    const results = await movies
        .aggregate(pipeline, {
            maxTimeMS: 3_000,
            comment: "genre-catalog",
        })
        .toArray();

    console.log(results);
} finally {
    await client.close();
}
```

Resultado: entre 1 e 100 filmes do género permitido; nenhuma chave MQL vem do exterior.

### Exemplo 4 — `$lookup` no driver

```javascript
/**
 * @file Junta comentários recentes aos filmes sem N+1 queries.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const pipeline = [
        { $match: { title: "The Matrix" } },
        { $limit: 1 },
        {
            $lookup: {
                from: "comments",
                let: { movieId: "$_id" },
                pipeline: [
                    {
                        $match: {
                            $expr: { $eq: ["$movie_id", "$$movieId"] },
                        },
                    },
                    { $sort: { date: -1 } },
                    { $limit: 5 },
                    { $project: { _id: 0, name: 1, text: 1, date: 1 } },
                ],
                as: "recentComments",
            },
        },
        {
            $project: {
                _id: 0,
                title: 1,
                year: 1,
                recentComments: 1,
            },
        },
    ];

    const movie = await movies.aggregate(pipeline).next();
    console.dir(movie, { depth: null });
} finally {
    await client.close();
}
```

Resultado: um documento com array `recentComments`, ou `null` se não houver filme.

---

## Explicação linha a linha

### Exemplo 1

1. Dates são valores BSON Date, não strings.
2. `$match` inicial reduz vendas.
3. `$unwind` expande linhas.
4. `$multiply` preserva a semântica Decimal do campo price.
5. `$group` calcula receita/unidades.
6. O `$match` pós-group usa `Decimal128`.
7. Sort, limit e project finalizam o relatório.
8. `toArray()` é seguro porque o output está limitado a cinco.

### Exemplo 2

1. Filtra documentos com array de realizadores.
2. Unwind cria uma ocorrência por realizador/filme.
3. Group calcula contagem e média.
4. `allowDiskUse` admite spill, mas não reduz o custo lógico.
5. Async iteration escreve uma linha de cada vez.
6. `finally` fecha cursor e client.

### Exemplo 3

1. Allowlist impede um género arbitrário.
2. Parse e range limitam cardinalidade.
3. Os stages são escolhidos pela aplicação.
4. Sort tem `_id` como desempate.
5. Projection renomeia rating.
6. `maxTimeMS` limita execução server-side.

### Exemplo 4

1. Match e limit restringem o lado local antes do join.
2. `let` expõe o id atual ao subpipeline.
3. `$expr` compara foreign `movie_id` com a variável.
4. O subpipeline ordena, limita e projeta comentários.
5. `as` é sempre array.
6. `next()` obtém um output ou `null` sem `toArray()`.

---

## Casos Reais

- **Endpoint analítico:** builder puro recebe intervalo e filtros validados.
- **Export:** async iteration produz CSV/JSONL sem memória O(n).
- **Catálogo:** pipeline allowlisted evita operator injection.
- **Detalhe:** `$lookup` elimina N+1 quando o join é adequado.
- **Relatório:** comment e maxTimeMS facilitam controlo operacional.
- **Testes:** pipeline builder é verificado como estrutura e contra dataset fixture.

---

## Performance

`toArray()` usa memória proporcional ao output; reservar para resultados limitados. `batchSize` não reduz resultados e deve equilibrar round trips/memória. O driver e o servidor têm buffers adicionais; não interpretar batch size como limite rígido de RAM.

Construir índice para o `$match`/`$sort` iniciais. Num `$lookup`, indexar o foreign field. Aplicar `$limit` cedo apenas quando não altera indevidamente a população da agregação.

`allowDiskUse` troca falha por I/O em stages elegíveis. Monitorizar spills, documentos examinados e cardinalidade. Um pipeline que retorna poucos documentos pode continuar caro se agrupou milhões antes do limit.

---

## Armadilhas comuns

- **Fazer `await collection.aggregate()` e esperar array:** devolve cursor.
- **Confundir `batchSize` com `$limit`.**
- **`toArray()` em output ilimitado.**
- **Passar datas/ids como strings.**
- **Aceitar stages do request.**
- **Mutar pipeline global entre pedidos.**
- **Executar async operations concorrentes no mesmo cursor.**
- **Fechar client antes de consumir o cursor.**
- **Esperar que Node execute expressions localmente.**
- **Assumir que `allowDiskUse` torna tudo rápido.**
- **N+1 queries quando `$lookup` seria uma transformação única adequada.**

---

## O que costuma aparecer no exame

- `aggregate()` devolve `AggregationCursor`.
- `toArray()` versus async iteration.
- Pipeline como array de stage documents.
- Options do aggregate.
- `batchSize` versus `$limit`.
- Conversão ObjectId/Date/Decimal128.
- `$lookup` com `let`, `$expr` e output array.
- Construção dinâmica segura.
- `maxTimeMS` e `allowDiskUse`.
- Diferença Node driver versus sintaxe `mongosh`.

---

## Resumo

No Node.js Driver, `aggregate()` devolve cursor. Materializar apenas outputs limitados; iterar resultados grandes. Pipelines são documentos BSON executados no servidor e devem ser construídos por código controlado, com parâmetros validados e tipos BSON corretos. Options gerem batches, timeouts, spill e observabilidade. Builders puros melhoram testes e evitam mutações entre requests.

Fontes oficiais: [aggregation no driver](https://www.mongodb.com/docs/drivers/node/current/aggregation/), [AggregationCursor API](https://mongodb.github.io/node-mongodb-native/), [pipeline optimization](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-optimization/) e [sample datasets](https://www.mongodb.com/docs/atlas/sample-data/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Alta** — `collection.aggregate(pipeline, options)` devolve `AggregationCursor`. `toArray()` materializa todos os resultados restantes; `for await...of` processa progressivamente. `batchSize` não é `$limit`.

> ### ARMADILHA
>
> Uma pipeline é uma estrutura BSON enviada ao servidor, não JavaScript executado localmente. Aceitar stages arbitrários de um request permite queries caras, acesso a fields não autorizados ou stages com efeitos. Valores validados podem entrar em posições controladas; a estrutura deve vir do servidor.

> ### DICA DE MEMORIZAÇÃO
>
> **Builder constrói; server executa; cursor transporta; consumer decide memória.**

> ### COMPARAÇÃO
>
> A forma de consumo muda memória e latência, não a semântica server-side.

| Consumo          | Memória no cliente           | Adequado a                      |
| ---------------- | ---------------------------- | ------------------------------- |
| `next()`         | um documento                 | metadata/primeiro resultado     |
| `toArray()`      | todo o restante              | output pequeno e limitado       |
| `for await...of` | progressiva por batches      | muitos resultados               |
| stream           | backpressure de Node streams | integração com pipelines stream |

> **Ligação entre capítulos:** semântica dos stages está no capítulo 10; cursor e client partilhado nos capítulos 04 e 08; tipos BSON no 02.

### Fluxo seguro de construção

```text
HTTP parameters
      |
 validar tipo / range / autorização
      |
      v
builder puro escolhe stages permitidos
      |
      v
collection.aggregate(pipeline, options)
      |
      v
AggregationCursor -> limitar / iterar / fechar
```

### Mini desafio

Um endpoint recebe `req.body.pipeline` e chama diretamente `collection.aggregate(req.body.pipeline)`. Identifica riscos de segurança, performance e previsibilidade e descreve uma alternativa com builder puro.

---

## Resumo Rápido

- `aggregate()` devolve `AggregationCursor`.
- `toArray()` materializa; async iteration processa progressivamente.
- `batchSize` controla batches, não total.
- Pipeline executa no servidor, apesar de ser escrita como objetos JS.
- Builders devem receber parâmetros validados, não stages arbitrários.
- `maxTimeMS`, `comment` e explain ajudam controlo e diagnóstico.

---

## Checklist

- [ ] Reconheço `AggregationCursor`.
- [ ] Escolho `toArray()` ou async iteration pela cardinalidade.
- [ ] Distingo `batchSize` de `$limit`.
- [ ] Preservo tipos `ObjectId`, `Date` e `Decimal128`.
- [ ] Construo pipelines com builders puros.
- [ ] Não aceito stages arbitrários de utilizadores.
- [ ] Configuro timeouts e comentários úteis.
- [ ] Fecho cursor quando interrompo consumo.

---

## Perguntas para confirmar conhecimentos

1. Que tipo devolve `collection.aggregate()`?
2. Quando a pipeline é enviada e executada?
3. Que diferença de memória existe entre `toArray()` e `for await...of`?
4. Porque `batchSize: 100` não limita o resultado a 100 documentos?
5. Que risco existe ao reutilizar e mutar o mesmo array de stages entre requests?
6. Como um builder puro melhora testes e isolamento?
7. Porque uma data ISO string pode não corresponder a um field BSON Date?
8. Para que servem `maxTimeMS` e `comment`?
9. Quando deve um cursor ser fechado explicitamente?
10. Que informação procurarias num explain de uma pipeline lenta?
