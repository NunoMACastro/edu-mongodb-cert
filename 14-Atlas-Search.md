# Atlas Search (MongoDB Search)

## Objetivos do capítulo

- Distinguir MongoDB Search de índices B-tree e `$text` clássico.
- Criar search indexes com mappings dinâmicos ou estáticos.
- Compreender analyzers, tokenização, scoring e relevância.
- Usar `$search`, `$searchMeta`, `text`, `compound` e `autocomplete`.
- Executar Search pipelines através do Node.js Driver.

---

## Conceitos Fundamentais

### O problema é diferente de procurar um valor exato

Uma query estruturada procura normalmente valores conhecidos:

```javascript
movies.find({ year: 1999, rated: "PG-13" });
```

Uma pesquisa textual coloca perguntas diferentes:

- “encontra filmes sobre exploração espacial”, mesmo que a frase não seja idêntica;
- “aceita uma pequena gralha em *interstellar*”;
- “mostra primeiro os resultados mais relevantes”;
- “sugere títulos enquanto o utilizador escreve `star wa`”;
- “pesquisa palavras segundo regras linguísticas”.

Um índice B-tree ordena valores completos e é excelente para igualdade, ranges e sort. Não transforma prosa em termos pesquisáveis nem calcula, por si só, relevância textual. MongoDB Search — tradicionalmente referido como Atlas Search — resolve esse segundo problema através de **search indexes** e integra os resultados na aggregation pipeline com `$search` e `$searchMeta`.

### Do texto ao índice invertido

Considere dois documentos:

```javascript
const movies = [
    { _id: 1, title: "Exploring Space" },
    { _id: 2, title: "Ocean Exploration" },
];
```

Durante a indexação, um **analyzer** transforma cada string em tokens. Numa explicação simplificada, o resultado poderia ser:

```text
documento 1 -> ["exploring", "space"]
documento 2 -> ["ocean", "exploration"]
```

Um **índice invertido** organiza a relação no sentido termo → documentos, por exemplo:

```text
"space"       -> documento 1
"ocean"       -> documento 2
"exploration" -> documento 2
```

Dependendo do analyzer, `exploring` e `exploration` podem ou não ser reduzidos a uma forma relacionada. O índice pode guardar também frequências, posições e outras metadata usadas para correspondência, highlights e scoring.

Quando chega a query `"space exploration"`, o texto da query também é analisado. O motor procura os tokens no índice, encontra candidatos, calcula scores e devolve primeiro os documentos considerados mais relevantes. É por isso que a escolha de analyzer faz parte da semântica dos dados e não é apenas uma opção de performance.

### Três famílias

| Mecanismo      | Índice                           | Query               | Adequado a                                  |
| -------------- | -------------------------------- | ------------------- | ------------------------------------------- |
| B-tree MongoDB | `createIndex({ field: 1 })`      | equality/range/sort | filtros estruturados                        |
| text clássico  | `createIndex({ field: "text" })` | `$text`             | full-text básico                            |
| MongoDB Search | search index                     | `$search`           | relevância, analyzers, autocomplete, facets |

As três famílias coexistem e podem existir sobre a mesma collection:

- `createIndex({ year: 1 })` pode apoiar filtros e sort por ano;
- `createIndex({ title: "text" })` apoia a pesquisa `$text` clássica, com capacidades mais limitadas;
- `createSearchIndex(...)` cria um índice de Search separado, consultado por `$search`.

Criar um search index não faz `movies.find({ year: 1999 })` passar a usá-lo. Do mesmo modo, um B-tree em `title` não satisfaz um operador `autocomplete`.

Não usar regex não ancorada como substituto de um motor de pesquisa. Regex carece de análise linguística, ranking relevante e escala adequada para muitos casos de search-as-you-type.

### Search index

Um search index define **que fields** entram no índice e **como** cada um é analisado. É uma estrutura derivada e separada dos índices de database. A sincronização acompanha alterações da collection, mas existe uma janela de consistência: uma escrita confirmada na fonte de verdade pode ainda não aparecer imediatamente nos resultados de pesquisa.

Convém distinguir três momentos:

1. **mapping** — declara o tipo de cada field no search index;
2. **index-time analysis** — transforma os valores armazenados em tokens/representações pesquisáveis;
3. **query-time analysis** — transforma o input da pesquisa antes de procurar correspondências.

### Dynamic versus static mapping

- **Dynamic:** indexa automaticamente campos suportados. Excelente para protótipo, mas aumenta índice e pode expor campos a pesquisas inesperadas.
- **Static:** `dynamic: false` e fields explícitos. Melhor controlo de storage, performance, segurança e comportamento.

Um field pode ter mais de uma representação com `multi`/mappings apropriados, por exemplo string para pesquisa textual e autocomplete para search-as-you-type.

### Analyzer

Um analyzer transforma texto em tokens durante indexação. Pode envolver character filters, tokenizer e token filters. `searchAnalyzer` processa a query; normalmente é compatível com o analyzer do índice.

Um analyzer linguístico pode:

- normalizar maiúsculas/minúsculas;
- remover stop words;
- aplicar stemming;
- tratar pontuação e acentos.

Analyzer errado produz falsos positivos/negativos. Pesquisa de códigos, emails e nomes próprios tem necessidades diferentes de prosa em português.

### Scoring

MongoDB Search atribui score de relevância e devolve, por defeito, maior para menor. A similarity default documentada para text é BM25 no contexto atual. Score pode ser alterado por boost, constant e function, conforme o operador.

`compound.filter` restringe resultados sem contribuir para score; condições estruturadas devem frequentemente ficar em `filter`, em vez de `must` ou de um `$match` posterior.

> **Importante para o exame:** um B-tree index criado com `createIndex()` não serve `$search`, e um search index não acelera `find()`. `$search` e `$searchMeta` têm de ser o primeiro stage da pipeline onde aparecem; sem `index` explícito, é procurado o índice chamado `default`.

---

## Funcionamento Interno

O serviço Search usa Apache Lucene. O processo de pesquisa mantém search indexes derivados da collection. Na query:

1. `$search` identifica índice e operador.
2. O query text é analisado em tokens.
3. O índice invertido localiza documentos candidatos.
4. Operadores calculam/restringem correspondências.
5. Scoring ordena candidatos.
6. MongoDB continua os stages seguintes da aggregation pipeline.

Na arquitetura Atlas clássica, componentes Search sincronizam alterações via change stream/oplog. Search é, portanto, um índice secundário derivado, não a fonte de verdade.

### $search e $searchMeta

- `$search` devolve documentos.
- `$searchMeta` devolve metadata, como counts/facets, sem documentos normais.

`$search`/`$searchMeta` devem ser o primeiro stage da pipeline em que aparecem. Escolher index por `index`; se omitido, o nome default esperado é `default`.

### compound

| Clause    | Significado                                         | Score                       |
| --------- | --------------------------------------------------- | --------------------------- |
| `must`    | todas têm de corresponder                           | contribui                   |
| `mustNot` | nenhuma pode corresponder                           | não contribui positivamente |
| `should`  | preferências; minimumShouldMatch controla exigência | contribui                   |
| `filter`  | todas têm de corresponder                           | não contribui               |

Usar `filter` para status, tenant, permissões e ranges que não representam relevância. Segurança não deve depender só de projection; o filtro de autorização tem de restringir candidatos.

### autocomplete

`autocomplete` exige que o field seja indexado como tipo autocomplete. Tokenization `edgeGram`, `rightEdgeGram` ou `nGram` altera matches e tamanho. Fuzzy search aumenta recall e custo. Exact matches podem precisar de field indexado também como string e de boosting adequado.

---

## Sintaxe

### Static search index

No Node.js Driver, a definição completa usada para criar um search index tem esta forma conceptual:

```javascript
const indexName = await collection.createSearchIndex({
    name,
    definition,
});
```

`name` identifica o search index que será referido em `$search.index`. `definition` contém a configuração entendida pelo serviço Search. O retorno é uma `Promise<string>` com o nome; a criação/sincronização do índice é um processo distinto de receber imediatamente resultados pesquisáveis.

```javascript
const searchIndex = {
    name: "movies_search",
    definition: {
        mappings: {
            dynamic: false,
            fields: {
                title: [
                    {
                        type: "string",
                        analyzer: "lucene.standard",
                    },
                    {
                        type: "autocomplete",
                        tokenization: "edgeGram",
                        minGrams: 2,
                        maxGrams: 15,
                    },
                ],
                fullplot: {
                    type: "string",
                    analyzer: "lucene.english",
                },
                genres: {
                    type: "string",
                    analyzer: "lucene.keyword",
                },
                year: {
                    type: "number",
                },
            },
        },
    },
};
```

Leitura por níveis:

- `searchIndex.name` é o nome operacional `movies_search`;
- `definition.mappings` descreve os fields indexados;
- `dynamic: false` impede que fields não declarados sejam automaticamente mapeados;
- `fields` associa cada path à respetiva configuração;
- `title` recebe um array porque o mesmo field terá duas representações de pesquisa;
- `{ type: "string" }` prepara full-text search;
- `{ type: "autocomplete" }` prepara o operador autocomplete;
- `tokenization: "edgeGram"` cria tokens a partir do início das palavras;
- `minGrams` e `maxGrams` limitam o comprimento desses tokens;
- `fullplot` usa análise de língua inglesa;
- `genres` usa `lucene.keyword`, preservando o valor como unidade adequada a labels exatas;
- `year: { type: "number" }` permite operadores estruturados de Search como `range`.

Um field não declarado não fica pesquisável neste mapping estático. O tipo do mapping tem de ser compatível com o operador usado mais tarde: configurar `title` apenas como `string` não prepara automaticamente `autocomplete`.

### Text

`$search` é um stage de aggregation. Dentro dele, `text` é um **operador de Search**, não um stage adicional:

```javascript
const textSearchStage = {
    $search: {
        index: "movies_search",
        text: {
            query: "space exploration",
            path: ["title", "fullplot"],
            fuzzy: {
                maxEdits: 1,
            },
        },
    },
};
```

Anatomia:

| Propriedade | Significado                                                                                  |
| ----------- | -------------------------------------------------------------------------------------------- |
| `index`     | nome do search index; se omitido, é procurado o índice `default`                             |
| `text`      | operador que executa correspondência textual                                                 |
| `query`     | string ou lista de strings a pesquisar, analisadas no query time                             |
| `path`      | field ou fields do mapping onde procurar                                                     |
| `fuzzy`     | permite aproximação por edit distance                                                        |
| `maxEdits`  | máximo de edições admitidas dentro das regras do operador                                    |

`fuzzy` pode aumentar recall, mas também candidatos e custo. Não deve ser ativado indiscriminadamente para compensar um analyzer ou UX mal escolhidos.

### Compound

```javascript
const compoundSearchStage = {
    $search: {
        index: "movies_search",
        compound: {
            must: [
                {
                    text: {
                        query: "robot",
                        path: ["title", "fullplot"],
                    },
                },
            ],
            filter: [
                {
                    range: {
                        path: "year",
                        gte: 2000,
                        lt: 2020,
                    },
                },
            ],
            should: [
                {
                    text: {
                        query: "Sci-Fi",
                        path: "genres",
                        score: { boost: { value: 2 } },
                    },
                },
            ],
        },
    },
};
```

O operador `compound` combina subqueries com significado explícito:

- `must` exige correspondência e contribui para o score;
- `filter` exige correspondência mas não aumenta relevância — o range de anos restringe o universo;
- `should` favorece resultados correspondentes; neste caso, o género `Sci-Fi` recebe `boost` de 2;
- `mustNot`, quando usado, exclui correspondências;
- as clauses são arrays porque podem conter vários operadores independentes.

O path `year` precisa de estar mapeado como `number`. O path `genres` usa o search index, não um B-tree de MongoDB com o mesmo nome. Para isolamento multi-tenant ou autorização, a condição deve restringir os candidatos em `filter`; remover fields apenas numa projection posterior não impede que documentos indevidos tenham participado na pesquisa.

### Metadata

```javascript
const scoreProjectionStage = {
    $project: {
        _id: 0,
        title: 1,
        score: { $meta: "searchScore" },
        highlights: { $meta: "searchHighlights" },
    },
};
```

Este é um stage `$project` normal colocado depois de `$search`:

- `_id: 0` exclui o identificador;
- `title: 1` inclui um field do documento;
- `{ $meta: "searchScore" }` lê metadata produzida por Search e guarda-a no novo field `score`;
- `{ $meta: "searchHighlights" }` lê os fragments de highlight.

`searchHighlights` exige que o `$search` correspondente configure a opção `highlight`; sem essa configuração não se deve projetar highlights. `$meta` aqui não executa nova pesquisa: expõe metadata da pesquisa já realizada.

---

## Exemplos

### Exemplo 1 — criar um search index com o driver

```javascript
/**
 * @file Cria um search index estático para a collection movies.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const indexName = await movies.createSearchIndex({
        name: "movies_search",
        definition: {
            mappings: {
                dynamic: false,
                fields: {
                    title: [
                        { type: "string" },
                        {
                            type: "autocomplete",
                            tokenization: "edgeGram",
                            minGrams: 2,
                            maxGrams: 15,
                        },
                    ],
                    fullplot: {
                        type: "string",
                        analyzer: "lucene.english",
                    },
                    genres: {
                        type: "string",
                        analyzer: "lucene.keyword",
                    },
                    year: { type: "number" },
                },
            },
        },
    });

    console.log(indexName);
} finally {
    await client.close();
}
```

Resultado: nome do índice submetido. A criação é assíncrona no serviço; aguardar que esteja queryable antes de pesquisar.

### Exemplo 2 — text search com score

```javascript
/**
 * @file Pesquisa título e descrição e devolve relevância.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const pipeline = [
        {
            $search: {
                index: "movies_search",
                text: {
                    query: "journey through space",
                    path: ["title", "fullplot"],
                    fuzzy: { maxEdits: 1, prefixLength: 2 },
                },
            },
        },
        { $limit: 10 },
        {
            $project: {
                _id: 0,
                title: 1,
                year: 1,
                score: { $meta: "searchScore" },
            },
        },
    ];

    const results = await movies
        .aggregate(pipeline, {
            maxTimeMS: 5_000,
            comment: "movies-text-search",
        })
        .toArray();

    console.table(results);
} finally {
    await client.close();
}
```

Resultado: até dez resultados, ordenados por relevância Search.

### Exemplo 3 — compound com filtro não pontuado

```javascript
/**
 * @file Combina relevância textual, ano e género preferencial.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const pipeline = [
        {
            $search: {
                index: "movies_search",
                compound: {
                    must: [
                        {
                            text: {
                                query: "artificial intelligence",
                                path: ["title", "fullplot"],
                            },
                        },
                    ],
                    filter: [
                        {
                            range: {
                                path: "year",
                                gte: 1990,
                                lte: 2025,
                            },
                        },
                    ],
                    should: [
                        {
                            text: {
                                query: "Sci-Fi",
                                path: "genres",
                                score: { boost: { value: 3 } },
                            },
                        },
                    ],
                },
            },
        },
        { $limit: 20 },
        {
            $project: {
                _id: 0,
                title: 1,
                year: 1,
                genres: 1,
                score: { $meta: "searchScore" },
            },
        },
    ];

    const results = await movies.aggregate(pipeline).toArray();
    console.log(results);
} finally {
    await client.close();
}
```

Resultado: must e range são obrigatórios; Sci-Fi aumenta score sem ser obrigatório.

### Exemplo 4 — autocomplete validado

```javascript
/**
 * @file Implementa sugestões de título com input limitado.
 */
import { MongoClient } from "mongodb";

const query = (process.env.QUERY ?? "").trim();

if (query.length < 2 || query.length > 80) {
    throw new RangeError("QUERY deve ter entre 2 e 80 caracteres.");
}

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const pipeline = [
        {
            $search: {
                index: "movies_search",
                autocomplete: {
                    query,
                    path: "title",
                    tokenOrder: "sequential",
                    fuzzy: {
                        maxEdits: 1,
                        prefixLength: 1,
                        maxExpansions: 50,
                    },
                },
            },
        },
        { $limit: 8 },
        {
            $project: {
                _id: 0,
                title: 1,
                year: 1,
                score: { $meta: "searchScore" },
            },
        },
    ];

    console.log(await movies.aggregate(pipeline).toArray());
} finally {
    await client.close();
}
```

Resultado: até oito sugestões para um field indexado como autocomplete.

### Exemplo 5 — count com `$searchMeta`

```javascript
/**
 * @file Obtém metadata de contagem sem devolver os filmes.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const metadata = await movies
        .aggregate([
            {
                $searchMeta: {
                    index: "movies_search",
                    text: {
                        query: "mongodb",
                        path: ["title", "fullplot"],
                    },
                    count: {
                        type: "total",
                    },
                },
            },
        ])
        .next();

    console.log(metadata);
} finally {
    await client.close();
}
```

Resultado: documento de metadata com count, não uma lista de filmes.

---

## Explicação linha a linha

### Exemplo 1

1. `createSearchIndex` atua sobre a collection.
2. `dynamic: false` impede indexação automática de outros fields.
3. `title` tem mapping string e autocomplete.
4. `fullplot` usa analyzer inglês; `genres` keyword.
5. `year` é number para range.
6. O método submete a criação; build/sync continua.

### Exemplo 2

1. `$search` é o primeiro stage.
2. `text.path` pesquisa dois fields.
3. Fuzzy admite uma edição após prefixo.
4. `$limit` reduz documentos materializados.
5. `$meta: "searchScore"` expõe a pontuação.

### Exemplo 3

1. `must` exige correspondência textual e contribui para score.
2. `filter` restringe ano sem pontuar.
3. `should` favorece Sci-Fi.
4. Boost multiplica a influência dessa clause.
5. O resultado continua ordenado por relevância.

### Exemplo 4

1. O input é normalizado e limitado antes da query.
2. `autocomplete` exige mapping do mesmo tipo.
3. `tokenOrder: "sequential"` respeita a ordem dos tokens.
4. Fuzzy tem limites explícitos para controlar expansões.
5. Limit protege latência e payload.

### Exemplo 5

1. `$searchMeta` é o primeiro e único stage necessário.
2. Text define o universo.
3. Count total pede contagem exata segundo o serviço.
4. `next()` devolve o documento de metadata.

---

## Casos Reais

- **E-commerce:** text + filter por disponibilidade/preço + facets.
- **Streaming:** título, descrição, cast e boosting por popularidade.
- **Blog:** analyzer por idioma e synonyms controlados.
- **Autocomplete:** collection de sugestões ou títulos, com limites de input.
- **Pesquisa multi-tenant:** tenant em `compound.filter`, nunca apenas pós-processado.
- **Analytics de search:** counts/facets com `$searchMeta`.

---

## Performance

Static mappings reduzem tamanho e trabalho comparado com dynamic indiscriminado. Cada analyzer/multi mapping aumenta storage. Autocomplete com n-grams curtos cria muitos tokens; escolher `minGrams` e `maxGrams` pelo comportamento real.

Colocar filtros estruturados dentro de `compound.filter` tende a ser mais eficiente do que executar `$match` depois de `$search`. Limitar cedo. Para pages profundas, usar `searchAfter`/`searchBefore` quando a funcionalidade e ordenação o suportam, em vez de skip elevado.

Fuzzy, wildcard e regex ampliam o espaço de candidatos. Limitar tamanho de query, edits e expansions. `returnStoredSource`/`storedSource` pode evitar recuperar documentos completos em casos adequados, com custo de duplicar fields no search index.

Search nodes, partitions e concurrency são decisões operacionais avançadas. Medir latência, index size, query analytics e relevância; performance sem qualidade de resultados não resolve o produto.

---

## Armadilhas comuns

- **Criar B-tree index e esperar que `$search` o use.**
- **Criar search index e esperar que `find()` o use.**
- **Colocar `$search` depois de outro stage.**
- **Omitir nome e não ter índice `default`.**
- **Pesquisar field com mapping/tipo incompatível:** pode devolver vazio/erro.
- **Usar `autocomplete` sem mapping autocomplete.**
- **Usar `must` para filtros que não devem pontuar:** usar `filter`.
- **Esperar consistência imediata após write.**
- **Dynamic mapping de todos os fields em produção sem necessidade.**
- **Fuzzy sem limites:** custo e baixa precisão.
- **Regex como motor full-text.**
- **Assumir que score é comparável entre queries diferentes como métrica absoluta.**

---

## O que costuma aparecer no exame

- Search index versus database index.
- `$search` versus `$searchMeta`.
- Primeiro stage da pipeline.
- Dynamic versus static mappings.
- Analyzer e tokenização.
- `text` e `autocomplete`.
- `compound.must`, `mustNot`, `should` e `filter`.
- `$meta: "searchScore"`.
- Relevância/scoring.
- Necessidade de mapping correto.
- Diferença entre Search e `$text` clássico.

---

## Resumo

MongoDB Search usa search indexes separados e stages de aggregation. Mappings definem fields/tipos; analyzers definem tokens. `$search` devolve documentos com score; `$searchMeta` devolve metadata. `compound.filter` restringe sem pontuar, enquanto must/should moldam relevância. Autocomplete requer mapping próprio. Static mappings, limites, filtros internos e observação de sincronização mantêm segurança, custo e qualidade.

Fontes oficiais: [MongoDB Search](https://www.mongodb.com/docs/search/), [queries e indexes](https://www.mongodb.com/docs/search/about/searching/), [query reference](https://www.mongodb.com/docs/search/query/query-ref/), [analyzers](https://www.mongodb.com/docs/search/index/analyzers/overview/), [autocomplete](https://www.mongodb.com/docs/search/query/operators-collectors/autocomplete/) e [Search no Node.js Driver](https://www.mongodb.com/docs/drivers/node/current/atlas-search/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Alta** — `$search` e `$searchMeta` têm de ser o primeiro stage da pipeline onde aparecem e usam search index, não B-tree index. Se `index` for omitido, o nome procurado é `default`.

> ### ARMADILHA
>
> Criar `createIndex({ title: 1 })` não prepara `title` para `$search` ou `autocomplete`. Inversamente, um search index não acelera `find({ title: ... })`. Cada motor exige índice, operadores e mappings próprios.

> ### DICA DE MEMORIZAÇÃO
>
> **B-tree filtra e ordena; Search analisa e pontua. Filter restringe; must/should pontuam.**

> ### COMPARAÇÃO
>
> Três mecanismos de pesquisa coexistem, mas não partilham o mesmo índice.

| Mecanismo      | Índice                      | Query             | Força principal                        |
| -------------- | --------------------------- | ----------------- | -------------------------------------- |
| B-tree         | `createIndex({ field: 1 })` | `find()`/`$match` | equality, range, sort                  |
| text clássico  | text index                  | `$text`           | full-text básico                       |
| MongoDB Search | search index                | `$search`         | analyzers, score, autocomplete, facets |
| `$searchMeta`  | search index                | metadata          | count/facets sem documentos normais    |

> **Ligação entre capítulos:** aggregation e primeira posição de stage estão no capítulo 11; execução pelo driver no 12; B-tree/compound indexes no 09.

### Mapa mental de Search

```text
documentos MongoDB
      |
      v
search index -> mappings -> field types -> analyzers -> tokens
      |
      v
$search / $searchMeta
      |
      +-> compound.filter  restringe sem score
      +-> must / should    correspondência + score
      `-> project          score / highlights / fields
```

### Mini desafio

Uma pipeline começa com `$match` e só depois usa `$search`. Explica porque é inválida, onde colocarias um filtro estruturado de tenant e como evitarias que esse filtro alterasse a relevância.

---

## Resumo Rápido

- Search index é separado de B-tree/text index.
- Mappings escolhem fields/tipos; analyzers criam tokens.
- `$search` devolve documentos e score.
- `$searchMeta` devolve metadata.
- `compound.filter` restringe sem contribuir para score.
- `autocomplete` exige mapping autocomplete; sincronização não é imediata.

---

## Checklist

- [ ] Distingo B-tree, text index e search index.
- [ ] Sei onde `$search`/`$searchMeta` aparecem na pipeline.
- [ ] Distingo dynamic de static mappings.
- [ ] Explico analyzer, token e score.
- [ ] Uso `must`, `should`, `mustNot` e `filter` corretamente.
- [ ] Sei o requisito de `autocomplete`.
- [ ] Sei projetar `searchScore`/highlights quando configurados.
- [ ] Considero latência de sincronização do índice.

---

## Perguntas para confirmar conhecimentos

1. Que diferença existe entre database index e search index?
2. Porque `createIndex({ title: 1 })` não serve `$search`?
3. Que posição devem ocupar `$search` e `$searchMeta`?
4. Que índice é usado se o nome for omitido?
5. Como diferem dynamic e static mappings?
6. Que papel têm analyzer e tokenizer?
7. Como diferem `must`, `should`, `mustNot` e `filter` no score?
8. Porque `autocomplete` exige um field type específico?
9. Que diferença existe entre `$search` e `$searchMeta`?
10. Porque uma escrita recente pode ainda não aparecer numa pesquisa?
