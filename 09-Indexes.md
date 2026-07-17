# Índices e otimização de queries

## Objetivos do capítulo

- Compreender a estrutura e o custo dos índices MongoDB.
- Desenhar índices single-field, compound e multikey.
- Aplicar prefixos, direções e a regra ESR.
- Reconhecer unique, partial, sparse, TTL, text, geospatial, hashed e wildcard.
- Ler `explain("executionStats")` e identificar covered queries.

---

## Conceitos Fundamentais

### Vocabulário dos índices

Um capítulo de índices exige distinguir a definição do índice, os valores que ele contém e o plano que o utiliza:

| Conceito            | Definição                                                                                              |
| ------------------- | ------------------------------------------------------------------------------------------------------ |
| Index specification | configuração passada a `createIndex()`, incluindo key pattern e options                               |
| Key pattern         | fields e ordem/tipo declarados, por exemplo `{ status: 1, createdAt: -1 }`                            |
| Index key           | valor derivado de um documento para um ou mais fields indexados                                       |
| Index entry         | associação ordenada entre uma index key e a referência ao documento correspondente                   |
| `COLLSCAN`          | leitura de documentos da collection sem usar um índice para localizar os candidatos                  |
| `IXSCAN`            | percurso de um intervalo de index entries                                                             |
| `FETCH`             | leitura do documento completo depois de localizar a referência através do índice                     |
| Query plan          | estratégia escolhida para executar filtro, sort, projection e restantes requisitos                  |
| Selectivity         | capacidade de uma condição reduzir o conjunto de candidatos                                          |
| Cardinality         | quantidade de valores distintos ou de documentos numa fase, conforme o contexto                     |
| Working set         | dados e índices usados frequentemente que o sistema procura manter em memória                        |

### O que é um índice?

Sem índice adequado, MongoDB pode fazer `COLLSCAN` e examinar todos os documentos. Um índice guarda chaves ordenadas e referências aos documentos, permitindo localizar ranges e devolver resultados ordenados sem ler toda a collection.

O benefício em reads tem custos:

- storage e RAM;
- CPU/I/O em inserts, updates e deletes;
- build e manutenção operacional;
- mais opções para o query planner.

O índice não altera o resultado correto da query; altera os caminhos disponíveis para o obter. Cada write que modifica um field indexado pode também ter de atualizar index entries.

### Single-field, compound e multikey

- **Single-field index:** indexa um único field, por exemplo `{ email: 1 }`.
- **Compound index:** usa vários fields num key pattern ordenado, por exemplo `{ status: 1, createdAt: -1 }`. A ordem define prefixos e capacidade de suportar determinados sorts.
- **Multikey index:** índice que contém keys derivadas de elementos de um array. MongoDB torna-o multikey automaticamente quando um path indexado contém um array.

Compound e multikey não são alternativas mutuamente exclusivas. Um índice pode ser simultaneamente compound, por ter vários fields, e multikey, porque um desses fields contém um array.

### Tipos e propriedades

| Tipo/propriedade | Finalidade                              | Limitação/nota                           |
| ---------------- | --------------------------------------- | ---------------------------------------- |
| single-field     | filtro/sort num campo                   | simples e reutilizável                   |
| compound         | vários campos e sorts                   | ordem dos campos é decisiva              |
| multikey         | indexar elementos de arrays             | restrições em arrays compostos/cobertura |
| unique           | impor unicidade                         | constraint, não só performance           |
| partial          | indexar documentos que passam expressão | query deve implicar o filtro             |
| sparse           | omitir documentos sem campo             | partial é geralmente mais expressivo     |
| TTL              | expirar por campo Date                  | remoção assíncrona, não SLA exato        |
| text             | `$text` clássico                        | MongoDB Search é solução diferente       |
| 2d/2dsphere      | geospatial                              | formato/query específicos                |
| hashed           | igualdade e hashed sharding             | não suporta ranges ordenados             |
| wildcard         | paths desconhecidos/flexíveis           | não substitui desenho focado             |
| clustered        | ordem física de clustered collection    | criação/limites específicos              |

### Compound index e prefixos

Para `{ a: 1, b: -1, c: 1 }`, os prefixos são:

- `{ a: 1 }`
- `{ a: 1, b: -1 }`
- `{ a: 1, b: -1, c: 1 }`

Não é prefixo `{ b: -1 }` nem `{ a: 1, c: 1 }`. Uma query pode ainda usar partes não prefixo em alguns planos, mas com menor eficiência; “poder usar” não significa “usar de forma ótima”.

### ESR

Heurística para compound indexes:

1. **Equality**: campos com igualdade primeiro.
2. **Sort**: campos de ordenação seguintes.
3. **Range**: ranges depois.

Exemplo de workload:

```javascript
find({ status: "paid", total: { $gte: 100 } }).sort({ createdAt: -1 });
```

Índice candidato:

```javascript
const candidateIndex = { status: 1, createdAt: -1, total: 1 };
```

ESR é uma heurística. Se o range for extremamente seletivo, ERS pode ser melhor. Medir com dados representativos e `explain`.

### Direção de sort

Um compound index pode suportar o sort declarado ou o inverso completo. `{ score: -1, username: 1 }` suporta `{ score: -1, username: 1 }` e `{ score: 1, username: -1 }`. Não suporta automaticamente `{ score: -1, username: -1 }`.

> **Importante para o exame:** a ordem de um compound index determina os prefixos e os sorts que pode suportar. Num compound multikey index, cada documento pode ter no máximo um field indexado cujo valor seja array; `$elemMatch` também pode ser necessário para compor/intersetar bounds correlacionados.

---

## Funcionamento Interno

MongoDB usa estruturas B-tree para índices comuns. Nós mantêm chaves ordenadas; a pesquisa desce a árvore e percorre folhas para ranges. Complexidade conceptual de lookup é O(log n), mas o custo real depende de fan-out, cache, número de keys examinadas e document fetches.

### Multikey

Ao indexar um campo array, MongoDB cria uma key por elemento e marca o índice multikey. Uma query `{ tags: "mongodb" }` localiza documentos por essa key. O índice pode aumentar muito se arrays forem grandes.

Num compound multikey index, cada documento pode ter no máximo um campo indexado cujo valor seja array. A regra impede a explosão cartesiana de combinações. O mesmo índice pode ser usado por documentos diferentes com arrays em campos diferentes, desde que nenhum documento viole a restrição.

### Query planner

O planner considera índices candidatos, gera planos, testa trabalho inicial e escolhe um winning plan. Plan cache evita replanear sempre. Alterações de distribuição podem tornar um plano menos adequado; não inferir performance apenas pelo nome do índice.

### Covered query

Se filtro e projection usam apenas campos do índice, o servidor pode responder sem `FETCH`. Sinais:

- winning plan contém `IXSCAN`/`PROJECTION_COVERED` conforme versão;
- `totalDocsExamined: 0`;
- `totalKeysExamined` próximo de `nReturned`.

Incluir `_id` por defeito quebra cobertura se o índice não contiver `_id`.

### Index intersection

O planner pode combinar índices, mas um compound index alinhado com o access pattern é frequentemente superior e suporta sort. Não desenhar uma estratégia dependente de intersection sem medir.

---

## Sintaxe

### Criar índices

```javascript
await collection.createIndex(
    { status: 1, createdAt: -1 },
    { name: "status_createdAt_desc" },
);

await collection.createIndexes([
    {
        key: { email: 1 },
        name: "email_unique",
        unique: true,
    },
    {
        key: { expiresAt: 1 },
        name: "expires_ttl",
        expireAfterSeconds: 0,
    },
]);
```

### Partial index

```javascript
await users.createIndex(
    { email: 1 },
    {
        name: "active_email_unique",
        unique: true,
        partialFilterExpression: { deletedAt: { $exists: false } },
    },
);
```

Isto impõe unicidade apenas entre documentos abrangidos. A semântica deve ser deliberada: documentos soft-deleted podem reutilizar email.

### Listar e remover

```javascript
const indexes = await collection.listIndexes().toArray();
await collection.dropIndex("status_createdAt_desc");
```

Remover índice em produção requer confirmar queries dependentes e plano de rollback. Ocultar um índice, quando suportado, permite testar o efeito antes de o apagar.

### Explain no driver

```javascript
const plan = await collection
    .find(filter, { projection })
    .sort(sort)
    .explain("executionStats");
```

Modos comuns: `"queryPlanner"`, `"executionStats"` e `"allPlansExecution"`. O último acrescenta custo de diagnóstico.

---

## Exemplos

### Exemplo 1 — compound index para filtro e sort

```javascript
/**
 * @file Cria e confirma um índice alinhado com listagem de encomendas.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const orders = client.db("shop").collection("orders");
    await orders.createIndex(
        { customerId: 1, status: 1, createdAt: -1 },
        { name: "customer_status_createdAt" },
    );

    const plan = await orders
        .find(
            { customerId: "customer-42", status: "paid" },
            { projection: { _id: 0, orderNumber: 1, createdAt: 1 } },
        )
        .sort({ createdAt: -1 })
        .limit(20)
        .explain("executionStats");

    console.log(plan.executionStats);
} finally {
    await client.close();
}
```

Resultado: o winning plan deve ser inspecionado; espera-se que o índice possa filtrar e ordenar.

### Exemplo 2 — covered query

```javascript
/**
 * @file Demonstra uma query que pode ser respondida só pelo índice.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    await movies.createIndex(
        { title: 1, year: 1 },
        { name: "title_year_cover" },
    );

    const plan = await movies
        .find(
            { title: "The Matrix" },
            { projection: { _id: 0, title: 1, year: 1 } },
        )
        .explain("executionStats");

    console.log({
        returned: plan.executionStats.nReturned,
        keys: plan.executionStats.totalKeysExamined,
        documents: plan.executionStats.totalDocsExamined,
    });
} finally {
    await client.close();
}
```

Resultado esperado para cobertura: `totalDocsExamined` igual a 0.

### Exemplo 3 — multikey e `$elemMatch`

```javascript
/**
 * @file Indexa subcampos de um array de linhas e analisa a query.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const orders = client.db("shop").collection("orders");
    await orders.createIndex(
        { "items.sku": 1, "items.quantity": 1 },
        { name: "items_sku_quantity" },
    );

    const plan = await orders
        .find({
            items: {
                $elemMatch: {
                    sku: "BOOK-001",
                    quantity: { $gte: 2 },
                },
            },
        })
        .explain("executionStats");

    console.log(plan.queryPlanner.winningPlan);
} finally {
    await client.close();
}
```

Resultado: índice multikey; `$elemMatch` mantém as condições ligadas ao mesmo subdocumento.

### Exemplo 4 — TTL para sessões

```javascript
/**
 * @file Configura expiração automática baseada numa data absoluta.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const sessions = client.db("app").collection("sessions");
    const indexName = await sessions.createIndex(
        { expiresAt: 1 },
        {
            name: "sessions_expiry",
            expireAfterSeconds: 0,
        },
    );

    console.log(indexName);
} finally {
    await client.close();
}
```

Resultado: documentos tornam-se elegíveis para remoção após `expiresAt`; a remoção não ocorre exatamente nesse milissegundo.

---

## Explicação linha a linha

### Exemplo 1

1. Equality fields `customerId` e `status` precedem o sort.
2. `createdAt: -1` corresponde à direção pedida.
3. Limit reduz o top N.
4. `explain("executionStats")` executa e mede.
5. A conclusão deve usar stage tree e métricas, não apenas existência do índice.

### Exemplo 2

1. O índice contém filtro e campos devolvidos.
2. A projection exclui `_id`.
3. Não é preciso ler o documento se o planner escolher cobertura.
4. Compara keys examinadas, docs examinados e resultados.

### Exemplo 3

1. Ambos os paths pertencem ao mesmo array `items`; isto é permitido.
2. O índice torna-se multikey.
3. Query `$elemMatch` alinha a correlação de elementos.
4. Explain confirma se e como o índice foi escolhido.

### Exemplo 4

1. TTL index é single-field sobre Date.
2. `expireAfterSeconds: 0` interpreta o campo como instante absoluto.
3. O monitor TTL remove documentos de forma assíncrona.
4. Aplicações não devem depender de eliminação instantânea para autorização.

---

## Casos Reais

- **Login:** unique index em email normalizado.
- **Histórico:** compound index por owner, status e data.
- **Catálogo:** multikey em tags.
- **Soft delete:** partial unique index entre documentos ativos.
- **Sessões:** TTL por `expiresAt`.
- **Geo:** 2dsphere para proximidade.
- **Sharding:** hashed index para distribuição de equality key, consciente de ranges.

---

## Performance

Índice eficiente maximiza seletividade e suporta sort/projection. Comparar:

```text
nReturned
totalKeysExamined
totalDocsExamined
executionTimeMillis
```

Métricas ideais dependem da query. Uma query que devolve 100 mil documentos tem inevitavelmente custo elevado. O objetivo não é “usar IXSCAN”, mas minimizar trabalho para o resultado necessário.

Índices redundantes aumentam working set. Um compound index pode tornar um single-field prefix redundante, exceto se unique/sparse/collation/opções ou tamanho justificarem ambos. Medir reads e writes antes de remover.

Builds consomem I/O/CPU e storage. Planear capacity e observar replicação. Em produção, nomes explícitos e migrations idempotentes tornam o ciclo de vida auditável.

---

## Armadilhas comuns

- **Criar índice para cada campo:** penaliza writes e cache.
- **Ignorar ordem do compound index.**
- **Achar que qualquer permutação usa o mesmo índice igualmente.**
- **Colocar range antes do sort sem medir.**
- **Esquecer `_id` numa covered query.**
- **Confundir multikey com compound:** um descreve array, o outro vários fields.
- **Compound index com dois arrays no mesmo documento:** inválido.
- **Assumir TTL imediato:** o monitor é assíncrono.
- **Usar hashed index para range sort:** hashes destroem ordenação original.
- **Confundir text index clássico com MongoDB Search index.**
- **Forçar hint permanentemente sem governar o índice.**
- **Olhar só para `executionTimeMillis` num ambiente frio/variável.**

---

## O que costuma aparecer no exame

- `COLLSCAN` versus `IXSCAN`.
- Single-field, compound e multikey.
- Prefixos de compound index.
- Direção e inversão total de sort.
- ESR.
- Unique, partial, sparse e TTL.
- Covered query.
- `totalKeysExamined`, `totalDocsExamined` e `nReturned`.
- Custo dos índices em writes.
- Restrição de arrays em compound multikey.

---

## Resumo

Índices B-tree trocam storage e custo de write por menos trabalho em reads. Compound indexes dependem da ordem, dos prefixos e da direção. ESR é uma boa heurística, não uma lei. Arrays produzem multikey indexes e restrições próprias. Covered queries evitam document fetch. `explain("executionStats")` é a prova: comparar plano e trabalho com resultados. Criar apenas índices que suportam constraints ou access patterns reais.

Fontes oficiais: [indexes](https://www.mongodb.com/docs/manual/indexes/), [index types](https://www.mongodb.com/docs/manual/core/indexes/index-types/), [compound indexes](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/), [multikey](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-multikey/), [ESR](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/) e [explain](https://www.mongodb.com/docs/manual/reference/explain-results/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — Num compound index, a ordem cria prefixos e condiciona sort/range. ESR é uma heurística: Equality, Sort, Range. A prova de eficiência não é a presença de `IXSCAN`, mas o trabalho observado em `executionStats` relativamente a `nReturned`.

> ### ARMADILHA
>
> “Compound” significa vários fields; “multikey” significa que pelo menos um path indexado contém array. Um índice pode ser simultaneamente compound e multikey. Num documento, um compound multikey index não pode indexar mais de um field cujo valor seja array.

> ### DICA DE MEMORIZAÇÃO
>
> **ESR põe igualdade antes de ordenar e range; explain prova, não promete.**

> ### COMPARAÇÃO
>
> Estas propriedades descrevem e restringem índices de formas diferentes.

| Conceito      | Define                      | Pergunta-chave                    |
| ------------- | --------------------------- | --------------------------------- |
| single-field  | uma key                     | a query usa esse field?           |
| compound      | várias keys ordenadas       | qual é o prefixo e a ordem?       |
| multikey      | indexa elementos de array   | que bounds podem ser combinados?  |
| unique        | constraint entre documentos | que duplicação é proibida?        |
| partial       | subset por expressão        | a query implica o filtro parcial? |
| covered query | resposta só pelo índice     | existe `FETCH`/docs examinados?   |

> **Ligação entre capítulos:** access patterns e arrays vêm dos capítulos 02 e 05; sort/projection do 07; stages iniciais de aggregation do 10.

### Fluxograma: preciso de um índice?

```text
Existe access pattern ou constraint frequente?
  |-- não --> não criar por antecipação
  `-- sim --> equality / sort / range conhecidos?
                |-- não --> medir e clarificar a query
                `-- sim --> desenhar ordem candidata (ESR)
                              |
                              v
                     testar com dados + explain
                              |
                              +-> pouco trabalho: manter/governar
                              `-> muito trabalho: rever índice/query/modelo
```

### Mini desafio

Para a query `{ status: "paid", createdAt: { $gte: start } }` com sort `{ total: -1 }`, propõe uma ordem candidata de compound index, explica a aplicação de ESR e indica que métricas confirmariam ou rejeitariam a escolha.

---

## Resumo Rápido

- Índice troca storage/write cost por menos read work.
- Ordem de compound index define prefixos e sorts.
- ESR é heurística; medir pode justificar ERS.
- Array indexado cria multikey index.
- Covered query pode evitar `FETCH`.
- `IXSCAN` sozinho não prova eficiência: ler `executionStats`.

---

## Checklist

- [ ] Distingo single, compound e multikey.
- [ ] Identifico prefixos válidos de um compound index.
- [ ] Aplico ESR e conheço a exceção ERS.
- [ ] Analiso direção e inversão total de sort.
- [ ] Conheço restrições compound multikey.
- [ ] Reconheço covered query e efeito de `_id`.
- [ ] Leio keys/docs examined e `nReturned`.
- [ ] Avalio custos de índices em writes e memória.

---

## Perguntas para confirmar conhecimentos

1. Que trade-off principal introduz cada índice?
2. Quais são os prefixos de `{ a: 1, b: -1, c: 1 }`?
3. O que significa ESR e porque não é uma lei absoluta?
4. Que sorts pode suportar um compound index e o seu inverso completo?
5. Qual é a diferença entre compound e multikey?
6. Que restrição existe quando mais de um indexed field é array no mesmo documento?
7. Quando `$elemMatch` ajuda a combinar bounds multikey?
8. Que condições tornam uma query covered?
9. Porque `IXSCAN` pode ainda representar um plano ineficiente?
10. Que relação entre `totalKeysExamined`, `totalDocsExamined` e `nReturned` procurarias?
