# Índices e otimização de queries

## Objetivos do capítulo

- Compreender a estrutura e o custo dos índices MongoDB.
- Desenhar índices single-field, compound e multikey.
- Aplicar prefixos, direções e a regra ESR.
- Reconhecer unique, partial, sparse, TTL, text, geospatial, hashed e wildcard.
- Ler `explain("executionStats")` e identificar covered queries.

---

## Conceitos Fundamentais

### Começar pelo problema: como encontra MongoDB um documento?

Considere uma collection `orders` com milhões de encomendas e a seguinte query:

```javascript
const filter = { status: "paid" };
const order = await orders.findOne(filter);
```

O resultado pedido é simples: encontrar uma encomenda cujo `status` seja `"paid"`. O problema é saber **onde** está esse documento.

Sem uma estrutura auxiliar, o servidor pode começar no primeiro documento da collection, verificar o `status`, avançar para o segundo e repetir até encontrar um match. Esse percurso é um **collection scan**, representado em `explain()` por `COLLSCAN`. Se existirem dez milhões de documentos, uma pergunta seletiva pode obrigar a examinar uma grande parte desses dez milhões.

Um **índice** é uma estrutura de dados separada da collection que guarda valores de fields escolhidos numa ordem pesquisável, juntamente com referências para os documentos de origem. É semelhante ao índice remissivo de um livro:

- o livro continua a conter o texto completo;
- o índice contém termos selecionados e a localização das páginas;
- procurar no índice evita percorrer todas as páginas;
- acrescentar ou alterar páginas pode obrigar a atualizar o índice.

Em MongoDB, criar `{ status: 1 }` não copia o documento inteiro. Conceptualmente, cria entradas semelhantes a estas:

```text
"cancelled" -> documento 104
"paid"      -> documento 017
"paid"      -> documento 281
"pending"   -> documento 092
```

As entradas estão ordenadas pelo valor da chave. Para procurar `"paid"`, o servidor localiza a primeira entrada relevante, percorre as entradas contíguas com esse valor e segue as referências para obter os documentos. Em `explain()`, o percurso do índice aparece normalmente como `IXSCAN`; quando é preciso ler o documento completo, aparece também uma fase `FETCH`.

Esta distinção é central:

```text
collection = documentos completos
índice      = chaves escolhidas, ordenadas, com referências aos documentos
query       = pedido que o query planner tenta resolver pelo caminho de menor custo
```

O índice não muda quais documentos correspondem ao filtro. Muda os caminhos disponíveis para os localizar e, em alguns casos, para os devolver já ordenados.

### Do documento à entrada do índice

Partamos de dois documentos:

```javascript
const orderA = {
    _id: 17,
    status: "paid",
    createdAt: new Date("2026-06-10T09:00:00Z"),
    total: 125,
};

const orderB = {
    _id: 92,
    status: "pending",
    createdAt: new Date("2026-06-11T14:00:00Z"),
    total: 80,
};
```

Com o índice `{ status: 1, createdAt: -1 }`, MongoDB deriva de cada documento uma **index key** composta:

```text
documento 17 -> ["paid",    2026-06-10T09:00:00Z]
documento 92 -> ["pending", 2026-06-11T14:00:00Z]
```

Cada key, acompanhada da referência ao documento, forma uma **index entry**. O `1` declara ordem ascendente para `status`; o `-1` declara ordem descendente para `createdAt` dentro de cada valor de `status`.

Isto significa que o índice é ordenado primeiro por `status` e, entre documentos com o mesmo `status`, por `createdAt` do mais recente para o mais antigo. Não significa “dar prioridade de performance de 1 e -1”, nem significa incluir/excluir fields como numa projection.

### Vocabulário usado no resto do capítulo

Agora que existe um modelo mental, os termos deixam de ser definições soltas:

| Conceito            | Significado neste capítulo                                                                             |
| ------------------- | ------------------------------------------------------------------------------------------------------ |
| Index specification | definição completa de um índice: key pattern e propriedades opcionais                                 |
| Key pattern         | documento que declara paths, ordem e tipo, por exemplo `{ status: 1, createdAt: -1 }`                 |
| Index key           | conjunto de valores extraído de um documento segundo o key pattern                                    |
| Index entry         | index key acompanhada da referência ao documento de origem                                            |
| `COLLSCAN`          | percurso de documentos da collection para descobrir os matches                                        |
| `IXSCAN`            | percurso de entradas de um índice                                                                      |
| `FETCH`             | leitura do documento depois de localizar a sua referência no índice                                   |
| Query plan          | estratégia escolhida para executar filtro, sort, projection e outros requisitos                      |
| Selectivity         | capacidade de uma condição eliminar candidatos; uma igualdade por email único é muito seletiva       |
| Cardinality         | quantidade de valores distintos ou de documentos, consoante o contexto                               |
| Working set         | dados e índices consultados com frequência que idealmente permanecem em memória                        |

### O custo que acompanha o benefício

Uma read pode ficar muito mais barata porque examina poucas index keys e poucos documentos. Contudo, cada índice ocupa storage e memória. Além disso:

1. num insert, MongoDB acrescenta entradas a todos os índices aplicáveis;
2. num update, pode remover a entrada antiga e inserir uma nova quando muda um field indexado;
3. num delete, remove também as entradas do documento;
4. durante um index build, percorre dados e consome recursos;
5. o query planner passa a ter mais planos candidatos para avaliar.

Por isso, “indexar todos os fields” não é uma estratégia. Um índice deve nascer de um **access pattern**: filtro, sort e projection que a aplicação executa de forma importante e frequente.

### Primeiro caso: índice de um só field

Um **single-field index** possui um único path no key pattern:

```javascript
await users.createIndex({ email: 1 });
```

Este índice pode ajudar uma query como `{ email: "ana@example.com" }`. Para igualdade, `1` e `-1` permitem ambos localizar o valor; a direção torna-se especialmente relevante para sorts e para combinações com vários fields.

Todas as collections normais possuem, por defeito, um unique index em `_id`. Esse índice explica por que motivo `_id` identifica documentos eficientemente e por que não é possível inserir dois documentos com o mesmo `_id`.

### Segundo caso: índice composto e ordem dos fields

Um **compound index** inclui dois ou mais paths:

```javascript
const keyPattern = { status: 1, createdAt: -1 };
```

A ordem escrita no objeto faz parte da definição. O índice está organizado por `status` e só depois por `createdAt`. Isto favorece, por exemplo:

```javascript
orders.find({ status: "paid" }).sort({ createdAt: -1 });
```

Depois de localizar a zona de `status: "paid"`, as entradas já estão na ordem temporal pedida. O mesmo índice não é equivalente a `{ createdAt: -1, status: 1 }`: nessa versão, datas diferentes ficam primeiro separadas e os estados encontram-se dispersos dentro da ordem temporal.

### Prefixos de um índice composto

Os fields iniciais de um compound index formam os seus **prefixos**. Para `{ a: 1, b: -1, c: 1 }`, os prefixos são:

- `{ a: 1 }`;
- `{ a: 1, b: -1 }`;
- `{ a: 1, b: -1, c: 1 }`.

`{ b: -1 }` não é prefixo porque ignora o primeiro field. `{ a: 1, c: 1 }` também não é um prefixo contínuo porque salta `b`. O servidor pode ainda aproveitar partes do índice em alguns planos, mas aproveitar não é o mesmo que restringir eficientemente todas as condições.

### Direção e ordenação

Num single-field index, MongoDB consegue percorrer as entradas para a frente ou para trás. Num compound index, a **relação entre as direções** é que importa.

O índice `{ score: -1, username: 1 }` suporta:

- `{ score: -1, username: 1 }`, percorrendo o índice na direção declarada;
- `{ score: 1, username: -1 }`, percorrendo todo o índice no sentido inverso.

Não suporta automaticamente `{ score: -1, username: -1 }`, porque apenas a segunda direção foi invertida. Se o primeiro field estiver fixado por igualdade, algumas combinações adicionais podem ser possíveis; deve confirmar-se o plano real com `explain()`.

### Arrays e índices multikey

Se o path indexado contiver um array, MongoDB cria normalmente uma index key por elemento e marca o índice como **multikey**:

```javascript
const article = {
    _id: 1,
    tags: ["mongodb", "nodejs", "databases"],
};
```

Com `{ tags: 1 }`, o documento contribui conceptualmente com três entradas. Assim, `{ tags: "mongodb" }` pode usar o índice sem a aplicação desenrolar o array.

Um índice pode ser simultaneamente compound e multikey: compound porque declara vários fields e multikey porque pelo menos um path encontra arrays. Num compound multikey index, cada documento pode ter no máximo um field indexado cujo valor seja array; isto evita criar um produto cartesiano ambíguo entre vários arrays.

### ESR aparece depois dos fundamentos

Só agora faz sentido introduzir **ESR**, uma heurística para ordenar fields de um compound index:

1. **Equality** — fields testados por igualdade;
2. **Sort** — fields que devem manter a ordem do resultado;
3. **Range** — fields com intervalos como `$gt`, `$gte`, `$lt`, `$lte` ou, em certos contextos, listas amplas.

Para:

```javascript
orders
    .find({ status: "paid", total: { $gte: 100 } })
    .sort({ createdAt: -1 });
```

um candidato coerente com ESR é:

```javascript
const candidateIndex = { status: 1, createdAt: -1, total: 1 };
```

`status` restringe primeiro uma igualdade; `createdAt` preserva o sort dentro dessa zona; `total` delimita o range. ESR não é uma lei universal: se o range for extremamente seletivo e o sort abranger poucos resultados, uma ordem ERS pode ser melhor. A distribuição real dos dados e `explain("executionStats")` decidem.

### Tipos de índice versus propriedades do índice

Nem todos os nomes abaixo descrevem a mesma dimensão. `compound`, `text` e `2dsphere` dizem sobretudo **como/que valores são indexados**; `unique`, `partial` e `hidden` são **propriedades** aplicadas a um índice compatível.

| Tipo ou propriedade | O que faz                                                                    | Quando não deve ser inferido                         |
| ------------------- | --------------------------------------------------------------------------- | ---------------------------------------------------- |
| single-field        | indexa um path                                                               | não resolve automaticamente queries noutros fields   |
| compound            | ordena keys por vários paths                                                 | a ordem dos paths não é intercambiável               |
| multikey            | cria keys a partir de arrays                                                 | arrays grandes aumentam muito o número de entries    |
| unique              | rejeita index keys duplicadas                                                | é constraint de dados, não apenas otimização         |
| partial             | inclui apenas documentos que satisfazem um filtro                            | a query tem de ser compatível com esse filtro        |
| sparse              | omite documentos sem o field                                                 | partial é normalmente mais expressivo                |
| TTL                 | agenda remoção de documentos segundo um BSON Date                            | não executa exatamente no segundo de expiração       |
| text                | suporta pesquisa `$text` clássica                                            | não é o mesmo que MongoDB Search                     |
| `2d` / `2dsphere`   | indexa dados geoespaciais                                                    | exige formato e operadores geoespaciais apropriados  |
| hashed              | indexa o hash do valor, útil em igualdade/sharding                           | perde ordenação útil para range                      |
| wildcard            | abrange paths variáveis                                                      | não substitui índices focados em queries críticas    |
| hidden              | continua mantido, mas fica invisível ao planner                              | não reduz o custo dos writes                         |

> **Importante para o exame:** um key pattern descreve fields e ordem/tipo; as options descrevem propriedades. A ordem de um compound index determina prefixos e sorts possíveis. Um multikey index nasce dos valores array existentes, não de uma option `multikey: true`.

---

## Funcionamento Interno

MongoDB usa estruturas B-tree para muitos índices. A árvore permite localizar uma zona de chaves sem comparar sequencialmente todas as entradas. Depois de chegar ao primeiro limite relevante, o servidor percorre as entries seguintes até ao limite final.

É útil pensar em três passos, sem transformar a notação assintótica numa promessa de latência:

1. **procurar o limite inicial** na árvore;
2. **percorrer as keys candidatas** dentro do intervalo;
3. **fazer `FETCH`** dos documentos quando o índice não contém tudo o que a operação precisa.

Um lookup pontual tem comportamento conceptual próximo de O(log n), mas o custo completo depende do número de keys examinadas, da quantidade de `FETCH`, da presença em cache, do tamanho das entries e do número de resultados. Um índice usado para devolver 800 000 matches não torna essa transferência barata.

### Multikey

Ao indexar um campo array, MongoDB cria uma key por elemento e marca o índice multikey. Uma query `{ tags: "mongodb" }` localiza documentos por essa key. O índice pode aumentar muito se arrays forem grandes.

Num compound multikey index, cada documento pode ter no máximo um campo indexado cujo valor seja array. A regra impede a explosão cartesiana de combinações. O mesmo índice pode ser usado por documentos diferentes com arrays em campos diferentes, desde que nenhum documento viole a restrição.

### Query planner

O query planner recebe o filtro, sort, projection e options. Considera planos como `COLLSCAN` e percursos sobre índices elegíveis, testa candidatos e escolhe um winning plan. A plan cache evita repetir toda a competição para cada forma de query.

Ter um índice com o nome “certo” não prova que ele é adequado nem que foi escolhido. É necessário observar:

- `winningPlan` para perceber o caminho;
- `nReturned` para saber quantos documentos saíram;
- `totalKeysExamined` para saber quantas entries foram percorridas;
- `totalDocsExamined` para saber quantos documentos foram lidos;
- presença de `SORT` para identificar ordenação não satisfeita pelo índice.

### Covered query

Se filtro e projection usam apenas campos do índice, o servidor pode responder sem `FETCH`. Sinais:

- winning plan contém `IXSCAN`/`PROJECTION_COVERED` conforme versão;
- `totalDocsExamined: 0`;
- `totalKeysExamined` próximo de `nReturned`.

Incluir `_id` por defeito quebra cobertura se o índice não contiver `_id`.

### Index intersection

O planner pode combinar índices, mas um compound index alinhado com o access pattern é frequentemente superior e suporta sort. Não desenhar uma estratégia dependente de intersection sem medir.

### Índice de suporte da shard key

Uma shard key precisa de um índice de suporte compatível. Esse índice organiza os valores usados na distribuição, mas não substitui os índices exigidos pelas restantes queries. Em sharding, avaliar separadamente:

1. se o filtro permite a `mongos` escolher os shards;
2. se cada shard possui um índice eficiente para executar o trabalho local.

Uma query pode ser targeted e fazer `COLLSCAN` no shard de destino. Também pode ser scatter-gather e usar `IXSCAN` em todos os shards. O capítulo 10 desenvolve esta distinção.

---

## Sintaxe

### As duas assinaturas antes do exemplo

No Node.js Driver, as formas conceptuais são:

```javascript
const indexName = await collection.createIndex(keyPattern, options);

const indexNames = await collection.createIndexes(indexSpecifications, options);
```

Não são a mesma assinatura com singular/plural:

| Parte                 | `createIndex()`                                                    | `createIndexes()`                                                   |
| --------------------- | ------------------------------------------------------------------ | ------------------------------------------------------------------- |
| primeiro argumento    | um key pattern                                                     | array de index specifications                                       |
| onde fica `key`       | o próprio primeiro argumento                                       | propriedade `key` de cada specification                             |
| propriedades do índice| no segundo argumento `options`                                     | normalmente dentro de cada specification                            |
| retorno               | `Promise<string>` com o nome do índice                             | `Promise<string[]>` com os nomes                                    |

O segundo argumento é opcional. `await` espera que o comando termine ou falhe; o valor devolvido é o nome, não um cursor nem o próprio índice materializado em JavaScript.

### Anatomia de `createIndex()`

```javascript
const indexName = await collection.createIndex(
    { status: 1, createdAt: -1 },
    { name: "status_createdAt_desc" },
);
```

Leitura de fora para dentro:

1. `collection` é o objeto `Collection` onde o índice será criado.
2. `createIndex(...)` pede a criação de **um** índice.
3. `{ status: 1, createdAt: -1 }` é o `keyPattern`, primeiro argumento obrigatório.
4. `status` e `createdAt` são paths dos documentos; dot notation, como `"customer.email"`, também é possível.
5. `1` significa ordem ascendente e `-1` descendente neste índice B-tree.
6. A ordem das propriedades é significativa: `status` é o primeiro field do compound index.
7. `{ name: ... }` é o objeto de options.
8. `name` dá um identificador operacional explícito ao índice; não altera as queries que ele suporta.
9. `indexName` recebe a string devolvida pelo driver, neste caso `"status_createdAt_desc"` se a operação tiver sucesso.

Sem `name`, o servidor gera normalmente um nome a partir dos fields e direções, por exemplo `status_1_createdAt_-1`. Dar nome é útil em migrations e observabilidade, mas a aplicação não deve presumir que mudar o nome muda a definição do key pattern.

### Valores aceites no key pattern

Os valores do key pattern não são options booleanas. Selecionam a ordem ou a família do índice:

| Valor                  | Significado principal                                                    |
| ---------------------- | ------------------------------------------------------------------------ |
| `1`                    | B-tree ascendente                                                        |
| `-1`                   | B-tree descendente                                                       |
| `"hashed"`            | hash do valor; igualdade e distribuição hashed                           |
| `"text"`              | índice para queries `$text` clássicas                                    |
| `"2dsphere"` / `"2d"`| índice geoespacial conforme o modelo de coordenadas                       |
| `"$**": 1`            | wildcard index sobre paths dinâmicos                                     |

Exemplo: `{ location: "2dsphere" }` não quer dizer ordenar `location`; escolhe um tipo de índice geoespacial. As propriedades compatíveis variam por tipo.

### Anatomia de `createIndexes()`

```javascript
const createdNames = await collection.createIndexes([
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

Aqui, o primeiro argumento é um array com duas **index specifications**:

- cada objeto do array descreve um índice independente;
- `key` é obrigatório em cada specification e contém o respetivo key pattern;
- `name`, `unique` e `expireAfterSeconds` são propriedades daquele índice;
- `createdNames` é um array de strings, não um result object com documentos.

Primeira specification:

```javascript
const uniqueIndexSpecification = {
    key: { email: 1 },
    name: "email_unique",
    unique: true,
};
```

- `key: { email: 1 }` cria um single-field ascending index em `email`;
- `name` permite referenciá-lo explicitamente em operações como `dropIndex()`;
- `unique: true` acrescenta uma constraint: um insert/update que produza uma key já existente é rejeitado;
- construir este índice numa collection que já contenha duplicados falha; o índice não “limpa” os dados.

Segunda specification:

```javascript
const ttlIndexSpecification = {
    key: { expiresAt: 1 },
    name: "expires_ttl",
    expireAfterSeconds: 0,
};
```

- `key: { expiresAt: 1 }` indexa um field que deve conter BSON Date para o caso normal;
- `expireAfterSeconds` transforma-o num TTL index;
- o valor `0` significa que a data armazenada em `expiresAt` é o instante de expiração: não se acrescenta um intervalo fixo;
- a remoção é assíncrona, logo um documento pode permanecer algum tempo depois desse instante;
- `0` **não** significa “apagar imediatamente após o insert”, salvo se `expiresAt` já representar o momento atual/passado.

Se fosse `expireAfterSeconds: 3600`, a expiração seria calculada aproximadamente uma hora depois do BSON Date indexado.

### Propriedades do índice e options do comando

Convém ainda separar **propriedades que passam a fazer parte da definição de um índice** de **options que controlam a execução do comando de criação**:

| Campo/option              | Scope                              | Efeito                                                                               | Cuidado principal                                                                    |
| ------------------------- | ---------------------------------- | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------ |
| `name`                    | definição de cada índice           | define o nome operacional                                                           | deve ser único na collection                                                         |
| `unique`                  | definição de cada índice           | rejeita keys duplicadas                                                              | build falha se já existirem duplicados; não está disponível para hashed indexes      |
| `partialFilterExpression` | definição de cada índice           | só cria entries para documentos que satisfazem o filtro                              | a query precisa de implicar o filtro para o planner usar o índice sem perder dados   |
| `sparse`                  | definição de cada índice           | omite documentos sem o field indexado                                                | altera cobertura do conjunto; partial é normalmente mais explícito                   |
| `expireAfterSeconds`      | definição de cada índice           | configura TTL a partir do BSON Date indexado                                         | eliminação assíncrona; existem restrições de tipo e de índice                        |
| `hidden`                  | definição de cada índice           | mantém o índice atualizado, mas esconde-o do query planner                           | continua a ocupar espaço e a acrescentar custo aos writes                            |
| `collation`               | definição de cada índice           | define regras linguísticas para comparar strings                                     | query e índice precisam de collation compatível para uso eficiente                   |
| `wildcardProjection`      | definição de wildcard index        | limita/inclui paths abrangidos                                                       | tem regras próprias de inclusão/exclusão                                              |
| `weights`                 | definição de text index clássico   | atribui pesos aos fields                                                             | não é scoring de MongoDB Search                                                      |
| `default_language`        | definição de text index clássico   | seleciona regras linguísticas                                                        | o nome da propriedade usa underscore                                                 |
| `commitQuorum`            | comando `createIndexes()` completo | controla quantos membros data-bearing concluem o build antes do commit               | não pertence a uma specification individual                                         |
| `maxTimeMS` / `comment`   | comando completo                   | limitam/identificam a operação administrativa                                        | não alteram as keys guardadas no índice                                              |

Em `createIndex(key, options)`, as propriedades do único índice aparecem no segundo argumento. Em `createIndexes(specifications, commandOptions)`, as propriedades específicas ficam normalmente dentro de cada specification; options operacionais do batch ficam no segundo argumento.

Nem todas as combinações são válidas. As propriedades devem ser escolhidas pela semântica necessária, não copiadas em bloco. A antiga option `background` ainda pode aparecer em tipos/compatibilidade, mas os index builds modernos usam um processo unificado; não deve ser a base de uma estratégia atual de deployment.

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

Neste exemplo, uma entry só existe quando `deletedAt` está ausente. Como `unique: true` atua apenas nas entries incluídas, a unicidade de `email` é imposta entre utilizadores ativos; documentos soft-deleted podem reutilizar email.

Uma query apenas por `{ email }` não pode assumir automaticamente este índice, porque devolver apenas documentos ativos alteraria o resultado se também existirem soft-deleted. Uma query como `{ email, deletedAt: { $exists: false } }` expressa a mesma condição e é candidata natural.

### Listar e remover

```javascript
const indexes = await collection.listIndexes().toArray();
const dropResult = await collection.dropIndex("status_createdAt_desc");
```

`listIndexes()` devolve um cursor porque uma collection pode ter várias especificações; `toArray()` materializa-as. `dropIndex()` aceita aqui o nome criado anteriormente. Remover o índice elimina a estrutura auxiliar, não os documentos.

Remover em produção requer confirmar queries dependentes e ter plano de rollback. Um hidden index permite testar a sua ausência no planner antes de o apagar; contudo, continua a ser mantido durante os writes.

### Explain no driver

```javascript
const plan = await collection
    .find(filter, { projection })
    .sort(sort)
    .explain("executionStats");
```

`filter`, `projection` e `sort` são objetos definidos pela aplicação. A ordem encadeada constrói um cursor; `explain("executionStats")` executa a análise e devolve o plano em vez dos documentos normais.

Modos comuns:

- `"queryPlanner"`: mostra a escolha do plano sem estatísticas completas de execução;
- `"executionStats"`: executa o winning plan e inclui contadores;
- `"allPlansExecution"`: acrescenta informação de trabalho dos planos candidatos, com maior custo de diagnóstico.

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
- Relação entre shard key, supporting index e routing.

---

## Resumo

Índices B-tree trocam storage e custo de write por menos trabalho em reads. Compound indexes dependem da ordem, dos prefixos e da direção. ESR é uma boa heurística, não uma lei. Arrays produzem multikey indexes e restrições próprias. Covered queries evitam document fetch. `explain("executionStats")` é a prova: comparar plano e trabalho com resultados. Criar apenas índices que suportam constraints ou access patterns reais.

Fontes oficiais: [indexes](https://www.mongodb.com/docs/manual/indexes/), [`createIndex()` e respetivas options](https://www.mongodb.com/docs/manual/reference/method/db.collection.createIndex/), [quick reference do Node.js Driver](https://www.mongodb.com/docs/drivers/node/current/reference/quick-reference/), [index types](https://www.mongodb.com/docs/manual/core/indexes/index-types/), [compound indexes](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-compound/), [multikey](https://www.mongodb.com/docs/manual/core/indexes/index-types/index-multikey/), [ESR](https://www.mongodb.com/docs/manual/tutorial/equality-sort-range-guideline/) e [explain](https://www.mongodb.com/docs/manual/reference/explain-results/).

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

> **Ligação entre capítulos:** access patterns e arrays vêm dos capítulos 02 e 05; sort/projection do 07; shard keys e respetivos índices são aprofundados no 10; stages iniciais de aggregation no 11.

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
