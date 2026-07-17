# CRUD: sort, limit, projection, count e paginação

## Objetivos do capítulo

- Ordenar resultados de forma determinística.
- Limitar, saltar e paginar sem resultados instáveis.
- Construir projections inclusivas e exclusivas válidas.
- Distinguir `countDocuments()` de `estimatedDocumentCount()`.
- Compreender o efeito de índices sobre sort e covered queries.

---

## Conceitos Fundamentais

### Uma query não termina no filtro

No capítulo anterior, o filtro decidiu **quais** documentos correspondem. Uma leitura real precisa frequentemente de responder a mais quatro perguntas:

1. **Em que ordem** devem aparecer? — `sort`.
2. **Quantos** podem ser devolvidos? — `limit`.
3. **Que campos** pode o cliente receber? — `projection`.
4. **Quantos matches existem** sem devolver os documentos? — operações de count.

Considere:

```javascript
const cursor = movies.find(
    { genres: "Drama" },
    {
        sort: { year: -1, _id: 1 },
        limit: 10,
        projection: { title: 1, year: 1 },
    },
);
```

Esta leitura deve ser interpretada por camadas:

```text
collection movies
    -> filtrar documentos cujo array genres contém "Drama"
    -> ordenar por year descendente e _id ascendente
    -> reter no máximo 10
    -> devolver apenas title, year e o _id incluído por defeito
    -> produzir um cursor que ainda terá de ser consumido
```

Os objetos não são intercambiáveis. `{ genres: "Drama" }` é um **filter**; `{ year: -1, _id: 1 }` é uma **sort specification**; `{ title: 1, year: 1 }` é uma **projection**. O mesmo número `1` tem significados dependentes do contexto: ascendente no sort, inclusão na projection.

### Sort

`{ field: 1 }` ordena ascendentemente e `{ field: -1 }` descendentemente. Num compound sort, a ordem das chaves define a precedência:

```javascript
const sort = { year: -1, title: 1 };
```

Ano é o critério principal; título resolve empates. Sem um tiebreaker único, paginação pode ser instável quando muitos documentos têm os mesmos valores.

MongoDB usa uma ordem de comparação BSON entre tipos. Misturar tipos no mesmo campo torna ordenações surpreendentes e dificulta índices. Collation altera comparação linguística de strings; a query só usa eficientemente um índice para collation compatível.

### Limit e skip

`limit(n)` restringe quantos documentos chegam ao cliente. `skip(n)` descarta n resultados depois de ordenar/filtrar. Skip profundo continua a exigir que o servidor percorra os resultados anteriores, por isso keyset/range pagination escala melhor.

### Projection

Uma projection controla campos devolvidos:

- **Inclusiva:** campos com `1`; os outros são omitidos.
- **Exclusiva:** campos com `0`; os outros são incluídos.
- `_id` é exceção: aparece por defeito numa projection inclusiva e pode ser excluído.
- Não misturar inclusão e exclusão, salvo a exclusão de `_id`.

Projection reduz rede e desserialização. Só evita ler documentos quando a query é coberta pelo índice.

Operadores de projection relevantes:

- `$slice`: subset de array.
- `$elemMatch`: primeiro elemento que satisfaz condições.
- `$` positional projection: primeiro elemento associado à condição da query.
- `$meta`: metadata, como text/search score no contexto aplicável.

### Count

| Método                     | Filtro | Exatidão/uso                                  |
| -------------------------- | ------ | --------------------------------------------- |
| `countDocuments(filter)`   | sim    | conta matches segundo query                   |
| `estimatedDocumentCount()` | não    | usa metadata para estimar total da collection |

`cursor.count()` e `collection.count()` pertencem a APIs antigas/deprecated; usar os métodos explícitos.

> **Importante para o exame:** uma projection mistura inclusão e exclusão apenas para `_id`. `limit()` limita o total devolvido; `batchSize()` apenas controla aproximadamente quantos documentos viajam em cada batch do cursor.

---

## Funcionamento Interno

### Sort suportado por índice

Um índice B-tree já mantém chaves ordenadas. Se filtro e sort respeitam os prefixos/direção compatíveis, o servidor percorre o índice em ordem e evita um blocking sort.

Exemplo:

```javascript
const index = { status: 1, createdAt: -1, _id: -1 };
```

Suporta filtro de igualdade em `status` e sort `{ createdAt: -1, _id: -1 }`. Também pode percorrer todo o padrão no sentido inverso. Misturar direções exige compatibilidade com o compound index.

Sem índice, `SORT` pode precisar de guardar resultados candidatos. `sort + limit` pode manter apenas os top n, reduzindo memória.

### Projection e covered query

Uma query é coberta quando filtro e campos devolvidos estão no índice e não é necessário `FETCH`. Como `_id` é incluído por defeito, uma projection deve excluí-lo se `_id` não pertence ao índice. Multikey indexes têm restrições de cobertura para campos array.

### Paginação

Offset pagination:

```javascript
find(filter)
    .sort(sort)
    .skip(page * size)
    .limit(size);
```

Keyset pagination:

```javascript
const pageFilter = {
    ...baseFilter,
    $or: [
        { createdAt: { $lt: lastCreatedAt } },
        { createdAt: lastCreatedAt, _id: { $lt: lastId } },
    ],
};
```

A segunda mantém uma fronteira lexicográfica para sort descendente `{ createdAt: -1, _id: -1 }`.

---

## Sintaxe

### Formas em options

A assinatura conceptual é:

```javascript
collection.find(filter, options);
```

`find()` devolve imediatamente um `FindCursor`. O `await` só aparece quando o cursor é consumido, por exemplo com `toArray()`, `next()` ou `for await...of`.

```javascript
const cursor = collection.find(filter, {
    projection: { _id: 0, title: 1, year: 1 },
    sort: { year: -1, title: 1 },
    limit: 25,
    skip: 0,
});
```

Cada parte tem uma responsabilidade diferente:

| Parte        | Forma esperada                  | O que controla                                                                  |
| ------------ | ------------------------------- | ------------------------------------------------------------------------------- |
| `filter`     | documento MQL                   | quais documentos são candidatos                                                 |
| `projection` | documento de inclusão/exclusão  | que fields são devolvidos                                                       |
| `sort`       | documento ordenado de fields    | ordem do resultado; `1` ascendente, `-1` descendente                            |
| `limit`      | integer não negativo            | máximo de documentos devolvidos                                                 |
| `skip`       | integer não negativo            | quantos resultados já ordenados são ignorados antes de devolver                 |

No exemplo, `_id: 0` é a exceção permitida dentro de uma projection inclusiva. `limit: 25` não quer dizer “batch de 25”; limita a cardinalidade total do cursor. `skip: 0` não altera o resultado e existe apenas para tornar explícita a primeira página por offset.

### Forma encadeada

```javascript
const cursor = collection
    .find(filter)
    .project({ _id: 0, title: 1, year: 1 })
    .sort({ year: -1, title: 1 })
    .skip(0)
    .limit(25);
```

Esta forma descreve a mesma intenção. Cada método configura e devolve o próprio cursor, permitindo encadear a chamada seguinte:

- `project(specification)` configura a projection;
- `sort(specification)` configura a ordem;
- `skip(number)` configura o offset;
- `limit(number)` configura o máximo total.

O encadeamento escrito não deve ser interpretado como quatro round trips independentes. O driver reúne a configuração para executar a operação. O servidor pode ainda otimizar a combinação, desde que preserve a semântica.

Configurar o cursor antes da iteração. Não executar operações assíncronas concorrentes sobre o mesmo cursor nem misturar paradigmas como `toArray()` e `for await`.

### Projection de arrays

```javascript
const exampleProjections = [
    { cast: { $slice: 3 } },
    {
        grades: {
            $elemMatch: {
                grade: "A",
                score: { $lte: 5 },
            },
        },
    },
];
```

No primeiro objeto, `$slice: 3` devolve os três primeiros elementos de `cast`; não filtra os documentos da collection. No segundo, `$elemMatch` é um operador de projection: devolve o primeiro elemento de `grades` que satisfaz ambas as condições. Isto é diferente de usar `$elemMatch` no filtro, onde o objetivo é decidir se o documento corresponde.

### Counts

```javascript
const exact = await collection.countDocuments(
    { status: "active" },
    { maxTimeMS: 2_000 },
);

const estimate = await collection.estimatedDocumentCount();
```

Anatomia das duas chamadas:

- `countDocuments(filter, options)` aceita um filtro e executa uma contagem exata dos matches segundo a semântica da query; `maxTimeMS` limita o tempo server-side;
- `estimatedDocumentCount(options)` não aceita filtro e usa metadata da collection para obter rapidamente uma estimativa do total;
- ambos devolvem `Promise<number>`, não cursores nem documentos;
- `countDocuments({})` e `estimatedDocumentCount()` podem produzir valores iguais num instante, mas não executam o mesmo trabalho nem oferecem a mesma garantia.

---

## Exemplos

### Exemplo 1 — top N com ordem determinística

```javascript
/**
 * @file Obtém os filmes melhor classificados com desempate estável.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const topMovies = await movies
        .find({
            year: { $gte: 2000 },
            "imdb.votes": { $gte: 10_000 },
        })
        .project({
            _id: 0,
            title: 1,
            year: 1,
            "imdb.rating": 1,
            "imdb.votes": 1,
        })
        .sort({
            "imdb.rating": -1,
            "imdb.votes": -1,
            title: 1,
        })
        .limit(20)
        .toArray();

    console.table(topMovies);
} finally {
    await client.close();
}
```

Resultado: 20 ou menos documentos, com desempates explícitos.

### Exemplo 2 — keyset pagination

```javascript
/**
 * @file Lê a página seguinte sem skip profundo.
 */
import { MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);
const pageSize = 25;
const lastCreatedAt = process.env.LAST_CREATED_AT
    ? new Date(process.env.LAST_CREATED_AT)
    : null;
const lastId = process.env.LAST_ID ? new ObjectId(process.env.LAST_ID) : null;

try {
    const posts = client.db("blog").collection("posts");
    const boundary =
        lastCreatedAt && lastId
            ? {
                  $or: [
                      { createdAt: { $lt: lastCreatedAt } },
                      { createdAt: lastCreatedAt, _id: { $lt: lastId } },
                  ],
              }
            : {};

    const page = await posts
        .find({ status: "published", ...boundary })
        .project({ title: 1, slug: 1, createdAt: 1 })
        .sort({ createdAt: -1, _id: -1 })
        .limit(pageSize)
        .toArray();

    console.log(page);
} finally {
    await client.close();
}
```

Resultado: primeira página sem cursor externo ou página seguinte após o par data/id.

### Exemplo 3 — projection de array

```javascript
/**
 * @file Devolve apenas os três primeiros atores de cada filme.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const documents = await movies
        .find(
            { genres: "Comedy" },
            {
                projection: {
                    _id: 0,
                    title: 1,
                    year: 1,
                    cast: { $slice: 3 },
                },
            },
        )
        .sort({ year: -1, title: 1 })
        .limit(10)
        .toArray();

    console.dir(documents, { depth: null });
} finally {
    await client.close();
}
```

Resultado: até dez comédias e, em cada uma, no máximo três elementos de `cast`.

### Exemplo 4 — count exato versus estimativa

```javascript
/**
 * @file Compara uma contagem filtrada com uma estimativa global.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const [dramaCount, collectionEstimate] = await Promise.all([
        movies.countDocuments({ genres: "Drama" }, { maxTimeMS: 5_000 }),
        movies.estimatedDocumentCount(),
    ]);

    console.log({ dramaCount, collectionEstimate });
} finally {
    await client.close();
}
```

Resultado: contagem de dramas e estimativa do total, com semânticas diferentes.

---

## Explicação linha a linha

### Exemplo 1

1. Filtra ano e mínimo de votos antes de ordenar.
2. Projection inclui só campos de apresentação.
3. Sort usa rating, votos e título como desempates.
4. Limit pequeno torna `toArray()` controlado.
5. O resultado não inclui `_id`.

### Exemplo 2

1. Converte token externo em Date e ObjectId.
2. Sem token, `boundary` é filtro vazio.
3. Com token, `$or` implementa a comparação lexicográfica.
4. O spread combina status e fronteira.
5. Sort tem exatamente os dois campos do token.
6. O último documento devolvido fornece o próximo token.

### Exemplo 3

1. A equality sobre `genres` corresponde a um elemento do array.
2. `$slice: 3` limita apenas o array devolvido, não os documentos.
3. Sort e limit controlam o conjunto.
4. A collection não é modificada.

### Exemplo 4

1. `countDocuments` executa um filtro exato.
2. `estimatedDocumentCount` não aceita filtro e usa metadata.
3. `Promise.all` é seguro porque são operações independentes, ao contrário de operações paralelas numa transação.
4. Os valores não devem ser comparados como se medissem o mesmo universo.

---

## Casos Reais

- **Leaderboard:** compound sort com tiebreaker único.
- **Feed:** keyset pagination por data e id.
- **Catálogo:** projection reduz payload de cards.
- **Dashboard:** `countDocuments` para filtro; estimativa para uma indicação global tolerante.
- **Detalhe de filme:** `$slice` evita devolver arrays enormes quando só se mostra uma amostra.

---

## Performance

Um índice que suporta filtro e sort evita blocking sort. Regra prática: campos de igualdade primeiro, depois sort, depois range — ESR — mas analisar `explain()` e access patterns reais. Um range antes dos campos de sort pode impedir a ordenação integral pelo índice.

Skip tem custo aproximadamente proporcional ao offset percorrido. Keyset pagination navega a partir de uma chave e mantém custo por página mais estável. Não permite saltar diretamente para “página 500” sem informação adicional.

Projection reduz bytes. Covered query pode reduzir `totalDocsExamined` a zero. `countDocuments` pode ser caro em filtros amplos; `estimatedDocumentCount` é rápido mas não responde a filtros nem garante a mesma exatidão operacional.

---

## Armadilhas comuns

- **Sort sem tiebreaker:** páginas repetem/omitem elementos.
- **Sort direction incompatível com compound index.**
- **Skip profundo:** o servidor percorre resultados descartados.
- **Misturar inclusion e exclusion:** projection inválida, salvo `_id`.
- **Esquecer que `_id` é incluído por defeito.**
- **Pensar que projection muda o documento:** altera apenas a resposta.
- **Confundir query `$elemMatch` com projection `$elemMatch`.**
- **Usar `estimatedDocumentCount` para filtro:** não suporta essa semântica.
- **Materializar sem limit.**
- **Configurar cursor depois de o iterar.**

---

## O que costuma aparecer no exame

- `1` versus `-1` no sort.
- Ordem dos campos num compound sort.
- Projection inclusiva/exclusiva e exceção de `_id`.
- `$slice` e projection `$elemMatch`.
- `limit` versus `skip`.
- Cursor chaining antes da iteração.
- `countDocuments` versus `estimatedDocumentCount`.
- Sort coberto por índice.
- Covered query e `FETCH`.
- Paginação estável.

---

## Resumo

Sort deve ser determinístico e idealmente suportado por índice. Limit controla cardinalidade; skip profundo não escala. Projection pode ser inclusiva ou exclusiva e reduz payload, mas só uma covered query evita document fetch. `countDocuments` conta um filtro; `estimatedDocumentCount` estima o total. Para feeds grandes, keyset pagination por uma chave composta é mais estável e eficiente.

Fontes oficiais: [sort no driver](https://www.mongodb.com/docs/drivers/node/current/crud/query/sort/), [limit](https://www.mongodb.com/docs/drivers/node/current/crud/query/limit/), [projection](https://www.mongodb.com/docs/drivers/node/current/crud/query/project/), [count](https://www.mongodb.com/docs/drivers/node/current/crud/query/count/) e [ordem BSON](https://www.mongodb.com/docs/manual/reference/bson-type-comparison-order/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — Uma projection é inclusiva ou exclusiva, com `_id` como exceção. `limit()` restringe o total; `batchSize()` afeta batches. Um sort para paginação precisa de tiebreaker único para ser determinístico.

> ### ARMADILHA
>
> Excluir fields reduz payload, mas não transforma automaticamente a query em covered query. Para evitar `FETCH`, filtro e projection têm de ser resolvidos pelo índice; `_id` incluído por defeito pode quebrar a cobertura.

> ### DICA DE MEMORIZAÇÃO
>
> **Projection escolhe colunas; limit escolhe total; batchSize escolhe viagens.** Para empates no sort, acrescenta `_id`.

> ### COMPARAÇÃO
>
> Operações parecidas controlam dimensões diferentes do resultado.

| Conceito                   | Controla                 | Não controla             |
| -------------------------- | ------------------------ | ------------------------ |
| projection                 | fields devolvidos        | número de documentos     |
| `limit()`                  | total máximo             | tamanho de cada batch    |
| `batchSize()`              | transferência por batch  | total final              |
| `skip()`                   | offset descartado        | estabilidade sem sort    |
| `countDocuments(filter)`   | contagem exata do filtro | estimativa barata global |
| `estimatedDocumentCount()` | estimativa da collection | filtro arbitrário        |

> **Ligação entre capítulos:** o cursor nasce no capítulo 05; lifecycle no 08; suporte de sort e covered queries no capítulo 09.

### Fluxo de paginação

```text
Dataset pequeno e página ocasional?
  |-- sim --> sort determinístico + skip + limit pode bastar
  `-- não --> páginas profundas ou feed contínuo?
                |-- sim --> keyset pela última chave composta
                `-- não --> medir ambos com índice adequado

Sort: { createdAt: -1, _id: -1 }
Cursor seguinte: valores de createdAt + _id da última linha
```

### Mini desafio

Uma query projeta `{ title: 1, year: 1 }` sobre um índice `{ year: 1, title: 1 }`. Decide que detalhe adicional pode impedir cobertura e como o corrigir sem mudar os dados persistidos.

---

## Resumo Rápido

- Sort determinístico usa tiebreaker único.
- Projection inclusiva/exclusiva não se mistura, salvo `_id`.
- Projection reduz payload; cobertura depende do índice.
- `limit()` e `batchSize()` não são equivalentes.
- Skip profundo custa trabalho descartado.
- `countDocuments()` filtra; `estimatedDocumentCount()` estima total.

---

## Checklist

- [ ] Construo sort determinístico.
- [ ] Sei quando um índice suporta sort.
- [ ] Escrevo projections válidas.
- [ ] Trato `_id` conscientemente.
- [ ] Distingo projection de covered query.
- [ ] Distingo limit de batch size.
- [ ] Escolho skip ou keyset conscientemente.
- [ ] Escolho o método de contagem correto.

---

## Perguntas para confirmar conhecimentos

1. Porque um sort apenas por data pode ser não determinístico?
2. Que papel pode ter `_id` como tiebreaker?
3. Que regra impede misturar inclusão e exclusão numa projection?
4. Porque excluir `_id` pode ser necessário numa covered query?
5. Qual é a diferença entre reduzir payload e evitar `FETCH`?
6. Que diferença existe entre `limit()` e `batchSize()`?
7. Porque skip profundo degrada com o número da página?
8. Que valores precisa um cursor de keyset composto para obter a página seguinte?
9. Quando usar `countDocuments()`?
10. Porque `estimatedDocumentCount()` não substitui uma contagem filtrada?
