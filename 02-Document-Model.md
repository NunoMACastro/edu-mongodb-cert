# Modelo documental, BSON e desenho de schema

## Objetivos do capítulo

- Compreender documentos BSON, collections e o campo `_id`.
- Escolher entre embedding e referencing com base nos access patterns.
- Distinguir flexibilidade de schema de ausência de regras.
- Usar dot notation, arrays e schema validation.
- Reconhecer os principais tipos BSON e o comportamento de `ObjectId`.

---

## Conceitos Fundamentais

### Vocabulário do modelo documental

Antes de decidir como relacionar dados, é necessário distinguir as unidades fundamentais do MongoDB:

| Conceito       | Definição                                                                                                    |
| -------------- | ------------------------------------------------------------------------------------------------------------ |
| Database       | contentor lógico de collections; faz parte do namespace e da organização dos dados                          |
| Collection     | conjunto de documentos relacionado com um domínio, como `customers`, `orders` ou `products`                 |
| Document       | unidade BSON persistida numa collection, formada por pares field-value                                      |
| Field          | nome de uma propriedade do documento, como `status` ou `shippingAddress`                                    |
| Value          | valor associado a um field; pode ser scalar, subdocumento, array ou outro tipo BSON                         |
| Subdocument    | documento BSON embebido como valor de um field de outro documento                                           |
| Array          | sequência ordenada de valores, incluindo scalars, subdocumentos ou outros arrays                            |
| `_id`          | identificador único obrigatório de cada documento dentro da collection                                      |
| Document model | organização de documentos, subdocumentos, arrays e referências para suportar os access patterns             |

A hierarquia lógica pode ser representada assim:

```text
MongoDB deployment
└── database: shop
    ├── collection: customers
    │   └── document: { _id, name, email, ... }
    └── collection: orders
        └── document: { _id, customerId, items, ... }
```

Uma **collection** é o alvo de operações como `find()`, `insertOne()` e `createIndex()`. Ao contrário de uma tabela relacional rígida, os seus documentos não têm obrigatoriamente os mesmos fields. Esta flexibilidade não elimina o schema: a aplicação, os índices e um validator podem estabelecer tipos, fields obrigatórios e outras invariantes.

### Documento como unidade de modelação

Um documento MongoDB agrupa campos relacionados numa estrutura BSON. É simultaneamente a unidade normal de leitura/escrita e a fronteira de atomicidade de um write single-document. Ao contrário de uma linha relacional estritamente tabular, pode conter subdocumentos e arrays com estruturas próprias.

Todos os documentos de uma collection têm um `_id` único. Se a aplicação não o fornecer, o driver gera normalmente um `ObjectId`. Existe um índice unique sobre `_id`.

Os fields podem ser scalars ou estruturas compostas:

- `status: "paid"` é um field com valor String;
- `shippingAddress: { city: "Lisboa", country: "PT" }` contém um subdocumento;
- `tags: ["mongodb", "nodejs"]` contém um array de scalars;
- `items: [{ productId, quantity }]` contém um array de subdocumentos.

Subdocumentos e arrays pertencem ao mesmo documento BSON pai. Não são linhas ou documentos independentes noutra collection. Esta distinção determina atomicidade, crescimento, índices multikey e a forma das queries.

A decisão central não é “normalizar sempre”, mas organizar os documentos para os **access patterns**, a cardinalidade, o lifecycle dos dados e os limites de consistência.

### O que é embedding?

**Embedding** consiste em guardar dados relacionados dentro do mesmo documento, através de um subdocumento ou de um array. Os dados embebidos fazem parte da mesma unidade BSON e são lidos, escritos e limitados juntamente com o documento pai.

Neste exemplo, a morada e os items estão embebidos na encomenda:

```javascript
const embeddedOrder = {
    _id: orderId,
    status: "pending",
    shippingAddress: {
        street: "Rua do Ouro, 10",
        city: "Lisboa",
        country: "PT",
    },
    items: [
        {
            productId,
            nameSnapshot: "Teclado mecânico",
            quantity: 1,
        },
    ],
};
```

Embedding é especialmente adequado quando:

- os dados são normalmente lidos em conjunto;
- são criados, atualizados ou eliminados com o documento pai;
- a relação é “contains” ou um-para-poucos;
- a cardinalidade e o crescimento são limitados;
- é valioso atualizar a estrutura numa única operação atómica.

Embedding pode duplicar deliberadamente informação. Por exemplo, `nameSnapshot` preserva o nome do produto no momento da compra. Esta duplicação é uma decisão de domínio, não necessariamente um erro de normalização.

Os principais custos são o crescimento do documento, a atualização de cópias duplicadas, a criação de mais index keys em arrays e o limite BSON de 16 MiB.

### O que é referencing?

**Referencing** consiste em guardar documentos relacionados separadamente e manter num deles o `_id` do outro. A referência é um valor BSON normal — frequentemente um `ObjectId` — e não transforma automaticamente os documentos numa única unidade.

```javascript
const customer = {
    _id: customerId,
    name: "Ana Martins",
    email: "ana@example.com",
};

const referencedOrder = {
    _id: orderId,
    customerId: customer._id,
    status: "pending",
};
```

MongoDB não carrega `customer` automaticamente quando lê `referencedOrder`. A aplicação pode:

- executar uma query adicional sobre `customers`;
- obter vários documentos antecipadamente e relacioná-los em memória;
- usar `$lookup` numa aggregation pipeline quando a operação o justificar;
- manter alguns fields duplicados como snapshot ou resumo, criando um modelo híbrido.

Uma referência também não cria, por defeito, uma foreign key com integridade referencial automática. A aplicação e o desenho das operações devem evitar referências inexistentes ou obsoletas; quando uma invariante abrange vários documentos, pode ser necessária uma transação.

Referencing é especialmente adequado quando:

- a entidade é partilhada por muitos documentos;
- tem lifecycle independente;
- muda frequentemente e deve existir uma única fonte atual;
- a cardinalidade pode crescer sem limite prático;
- os dados relacionados não são normalmente lidos em conjunto.

### Embedding versus referencing

| Critério               | Embedding                  | Referencing                                 |
| ---------------------- | -------------------------- | ------------------------------------------- |
| Leitura conjunta       | uma operação               | pode exigir operação adicional ou `$lookup` |
| Atomicidade            | single-document            | transação para invariantes multi-documento  |
| Duplicação             | possível                   | menor                                       |
| Crescimento            | limitado pelo documento    | entidades crescem separadamente             |
| Relação                | “contains”, um-para-poucos | muitos-para-muitos, alta cardinalidade      |
| Atualização partilhada | várias cópias              | uma fonte                                   |

Esta comparação não é uma regra binária. Um modelo pode referenciar a entidade atual e, ao mesmo tempo, embebedar um snapshot dos fields que precisam de preservar contexto histórico.

Usar embedding quando os dados são lidos/alterados juntos, têm cardinalidade limitada e pertencem ao ciclo de vida do pai. Usar references quando a entidade é partilhada, cresce sem limite prático, muda independentemente ou seria duplicada de forma dispendiosa.

Evitar arrays sem limite, como todos os eventos de uma conta num único documento. Mesmo antes do limite BSON de 16 MiB, documentos crescentes aumentam movement, working set e custo de updates.

### Flexible schema não significa schemaless

Documentos de uma collection podem ter formas diferentes, útil para evolução incremental e dados polimórficos. A aplicação continua a precisar de invariantes. JSON Schema validation no servidor impede writes inválidos independentemente do cliente. Validação na aplicação melhora UX, mas não substitui a fronteira de integridade do servidor.

### Tipos BSON de maior relevância

| Tipo                               | Uso                                       |
| ---------------------------------- | ----------------------------------------- |
| String                             | texto UTF-8                               |
| Object                             | subdocumento                              |
| Array                              | sequência ordenada                        |
| ObjectId                           | identificador BSON de 12 bytes            |
| Boolean                            | verdadeiro/falso                          |
| Date                               | instante UTC em milissegundos desde epoch |
| Int32 / Long / Double / Decimal128 | semânticas numéricas distintas            |
| Null                               | valor nulo explícito                      |
| Binary                             | bytes e subtipos                          |
| Regular Expression                 | padrão                                    |
| Timestamp                          | tipo interno BSON, distinto de Date       |

Para dinheiro, `Decimal128` evita erros binários de `double`, mas a aplicação deve converter explicitamente e manter regras de escala/arredondamento.

### ObjectId

Um `ObjectId` tem 12 bytes: timestamp de 4 bytes, valor aleatório de 5 bytes por processo e contador de 3 bytes. O timestamp permite uma ordenação aproximadamente cronológica, mas:

- não substitui um campo `createdAt` com semântica de negócio;
- não é totalmente aleatório;
- não deve ser tratado como segredo;
- uma string hexadecimal de 24 caracteres não é igual ao `ObjectId` BSON correspondente.

> **Importante para o exame:** MongoDB compara valor e tipo. `ObjectId("...")` e a string `"..."` não são iguais; além disso, qualquer documento BSON, incluindo documentos devolvidos por uma pipeline, está sujeito ao limite de 16 MiB.

---

## Funcionamento Interno

### Serialização BSON

O driver percorre o objeto JavaScript, mapeia valores para tipos BSON e codifica o tamanho e os elementos. O servidor persiste BSON e os índices armazenam representações ordenadas de chaves. A ordem dos campos é preservada e pode importar em comparação de documentos, embora filtros comuns acedam por nome.

O limite de 16 MiB aplica-se a cada documento BSON. Para ficheiros maiores, GridFS divide o conteúdo em chunks; não é uma forma de contornar um schema de domínio mal modelado.

### Atomicidade e concorrência

Um update single-document adquire a coordenação necessária e aplica a alteração atomicamente. Se duas operações condicionais competirem, incluir o estado esperado no filtro transforma a operação num compare-and-set:

```javascript
const result = await orders.updateOne(
    { _id: orderId, status: "pending" },
    { $set: { status: "paid", paidAt: new Date() } },
);
```

Só uma operação que ainda encontre `status: "pending"` modifica o documento. `matchedCount === 0` indica conflito ou ausência.

### Multikey

Quando um índice inclui um array, MongoDB cria entradas de índice para os elementos e marca-o multikey. Isto suporta pesquisa por elementos, mas amplia o índice. Um compound multikey index não pode indexar mais de um campo array por documento, porque o produto cartesiano seria ambíguo/explosivo.

---

## Sintaxe

### Documento com embedding

```javascript
const order = {
    customerId: new ObjectId("64f000000000000000000001"),
    status: "pending",
    shippingAddress: {
        street: "Rua do Ouro, 10",
        city: "Lisboa",
        postalCode: "1100-060",
        country: "PT",
    },
    items: [
        {
            productId: new ObjectId("64f000000000000000000101"),
            nameSnapshot: "Teclado mecânico",
            quantity: 1,
            unitPrice: Decimal128.fromString("89.90"),
        },
    ],
    createdAt: new Date(),
};
```

Este objeto mostra três níveis do modelo:

- fields escalares no documento raiz, como `status` e `createdAt`;
- um subdocumento embedded em `shippingAddress`;
- um array de subdocumentos embedded em `items`.

`new ObjectId(...)`, `Decimal128.fromString(...)` e `new Date()` não são decoração: preservam tipos BSON diferentes de strings. O snapshot de nome e preço é duplicação intencional: uma encomenda histórica não deve mudar quando o catálogo muda.

### Dot notation

```javascript
const exampleFilters = [
    { "shippingAddress.country": "PT" },
    { "items.productId": productId },
    {
        items: {
            $elemMatch: {
                quantity: { $gte: 2 },
                unitPrice: { $lt: price },
            },
        },
    },
];
```

`$elemMatch` garante que as condições se aplicam ao mesmo elemento do array. Sem ele, condições em paths paralelos podem ser satisfeitas por elementos diferentes.

### Validator

A assinatura usada aqui é `db.createCollection(name, options)`. O primeiro argumento escolhe o nome; o segundo configura a collection. `validator` é uma das options e `$jsonSchema` é a expressão de validação, não um schema TypeScript nem uma validação executada no cliente.

```javascript
await db.createCollection("orders", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["customerId", "status", "items", "createdAt"],
            properties: {
                customerId: { bsonType: "objectId" },
                status: { enum: ["pending", "paid", "cancelled"] },
                items: {
                    bsonType: "array",
                    minItems: 1,
                    items: {
                        bsonType: "object",
                        required: ["productId", "quantity", "unitPrice"],
                        properties: {
                            productId: { bsonType: "objectId" },
                            quantity: { bsonType: "int", minimum: 1 },
                            unitPrice: { bsonType: "decimal", minimum: 0 },
                        },
                    },
                },
                createdAt: { bsonType: "date" },
            },
        },
    },
    validationLevel: "strict",
    validationAction: "error",
});
```

Leitura de fora para dentro:

- `bsonType: "object"` exige que o valor raiz seja um documento;
- `required` exige a presença dos fields listados, mas não define sozinho o tipo deles;
- `properties` associa cada field às suas regras;
- `enum` restringe `status` aos três valores indicados;
- `items.bsonType: "array"` exige um array e `minItems: 1` impede o array vazio;
- o `items` mais interior descreve **cada elemento** desse array;
- `quantity` tem de ser BSON Int32 e pelo menos 1;
- `unitPrice` tem de ser BSON Decimal128 e não negativo;
- `validationLevel: "strict"` aplica validação a inserts e updates segundo a política strict;
- `validationAction: "error"` rejeita violações, enquanto `"warn"` apenas as regista.

O validator protege a database mesmo quando outro cliente não usa a validação da aplicação. Não substitui regras de autorização, relações entre collections nem todas as invariantes de negócio.

---

## Exemplos

### Exemplo 1 — inserir tipos BSON deliberados

```javascript
import { Decimal128, MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const products = client.db("certification_lab").collection("products");
    const product = {
        _id: new ObjectId(),
        sku: "KB-PT-001",
        name: "Teclado mecânico",
        price: Decimal128.fromString("89.90"),
        tags: ["keyboards", "mechanical"],
        dimensions: { widthMm: 440, depthMm: 135 },
        active: true,
        createdAt: new Date(),
    };

    const result = await products.insertOne(product);
    console.log(result.insertedId);
} finally {
    await client.close();
}
```

Resultado: documento com `ObjectId`, `Decimal128` e Date BSON, não strings equivalentes.

### Exemplo 2 — query correta sobre array de subdocumentos

```javascript
import { Decimal128, MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const sales = client.db("sample_supplies").collection("sales");
    const filter = {
        items: {
            $elemMatch: {
                quantity: { $gte: 5 },
                price: { $gte: Decimal128.fromString("100.00") },
            },
        },
    };

    const sale = await sales.findOne(filter, {
        projection: { _id: 0, saleDate: 1, items: 1 },
    });

    console.dir(sale, { depth: null });
} finally {
    await client.close();
}
```

Resultado: uma venda em que pelo menos um mesmo item satisfaz quantidade e preço, ou `null`.

### Exemplo 3 — update condicional atómico

```javascript
import { MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);
const orderId = new ObjectId(process.env.ORDER_ID);

try {
    const orders = client.db("shop").collection("orders");
    const result = await orders.updateOne(
        { _id: orderId, status: "pending" },
        {
            $set: {
                status: "paid",
                paidAt: new Date(),
            },
        },
    );

    if (result.modifiedCount !== 1) {
        throw new Error("A encomenda não existe ou já não está pendente.");
    }
} finally {
    await client.close();
}
```

Resultado: exatamente uma transição válida ou um erro de domínio lançado pela aplicação.

---

## Explicação linha a linha

### Exemplo 1

1. Importa classes BSON explicitamente.
2. Cria um `ObjectId` antes do insert; também poderia deixar o driver gerar `_id`.
3. `Decimal128.fromString()` evita passar primeiro por um Number binário.
4. Array e subdocumento são embedded.
5. `new Date()` é serializado como BSON Date.
6. `insertOne()` devolve `InsertOneResult`; `insertedId` confirma o identificador.

### Exemplo 2

1. O filtro é aplicado a `items`.
2. `$elemMatch` cria uma única fronteira de elemento.
3. `$gte` é inclusivo.
4. O valor decimal usa o mesmo tipo esperado no armazenamento.
5. A projection devolve apenas dados necessários.
6. `findOne()` devolve documento ou `null`.

### Exemplo 3

1. Converte a string externa para `ObjectId`; sem conversão, o tipo não corresponde.
2. O filtro inclui identidade e estado esperado.
3. `$set` altera só os campos indicados, preservando o resto.
4. O write é atómico no documento.
5. `modifiedCount` valida a transição; não basta ignorar o result object.

---

## Casos Reais

- **Encomenda:** embebe linhas com snapshots; referencia cliente/produto por identidade.
- **Blog:** embebe poucos autores resumidos, mas guarda comentários ilimitados noutra collection.
- **Streaming:** embebe metadados pequenos; eventos de visualização são documentos separados.
- **Reservas:** filtro condicional impede transições concorrentes inválidas.
- **Pagamentos:** `Decimal128` para montantes e currency separada; nunca Number sem política.
- **IoT:** bucketing agrupa séries temporais sem criar arrays ilimitados arbitrários.

---

## Performance

Embedding elimina joins e round trips e permite writes atómicos, mas documentos grandes consomem mais rede e cache. Referencing reduz duplicação, mas pode provocar N+1 queries ou pipelines `$lookup`. A melhor escolha resulta dos access patterns medidos.

Atualizar um campo indexado exige também atualizar entradas de índice. Arrays grandes produzem muitos index keys. Projeção de subcampos reduz payload, mas o servidor pode ainda ler o documento completo se a query não estiver coberta.

Um `ObjectId` crescente no tempo tende a boa localidade no índice `_id`, embora não seja estritamente ordenado globalmente entre todos os processos. Identificadores totalmente aleatórios podem aumentar dispersão de páginas; a escolha deve respeitar requisitos, não micro-otimizações prematuras.

---

## Armadilhas comuns

- **Guardar ObjectId como string:** a comparação é type-sensitive.
- **Usar double para dinheiro:** introduz representação binária inexata.
- **Arrays ilimitados:** aproximam o limite de 16 MiB e degradam updates.
- **Normalizar por hábito relacional:** multiplica queries sem benefício.
- **Embebedar entidade muito partilhada e mutável:** exige fan-out de updates.
- **Confundir flexible schema com ausência de validação:** dados inválidos tornam-se dívida.
- **Usar duas condições de array sem `$elemMatch`:** podem corresponder a elementos diferentes.
- **Inferir data de negócio a partir de ObjectId:** usar `createdAt` explícito.
- **Expor ObjectId como autorização:** conhecer um id não concede acesso.
- **Substituir o documento para mudar um campo:** pode apagar campos omitidos.

---

## O que costuma aparecer no exame

- Embedding versus referencing.
- Benefícios da atomicidade single-document.
- Limite de documento BSON.
- Tipos BSON versus tipos JSON.
- Estrutura e propriedades de `ObjectId`.
- Dot notation para subdocumentos.
- Semântica de arrays e `$elemMatch`.
- Razão do índice unique em `_id`.
- Flexible schema e schema validation.
- Relação entre arrays e multikey index.

---

## Resumo

Modelar em MongoDB é agrupar dados pelos access patterns e invariantes. Embedding favorece leitura conjunta e atomicidade; references favorecem entidades partilhadas, independentes ou sem limite. BSON preserva tipos ricos, e tipo faz parte da igualdade. `ObjectId` é um identificador de 12 bytes, não um segredo nem uma data de negócio. Validar no servidor protege todas as aplicações. Evitar arrays ilimitados, números inadequados e queries de arrays sem `$elemMatch`.

Fontes oficiais: [data modeling](https://www.mongodb.com/docs/manual/data-modeling/), [embedding](https://www.mongodb.com/docs/manual/data-modeling/embedding/), [references](https://www.mongodb.com/docs/manual/data-modeling/referencing/), [BSON types](https://www.mongodb.com/docs/manual/reference/bson-types/), [ObjectId](https://www.mongodb.com/docs/manual/reference/bson-types/#objectid), [schema validation](https://www.mongodb.com/docs/manual/core/schema-validation/) e [document limits](https://www.mongodb.com/docs/manual/reference/limits/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — A escolha entre embedding e referencing parte dos access patterns e da fronteira de atomicidade, não de uma regra universal. Dados lidos e alterados em conjunto, com crescimento limitado, favorecem embedding; dados partilhados, independentes ou sem limite favorecem references.

> ### ARMADILHA
>
> `new ObjectId("...")` e `"..."` podem mostrar os mesmos 24 caracteres, mas têm tipos BSON diferentes e não são iguais numa query. Converter o valor errado ou persistir IDs como tipos inconsistentes produz “zero resultados” sem erro de sintaxe.

> ### DICA DE MEMORIZAÇÃO
>
> **Juntos e limitados: embed. Partilhados ou ilimitados: reference.** Depois confirma sempre o access pattern real.

> ### COMPARAÇÃO
>
> A decisão de modelação troca coesão por independência.

| Critério         | Embedding                      | Referencing                      |
| ---------------- | ------------------------------ | -------------------------------- |
| leitura conjunta | uma leitura natural            | pode exigir nova query/`$lookup` |
| atomicidade      | single-document                | pode exigir transação            |
| crescimento      | deve ser controlado            | suporta conjuntos independentes  |
| partilha         | duplica dados se muitos owners | entidade comum reutilizável      |
| lifecycle        | normalmente conjunto           | pode evoluir separadamente       |

> **Ligação entre capítulos:** `$elemMatch` é usado no capítulo 05; updates de arrays no 06; multikey indexes no 09; transações no 12.

### Fluxograma: embedding ou referencing?

```text
Os dados são lidos e alterados juntos?
  |-- não --> Referencing tende a ser mais natural
  `-- sim --> O conjunto cresce sem limite ou é muito partilhado?
                |-- sim --> Referencing ou modelo híbrido
                `-- não --> A atomicidade conjunta é valiosa?
                              |-- sim --> Embedding
                              `-- não --> comparar custo e lifecycle
```

### Mini desafio

Uma encomenda precisa de preservar o nome e preço vendidos, mesmo que o produto mude depois. Decide que dados embebes, que identificador podes referenciar e que trade-off aceitas. Não procures uma resposta baseada apenas em “evitar duplicação”.

---

## Resumo Rápido

- Modelar começa nos access patterns e invariantes.
- Embedding favorece leitura e atomicidade conjunta.
- Referencing favorece partilha, independência e crescimento.
- Valor e tipo BSON participam na comparação.
- `ObjectId` é um tipo de 12 bytes, não uma string nem um segredo.
- Documento BSON tem limite de 16 MiB; arrays ilimitados são risco.

---

## Checklist

- [ ] Decido embedding/referencing por access pattern.
- [ ] Identifico a fronteira de atomicidade do modelo.
- [ ] Evito arrays com crescimento ilimitado.
- [ ] Distingo `ObjectId` de string hexadecimal.
- [ ] Escolho `Date`, `Decimal128` e outros tipos conscientemente.
- [ ] Sei para que serve schema validation.
- [ ] Reconheço duplicação deliberada e respetivo custo.
- [ ] Consigo justificar um modelo híbrido.

---

## Perguntas para confirmar conhecimentos

1. Que access patterns favorecem embedding?
2. Que características favorecem referencing?
3. Porque embedding pode reduzir a necessidade de transações?
4. Que risco introduz um array que cresce sem limite?
5. Porque uma string hexadecimal não corresponde a um `ObjectId` numa query?
6. Que informação aproximada pode ser extraída de um `ObjectId` e que uso não deve ser feito dela?
7. Quando a duplicação de dados pode ser uma decisão correta?
8. Que diferença existe entre schema flexível e ausência de validação?
9. Porque dinheiro pode exigir `Decimal128` em vez de `number`?
10. Como modelarias uma relação que é lida em conjunto, mas contém uma lista potencialmente ilimitada?
