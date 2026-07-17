# Aggregation Framework

## Objetivos do capítulo

- Construir pipelines como sequências de transformação.
- Distinguir stages, expressions e accumulators.
- Usar `$match`, `$project`, `$set`, `$unwind`, `$group`, `$sort` e `$limit`.
- Compreender `$lookup`, `$bucket`, `$facet`, `$out` e `$merge`.
- Otimizar ordem dos stages, índices e uso de memória.

---

## Conceitos Fundamentais

### O problema que uma pipeline resolve

`find()` é excelente quando o resultado continua a ser, essencialmente, um conjunto de documentos armazenados: filtrar, escolher fields, ordenar e limitar. Contudo, imagine que a aplicação não quer as encomendas; quer responder:

> Qual foi a receita total por país apenas para encomendas pagas?

Os documentos de origem podem ser:

```javascript
const orders = [
    { _id: 1, status: "paid", country: "PT", total: 40 },
    { _id: 2, status: "pending", country: "PT", total: 15 },
    { _id: 3, status: "paid", country: "PT", total: 25 },
    { _id: 4, status: "paid", country: "ES", total: 30 },
];
```

O output pretendido já não tem a forma de uma encomenda:

```javascript
[
    { _id: "PT", revenue: 65 },
    { _id: "ES", revenue: 30 },
]
```

É necessário executar várias transformações em sequência: remover encomendas não pagas, criar um grupo para cada país, somar os totais e ordenar os novos documentos. É esse encadeamento que o Aggregation Framework representa.

### A pipeline como linha de transformação

Uma **aggregation pipeline** é um array ordenado de **stages**. Cada stage recebe os documentos emitidos pelo anterior e produz documentos para o seguinte. O output de um stage torna-se o input do próximo.

```javascript
const pipeline = [
    { $match: { status: "paid" } },
    { $group: { _id: "$country", revenue: { $sum: "$total" } } },
    { $sort: { revenue: -1 } },
];
```

Leitura progressiva:

1. `$match` recebe as quatro encomendas e deixa passar apenas três com `status: "paid"`.
2. `$group` deixa de emitir encomendas individuais. Cria um documento por valor de `country`.
3. `_id: "$country"` diz que o valor do field `country` é a chave de agrupamento; o prefixo `$` significa “ler o valor deste field”.
4. `revenue: { $sum: "$total" }` cria no documento do grupo um field calculado com a soma dos valores `total`.
5. `$sort` recebe os documentos agrupados e ordena-os por `revenue` descendente.

A ordem é semântica. Se `$group` vier antes de `$match`, os documentos individuais deixam de ter `status` na forma original e a pergunta já não é equivalente. Uma pipeline não é uma lista de cláusulas que MongoDB pode interpretar em qualquer ordem.

### Stage versus expression versus accumulator

| Categoria   | Exemplo             | Papel                                           |
| ----------- | ------------------- | ----------------------------------------------- |
| stage       | `$match`, `$group`  | transforma o stream                             |
| expression  | `$add`, `$multiply` | calcula um valor dentro de stage                |
| accumulator | `$sum`, `$avg`      | agrega valores no `$group`/contextos suportados |

No exemplo anterior:

- `{ $match: ... }` é um stage completo porque ocupa um elemento do array da pipeline;
- `"$country"` é uma field-path expression que obtém um valor de cada documento;
- `{ $sum: "$total" }` é um accumulator dentro do `$group`;
- `{ $sum: ... }` não pode ser colocado sozinho como elemento da pipeline.

`$match` usa query predicate syntax. `$project`/`$set` usam aggregation expressions. O símbolo `$` aparece nos três contextos, mas isso não torna as formas intercambiáveis.

### Stages essenciais

| Stage                     | Função                                   |
| ------------------------- | ---------------------------------------- |
| `$match`                  | filtra                                   |
| `$project`                | inclui, exclui e calcula campos          |
| `$set` / `$addFields`     | adiciona/substitui campos                |
| `$unset`                  | remove campos                            |
| `$unwind`                 | emite um documento por elemento do array |
| `$group`                  | agrupa por key e acumula                 |
| `$sort`                   | ordena                                   |
| `$skip` / `$limit`        | pagina/restringe                         |
| `$count`                  | emite contagem                           |
| `$lookup`                 | junta dados de outra collection          |
| `$bucket` / `$bucketAuto` | cria intervalos                          |
| `$facet`                  | executa subpipelines sobre o mesmo input |
| `$out` / `$merge`         | persiste resultados                      |

`aggregate()`, por defeito, não modifica a collection. `$out` e `$merge` são exceções explícitas.

### aggregate versus find

`find()` é preferível para filtro, projection, sort e limit simples: intenção clara e API direta. `aggregate()` é adequado a transformações, joins, grouping, cálculo, múltiplas facetas e stages especializados. Não usar pipeline apenas por ser “mais poderoso”.

### group

`$group._id` define a chave, não o `_id` original. `_id: null` agrupa todos os documentos num grupo. `$sum: 1` conta documentos do grupo.

### unwind

`$unwind: "$items"` transforma cada elemento em um documento lógico. Um pedido com cinco linhas passa a cinco documentos. Isto altera cardinalidade; agrupar depois exige perceber qual unidade se está a contar.

> **Importante para o exame:** a ordem dos stages altera semântica e custo. `$match` filtra documentos, `$project` remodela-os e `$group` cria novos documentos agrupados; um accumulator como `$sum` não é um stage autónomo.

---

## Funcionamento Interno

### Streaming e blocking stages

Stages como `$match` e `$project` podem processar documento a documento. Stages como `$sort` e `$group` podem precisar de acumular estado e são blocking. Blocking aumenta memória e latência até ao primeiro resultado.

Stages sujeitos a limite de memória podem escrever temporariamente em disco conforme `allowDiskUseByDefault` e a opção `allowDiskUse`. O limiar frequentemente relevante é 100 MB por stage, com exceções/regras próprias. Spill evita erro, mas é mais lento e usa storage.

`$facet` é uma exceção decisiva: o documento resultante durante esse stage está limitado a 100 MB e o stage não pode fazer spill para disco, pelo que `allowDiskUse` não remove esse limite. O documento final da pipeline continua sujeito ao limite BSON de 16 MiB.

### Otimizador

MongoDB pode reordenar/coalescer stages sem alterar semântica:

- mover partes independentes de `$match` antes de projections calculadas;
- mover `$match` antes de `$sort`;
- coalescer `$sort + $limit`;
- eliminar fields desnecessários internamente.

Não escrever pipelines obscuros esperando que o otimizador corrija todas as decisões. Consultar `explain`; otimizações podem mudar entre versões.

### Índices

Um `$match` inicial pode usar índice. Um `$sort` inicial ou após `$match` compatível pode usar índice. Depois de stages que alteram a forma/cardinalidade, o índice da collection de origem já não ordena o stream transformado.

`$lookup` pode usar índices na foreign collection para as condições elegíveis. Sem índice, joins amplos tornam-se caros.

### SBE

O slot-based execution engine pode executar stages elegíveis com menor CPU/memória. Não é necessário memorizar todos os critérios para o exame Associate; saber que `explain` revela o plano e que a engine é uma decisão interna.

---

## Sintaxe

### Pipeline

Em `mongosh`, a assinatura conceptual é:

```javascript
const cursor = db.collection.aggregate(pipeline, options);
```

- `pipeline` é um array, mesmo que tenha apenas um stage;
- cada elemento do array é normalmente um documento com um stage de topo;
- `options` é um único documento opcional aplicado à execução completa;
- o retorno é um cursor, porque a pipeline pode produzir zero, um ou muitos documentos.

```javascript
const pipeline = [{ $match: { status: "paid" } }, { $limit: 10 }];

const options = {
    allowDiskUse: true,
    maxTimeMS: 10_000,
    comment: "monthly-report",
};

db.collection.aggregate(pipeline, options);
```

As options do exemplo não são stages e, por isso, ficam fora do array:

| Option         | O que controla                                                                  |
| -------------- | -------------------------------------------------------------------------------- |
| `allowDiskUse` | permite spill para disco em stages elegíveis quando excedem o limite de memória  |
| `maxTimeMS`    | tempo máximo de execução server-side antes de interrupção                        |
| `comment`      | etiqueta de diagnóstico para logs/profiler                                      |

`allowDiskUse: true` não elimina o limite BSON de 16 MiB do documento final nem o limite próprio de memória do `$facet`.

### Group

```javascript
const groupStage = {
    $group: {
        _id: "$groupKey",
        count: { $sum: 1 },
        total: { $sum: "$amount" },
        average: { $avg: "$amount" },
        first: { $first: "$value" },
        values: { $addToSet: "$value" },
    },
};
```

Anatomia de `$group`:

- `$group` é o nome do stage;
- `_id` é obrigatório e define a key de cada grupo; não é necessariamente o `_id` de origem;
- `_id: "$groupKey"` lê `groupKey` de cada documento;
- `_id: null` junta todos os documentos num único grupo;
- cada outro field do output, como `count` ou `total`, recebe normalmente uma expressão accumulator;
- `$sum: 1` soma a constante `1` por documento e, portanto, conta;
- `$sum: "$amount"` soma valores provenientes do field `amount`;
- `$addToSet` produz valores distintos, enquanto `$push` preservaria repetições.

`$first` e `$last` dependem da ordem do stream; usar `$sort` antes do `$group` quando a ordem faz parte da regra. Os nomes `first` e `values` no snippet são fields de output escolhidos pela aplicação, não palavras reservadas.

### Lookup simples

```javascript
const lookupStage = {
    $lookup: {
        from: "products",
        localField: "productId",
        foreignField: "_id",
        as: "product",
    },
};
```

Leitura das propriedades:

- `from` é o nome da foreign collection na mesma database;
- `localField` é o path lido no documento que percorre a pipeline;
- `foreignField` é o path comparado nos documentos de `from`;
- `as` é o nome do novo field criado no output;
- o novo field é sempre um array de matches, mesmo que a relação de negócio seja um-para-um.

Para uma relação um-para-um lógica, pode seguir `$unwind`, decidindo explicitamente `preserveNullAndEmptyArrays`. `$lookup` não cria constraints referenciais e não garante que exista exatamente um produto.

### Lookup correlacionado

```javascript
const correlatedLookupStage = {
    $lookup: {
        from: "warehouses",
        let: { orderedQuantity: "$ordered", itemSku: "$item" },
        pipeline: [
            {
                $match: {
                    $expr: {
                        $and: [
                            { $eq: ["$stockItem", "$$itemSku"] },
                            { $gte: ["$inStock", "$$orderedQuantity"] },
                        ],
                    },
                },
            },
        ],
        as: "eligibleWarehouses",
    },
};
```

Nesta forma:

- `let` publica valores do documento local como variáveis da subpipeline;
- `orderedQuantity: "$ordered"` cria a variável `$$orderedQuantity`;
- dentro da subpipeline, `$expr` permite usar aggregation expressions no `$match`;
- `$stockItem` e `$inStock` são fields da foreign collection;
- `$$itemSku` e `$$orderedQuantity` são as variáveis recebidas do documento local;
- cada documento de origem pode, por isso, executar a mesma estrutura com valores correlacionados diferentes.

`$expr` permite usar expressions no match e `$$variable` referencia variáveis de `let`.

---

## Exemplos

### Exemplo 1 — receita mensal por loja

```mongosh
use sample_supplies

const pipeline = [
  {
    $match: {
      saleDate: {
        $gte: ISODate("2016-01-01T00:00:00Z"),
        $lt: ISODate("2017-01-01T00:00:00Z")
      }
    }
  },
  { $unwind: "$items" },
  {
    $set: {
      lineRevenue: {
        $multiply: ["$items.price", "$items.quantity"]
      }
    }
  },
  {
    $group: {
      _id: {
        store: "$storeLocation",
        month: { $month: "$saleDate" }
      },
      revenue: { $sum: "$lineRevenue" },
      units: { $sum: "$items.quantity" }
    }
  },
  { $sort: { "_id.month": 1, revenue: -1 } }
];

db.sales.aggregate(pipeline);
```

Resultado: um documento por loja/mês com receita e unidades.

### Exemplo 2 — distribuição de ratings

```mongosh
use sample_mflix

db.movies.aggregate([
  {
    $match: {
      "imdb.rating": { $type: "number" }
    }
  },
  {
    $bucket: {
      groupBy: "$imdb.rating",
      boundaries: [0, 2, 4, 6, 8, 10.1],
      default: "other",
      output: {
        count: { $sum: 1 },
        examples: { $push: "$title" }
      }
    }
  },
  {
    $project: {
      count: 1,
      examples: { $slice: ["$examples", 3] }
    }
  }
]);
```

Resultado: buckets pelos limites inferiores e até três exemplos por bucket.

### Exemplo 3 — uma passagem, várias métricas

```mongosh
use sample_mflix

db.movies.aggregate([
  {
    $match: {
      year: { $gte: 2000 },
      "imdb.rating": { $type: "number" }
    }
  },
  {
    $facet: {
      topRated: [
        { $sort: { "imdb.rating": -1, title: 1 } },
        { $limit: 5 },
        { $project: { _id: 0, title: 1, "imdb.rating": 1 } }
      ],
      byYear: [
        { $group: { _id: "$year", count: { $sum: 1 } } },
        { $sort: { _id: 1 } }
      ],
      summary: [
        {
          $group: {
            _id: null,
            movies: { $sum: 1 },
            averageRating: { $avg: "$imdb.rating" }
          }
        }
      ]
    }
  }
]);
```

Resultado: um documento com três arrays, todos calculados sobre o mesmo input filtrado.

### Exemplo 4 — preservar ou eliminar arrays vazios

```mongosh
use sample_mflix

db.movies.aggregate([
  { $match: { title: { $in: ["The Matrix", "Unknown Example"] } } },
  {
    $unwind: {
      path: "$genres",
      includeArrayIndex: "genrePosition",
      preserveNullAndEmptyArrays: true
    }
  },
  {
    $project: {
      _id: 0,
      title: 1,
      genre: "$genres",
      genrePosition: 1
    }
  }
]);
```

Resultado: um documento por género; filmes sem array podem ser preservados.

---

## Explicação linha a linha

### Exemplo 1

1. `$match` reduz o ano antes de expandir arrays.
2. `$unwind` transforma cada sale em linhas.
3. `$multiply` calcula preço × quantidade.
4. `$group` usa chave composta loja/mês.
5. `$sum` soma receita e unidades.
6. `$sort` ordena output agregado.

### Exemplo 2

1. `$type` remove ratings ausentes/não numéricos.
2. `$bucket` usa boundaries inclusivos no limite inferior.
3. `$push` poderia crescer muito; o exemplo limita depois.
4. `$slice` reduz a apresentação, mas o accumulator já reuniu os títulos; para grande escala, escolher uma estratégia que limite durante a agregação quando disponível/aplicável.

### Exemplo 3

1. O `$match` comum corre antes do `$facet`.
2. Cada subpipeline recebe exatamente o mesmo stream.
3. `topRated` ordena e limita.
4. `byYear` agrupa.
5. `summary` usa `_id: null` para um único grupo.
6. `$facet` pode usar muita memória; não é automaticamente mais barato que queries separadas.

### Exemplo 4

1. `path` identifica o array.
2. `includeArrayIndex` guarda posição.
3. `preserveNullAndEmptyArrays` evita perder documentos sem elementos.
4. `$project` renomeia `genres` para `genre`.

---

## Casos Reais

- **E-commerce:** receita por categoria/mês.
- **Streaming:** duração média e top títulos.
- **Reservas:** ocupação por intervalo.
- **Dashboard:** `$facet` para resultados e métricas relacionadas.
- **Enriquecimento:** `$lookup` de pedidos com dados de produto.
- **Materialized view:** `$merge` atualiza uma collection de relatório.

---

## Performance

Filtrar cedo reduz cardinalidade. Colocar `$project` cedo apenas para reduzir fields é frequentemente desnecessário porque o otimizador já faz projection optimization; usá-lo cedo quando altera semântica ou calcula/remova dados necessários.

Evitar `$unwind` antes de `$match` quando o filtro pode ocorrer no documento original. Um array médio de 100 elementos multiplica o stream por 100. `$facet` mantém múltiplos resultados e tem limites próprios; usar outputs controlados.

`$sort`, `$group` e `$bucket` são blocking. Índices ajudam stages iniciais. `allowDiskUse` evita certos limites de memória, mas spill aumenta I/O; não é uma correção para pipeline sem seletividade.

---

## Armadilhas comuns

- **Confundir stage e operator expression.**
- **Pensar que `$group` preserva documentos completos:** só emite `_id` e accumulators.
- **Usar `$first` sem sort determinístico.**
- **Contar depois de `$unwind` sem perceber que conta elementos.**
- **Assumir que `$lookup` devolve objeto:** devolve array.
- **Perder documentos no `$unwind` por não usar `preserveNullAndEmptyArrays`.**
- **Começar com `$facet`:** impede usar um índice de origem como filtro inicial.
- **Usar `$push` ilimitado num grupo.**
- **Esperar que aggregation modifique dados sem `$out`/`$merge`.**
- **Usar `allowDiskUse` como desculpa para pipeline caro.**
- **Confundir `$set` aggregation stage com `$set` update operator.**

---

## O que costuma aparecer no exame

- Estrutura e ordem de uma pipeline.
- `$match`, `$project`, `$set`, `$group`, `$sort`, `$limit`.
- `$unwind` e mudança de cardinalidade.
- `$sum`, `$avg`, `$first`, `$last`, `$push`, `$addToSet`.
- `_id` do `$group`.
- `$lookup` e output array.
- `$bucket` e `$facet`.
- Blocking stages e memória.
- Uso de índices no início.
- `aggregate()` versus `find()`.

---

## Resumo

Aggregation encadeia stages sobre um stream. Stages filtram/transformam; expressions calculam valores; accumulators resumem grupos. Ordem e cardinalidade são centrais: filtrar cedo, expandir arrays conscientemente e ordenar antes de accumulators dependentes de ordem. Blocking stages usam memória e podem fazer spill. Índices beneficiam o início da pipeline. `$lookup` devolve arrays; `$facet` executa subpipelines; `$out`/`$merge` persistem.

Fontes oficiais: [aggregation](https://www.mongodb.com/docs/manual/aggregation/), [stages](https://www.mongodb.com/docs/manual/reference/operator/aggregation-pipeline/), [expressions](https://www.mongodb.com/docs/manual/reference/operator/aggregation/), [optimization](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-optimization/), [limits](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-limits/), [`$facet`](https://www.mongodb.com/docs/manual/reference/operator/aggregation/facet/) e [variables](https://www.mongodb.com/docs/manual/reference/aggregation-variables/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — Stage, expression e accumulator não são intercambiáveis. `$match` é stage; `$multiply` é expression; `$sum` pode ser accumulator ou expression conforme o contexto. A posição no documento/pipeline revela a função.

> ### ARMADILHA
>
> `$unwind` altera cardinalidade: depois dele, `$count` e `$group` contam elementos emitidos, não necessariamente documentos originais. `$lookup` devolve sempre um array, mesmo quando a relação de domínio parece one-to-one.

> ### DICA DE MEMORIZAÇÃO
>
> **Match reduz, unwind multiplica, group resume, project molda, sort ordena, limit trava.**

> ### COMPARAÇÃO
>
> Escolher `find()` ou `aggregate()` depende da transformação necessária.

| Necessidade               | `find()`       | `aggregate()`                      |
| ------------------------- | -------------- | ---------------------------------- |
| filtro/projection simples | direto e claro | possível, geralmente desnecessário |
| sort/limit simples        | API de cursor  | possível                           |
| grouping/accumulators     | não            | sim                                |
| join/facet/window         | não            | sim                                |
| retorno                   | `FindCursor`   | `AggregationCursor` no driver      |

> **Ligação entre capítulos:** índices do capítulo 09 ajudam os stages iniciais; sharding e distribuição estão no 10; execução no driver está no 12; Search adiciona stages especializados no 14.

### Mapa mental da pipeline

```text
collection
   |
 $match        reduz documentos
   |
 $unwind       pode multiplicar cardinalidade
   |
 $group        cria um documento por grupo
   |
 $project      define a forma
   |
 $sort
   |
 $limit
```

### Mini desafio

Uma collection tem 100 pedidos, cada um com 5 items. A pipeline faz `$unwind: "$items"` e depois `$count: "total"`. Sem responder com uma regra decorada, explica que unidade está a ser contada e que alterações de dados podem mudar o resultado.

---

## Resumo Rápido

- Pipeline é sequência ordenada de stages.
- Stage transforma stream; expression calcula; accumulator resume.
- `$unwind` muda cardinalidade.
- `$group` só preserva `_id` e accumulators declarados.
- Blocking stages podem usar muita memória/spill.
- `$facet` não faz spill e tem limite próprio de 100 MB.

---

## Checklist

- [ ] Distingo stage, expression e accumulator.
- [ ] Prevejo cardinalidade após cada stage.
- [ ] Sei usar `$match`, `$project`, `$set` e `$group`.
- [ ] Ordeno antes de `$first`/`$last` quando necessário.
- [ ] Sei que `$lookup` devolve array.
- [ ] Identifico blocking stages.
- [ ] Conheço limites de `$facet`.
- [ ] Escolho `find()` ou `aggregate()` pela necessidade.

---

## Perguntas para confirmar conhecimentos

1. Qual é a diferença entre stage, expression e accumulator?
2. Porque a ordem dos stages altera resultado e performance?
3. Que efeito tem `$unwind` sobre cardinalidade?
4. Que fields existem depois de `$group` se não forem acumulados?
5. Porque `$first` sem sort pode ser semanticamente inadequado?
6. Que shape devolve `$lookup` para cada documento de origem?
7. Que stages são frequentemente blocking?
8. Quando um stage inicial pode beneficiar de índice?
9. Porque `allowDiskUse` não remove o limite de `$facet`?
10. Quando `find()` é preferível a uma pipeline equivalente?
