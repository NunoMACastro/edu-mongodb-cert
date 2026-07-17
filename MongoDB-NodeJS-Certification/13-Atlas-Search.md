# Atlas Search (MongoDB Search)

## Objetivos do capítulo

- Distinguir MongoDB Search de índices B-tree e `$text` clássico.
- Criar search indexes com mappings dinâmicos ou estáticos.
- Compreender analyzers, tokenização, scoring e relevância.
- Usar `$search`, `$searchMeta`, `text`, `compound` e `autocomplete`.
- Executar Search pipelines através do Node.js Driver.

---

## Conceitos Fundamentais

MongoDB Search — tradicionalmente referido como Atlas Search — oferece pesquisa full-text baseada em índices de pesquisa. Integra-se com aggregation através de `$search` e `$searchMeta`. Não é o mesmo índice B-tree usado por `find()`, nem o text index clássico consultado com `$text`.

### Três famílias

| Mecanismo | Índice | Query | Adequado a |
|---|---|---|---|
| B-tree MongoDB | `createIndex({ field: 1 })` | equality/range/sort | filtros estruturados |
| text clássico | `createIndex({ field: "text" })` | `$text` | full-text básico |
| MongoDB Search | search index | `$search` | relevância, analyzers, autocomplete, facets |

Não usar regex não ancorada como substituto de um motor de pesquisa. Regex carece de análise linguística, ranking relevante e escala adequada para muitos casos de search-as-you-type.

### Search index

Um search index mapeia termos para documentos e guarda metadata, como posições. É separado dos índices de database. A sincronização acompanha alterações da collection, mas existe uma janela de consistência do índice: uma escrita pode não aparecer imediatamente na pesquisa.

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

| Clause | Significado | Score |
|---|---|---|
| `must` | todas têm de corresponder | contribui |
| `mustNot` | nenhuma pode corresponder | não contribui positivamente |
| `should` | preferências; minimumShouldMatch controla exigência | contribui |
| `filter` | todas têm de corresponder | não contribui |

Usar `filter` para status, tenant, permissões e ranges que não representam relevância. Segurança não deve depender só de projection; o filtro de autorização tem de restringir candidatos.

### autocomplete

`autocomplete` exige que o field seja indexado como tipo autocomplete. Tokenization `edgeGram`, `rightEdgeGram` ou `nGram` altera matches e tamanho. Fuzzy search aumenta recall e custo. Exact matches podem precisar de field indexado também como string e de boosting adequado.

---

## Sintaxe

### Static search index

~~~javascript
const searchIndex = {
  name: "movies_search",
  definition: {
    mappings: {
      dynamic: false,
      fields: {
        title: [
          {
            type: "string",
            analyzer: "lucene.standard"
          },
          {
            type: "autocomplete",
            tokenization: "edgeGram",
            minGrams: 2,
            maxGrams: 15
          }
        ],
        fullplot: {
          type: "string",
          analyzer: "lucene.english"
        },
        genres: {
          type: "string",
          analyzer: "lucene.keyword"
        },
        year: {
          type: "number"
        }
      }
    }
  }
};
~~~

### Text

~~~javascript
const textSearchStage = {
  $search: {
    index: "movies_search",
    text: {
      query: "space exploration",
      path: ["title", "fullplot"],
      fuzzy: {
        maxEdits: 1
      }
    }
  }
};
~~~

### Compound

~~~javascript
const compoundSearchStage = {
  $search: {
    index: "movies_search",
    compound: {
      must: [
        {
          text: {
            query: "robot",
            path: ["title", "fullplot"]
          }
        }
      ],
      filter: [
        {
          range: {
            path: "year",
            gte: 2000,
            lt: 2020
          }
        }
      ],
      should: [
        {
          text: {
            query: "Sci-Fi",
            path: "genres",
            score: { boost: { value: 2 } }
          }
        }
      ]
    }
  }
};
~~~

### Metadata

~~~javascript
const scoreProjectionStage = {
  $project: {
    _id: 0,
    title: 1,
    score: { $meta: "searchScore" },
    highlights: { $meta: "searchHighlights" }
  }
};
~~~

`searchHighlights` exige que o `$search` correspondente configure a opção `highlight`; sem essa configuração não se deve projetar highlights.

---

## Exemplos

### Exemplo 1 — criar um search index com o driver

~~~javascript
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
              maxGrams: 15
            }
          ],
          fullplot: {
            type: "string",
            analyzer: "lucene.english"
          },
          genres: {
            type: "string",
            analyzer: "lucene.keyword"
          },
          year: { type: "number" }
        }
      }
    }
  });

  console.log(indexName);
} finally {
  await client.close();
}
~~~

Resultado: nome do índice submetido. A criação é assíncrona no serviço; aguardar que esteja queryable antes de pesquisar.

### Exemplo 2 — text search com score

~~~javascript
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
          fuzzy: { maxEdits: 1, prefixLength: 2 }
        }
      }
    },
    { $limit: 10 },
    {
      $project: {
        _id: 0,
        title: 1,
        year: 1,
        score: { $meta: "searchScore" }
      }
    }
  ];

  const results = await movies.aggregate(pipeline, {
    maxTimeMS: 5_000,
    comment: "movies-text-search"
  }).toArray();

  console.table(results);
} finally {
  await client.close();
}
~~~

Resultado: até dez resultados, ordenados por relevância Search.

### Exemplo 3 — compound com filtro não pontuado

~~~javascript
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
                path: ["title", "fullplot"]
              }
            }
          ],
          filter: [
            {
              range: {
                path: "year",
                gte: 1990,
                lte: 2025
              }
            }
          ],
          should: [
            {
              text: {
                query: "Sci-Fi",
                path: "genres",
                score: { boost: { value: 3 } }
              }
            }
          ]
        }
      }
    },
    { $limit: 20 },
    {
      $project: {
        _id: 0,
        title: 1,
        year: 1,
        genres: 1,
        score: { $meta: "searchScore" }
      }
    }
  ];

  const results = await movies.aggregate(pipeline).toArray();
  console.log(results);
} finally {
  await client.close();
}
~~~

Resultado: must e range são obrigatórios; Sci-Fi aumenta score sem ser obrigatório.

### Exemplo 4 — autocomplete validado

~~~javascript
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
            maxExpansions: 50
          }
        }
      }
    },
    { $limit: 8 },
    {
      $project: {
        _id: 0,
        title: 1,
        year: 1,
        score: { $meta: "searchScore" }
      }
    }
  ];

  console.log(await movies.aggregate(pipeline).toArray());
} finally {
  await client.close();
}
~~~

Resultado: até oito sugestões para um field indexado como autocomplete.

### Exemplo 5 — count com `$searchMeta`

~~~javascript
/**
 * @file Obtém metadata de contagem sem devolver os filmes.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
  const movies = client.db("sample_mflix").collection("movies");
  const metadata = await movies.aggregate([
    {
      $searchMeta: {
        index: "movies_search",
        text: {
          query: "mongodb",
          path: ["title", "fullplot"]
        },
        count: {
          type: "total"
        }
      }
    }
  ]).next();

  console.log(metadata);
} finally {
  await client.close();
}
~~~

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

| Mecanismo | Índice | Query | Força principal |
|---|---|---|---|
| B-tree | `createIndex({ field: 1 })` | `find()`/`$match` | equality, range, sort |
| text clássico | text index | `$text` | full-text básico |
| MongoDB Search | search index | `$search` | analyzers, score, autocomplete, facets |
| `$searchMeta` | search index | metadata | count/facets sem documentos normais |

> **Ligação entre capítulos:** aggregation e primeira posição de stage estão no capítulo 10; execução pelo driver no 11; B-tree/compound indexes no 09.

### Mapa mental de Search

~~~text
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
~~~

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
