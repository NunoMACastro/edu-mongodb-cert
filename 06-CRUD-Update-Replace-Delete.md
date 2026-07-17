# CRUD: update, replace e delete

## Objetivos do capítulo

- Distinguir update parcial de substituição integral.
- Usar update operators e updates sobre arrays.
- Interpretar `matchedCount`, `modifiedCount`, `upsertedId` e `deletedCount`.
- Aplicar upsert sem races desnecessárias.
- Evitar updates/deletes massivos acidentais.

---

## Conceitos Fundamentais

### update versus replace

| Operação                 | Segundo argumento      |    Preserva campos omitidos? | Pode alterar `_id`? |
| ------------------------ | ---------------------- | ---------------------------: | ------------------: |
| `updateOne`/`updateMany` | operadores ou pipeline | sim, salvo remoção explícita |                 não |
| `replaceOne`             | replacement document   |                          não |                 não |

`$set` altera paths selecionados. `replaceOne` substitui todo o documento correspondente, preservando `_id` se omitido no replacement. Não usar replacement para “editar dois campos” a menos que a substituição total seja intencional.

### Operadores de update

| Operador        | Efeito                                |
| --------------- | ------------------------------------- |
| `$set`          | atribui/cria campo                    |
| `$unset`        | remove campo                          |
| `$inc`          | incrementa valor numérico             |
| `$mul`          | multiplica                            |
| `$min` / `$max` | altera apenas se menor/maior          |
| `$rename`       | muda nome do campo                    |
| `$currentDate`  | define data/timestamp atual           |
| `$push`         | acrescenta ao array, aceita modifiers |
| `$addToSet`     | acrescenta se valor exato não existe  |
| `$pull`         | remove elementos correspondentes      |
| `$pop`          | remove primeiro/último                |

`$addToSet` evita duplicados por igualdade BSON, mas não transforma retroativamente um array com duplicados num set. Em subdocumentos, ordem de campos pode afetar igualdade.

### Um versus muitos

`updateOne` e `deleteOne` afetam no máximo um match. Se vários documentos correspondem e não existe sort aplicável/identidade única, qual é escolhido não deve sustentar uma regra de negócio. `updateMany`/`deleteMany` afetam todos os matches e exigem filtros cuidadosamente validados.

### Upsert

Com `upsert: true`, se nenhum documento corresponder, MongoDB constrói e insere um documento a partir do filtro de igualdade e do update. `$setOnInsert` aplica campos apenas no ramo insert.

Upsert deve ter um unique index alinhado com a chave lógica. Sem ele, upserts concorrentes podem criar duplicados.

### Delete

Delete remove documentos, não os campos. Para soft delete, atualizar `deletedAt`/status e garantir que todas as queries respeitam a política. Soft delete melhora auditabilidade mas aumenta complexidade, índices e risco de “ressuscitar” dados.

> **Importante para o exame:** `updateOne()` aplica update operators ou uma update pipeline; `replaceOne()` substitui o documento, preservando o `_id`. `matchedCount: 1` com `modifiedCount: 0` pode significar que o valor final já era igual, não que o filtro falhou.

---

## Funcionamento Interno

O servidor usa o filtro para localizar matches. Cada documento é modificado atomicamente. Operadores produzem uma nova versão lógica do documento; o storage engine atualiza dados e todas as index keys alteradas.

`matchedCount` conta documentos que corresponderam, mesmo que o novo valor seja igual. `modifiedCount` conta modificações efetivas e pode ser 0 quando já estava no estado desejado. Isto torna updates idempotentes possíveis:

```javascript
await users.updateOne({ _id: userId }, { $set: { locale: "pt-PT" } });
```

Executar duas vezes deixa o mesmo estado.

### Array positional operators

- `$`: primeiro elemento que corresponde à query.
- `$[]`: todos os elementos.
- `$[identifier]`: elementos que satisfazem `arrayFilters`.

`arrayFilters` é opção da operação, não parte do update document.

### Delete e índices

O delete encontra documentos pelo plano de query e remove entradas de todos os índices. Um `deleteMany({})` faz uma eliminação lógica de toda a collection, mas não equivale a `drop()`: a collection e os índices permanecem.

---

## Sintaxe

### Updates

```javascript
const result = await collection.updateOne(
    filter,
    { $set: changes, $currentDate: { updatedAt: true } },
    { upsert: false, arrayFilters, hint, writeConcern },
);

const manyResult = await collection.updateMany(filter, update, options);
```

`UpdateResult` contém `acknowledged`, `matchedCount`, `modifiedCount`, `upsertedCount` e `upsertedId`.

### Replace

```javascript
const result = await collection.replaceOne({ _id: documentId }, replacement, {
    upsert: false,
});
```

O replacement não pode conter update operators de topo.

### Delete

```javascript
const one = await collection.deleteOne(filter, { hint });
const many = await collection.deleteMany(filter, { hint });
```

`DeleteResult.deletedCount` indica quantos documentos foram eliminados.

### Update pipeline

```javascript
await collection.updateMany({ subtotal: { $exists: true } }, [
    {
        $set: {
            total: { $round: [{ $multiply: ["$subtotal", 1.23] }, 2] },
        },
    },
]);
```

Num update pipeline, `$set` é aggregation stage, não o update operator clássico, embora o nome coincida.

---

## Exemplos

### Exemplo 1 — update parcial e contadores

```javascript
/**
 * @file Atualiza metadados sem substituir o filme inteiro.
 */
import { MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);
const movieId = new ObjectId(process.env.MOVIE_ID);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const result = await movies.updateOne(
        { _id: movieId },
        {
            $set: { "viewerSettings.featured": true },
            $currentDate: { "viewerSettings.updatedAt": true },
        },
    );

    console.log({
        matched: result.matchedCount,
        modified: result.modifiedCount,
    });
} finally {
    await client.close();
}
```

Resultado: match 1 se o filme existe; modified pode ser 0 ou 1 conforme o estado.

### Exemplo 2 — upsert com `$setOnInsert`

```javascript
/**
 * @file Regista a preferência atual ou cria-a de forma concorrente.
 */
import { MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);
const userId = new ObjectId(process.env.USER_ID);

try {
    const preferences = client.db("app").collection("preferences");
    await preferences.createIndex({ userId: 1 }, { unique: true });

    const result = await preferences.updateOne(
        { userId },
        {
            $set: {
                locale: "pt-PT",
                theme: "dark",
                updatedAt: new Date(),
            },
            $setOnInsert: {
                createdAt: new Date(),
            },
        },
        { upsert: true },
    );

    console.log({
        matched: result.matchedCount,
        upsertedId: result.upsertedId,
    });
} finally {
    await client.close();
}
```

Resultado: update quando existe; insert e `upsertedId` quando não existe.

### Exemplo 3 — atualizar elementos de array filtrados

```javascript
/**
 * @file Aplica desconto apenas a linhas elegíveis de uma encomenda.
 */
import { Decimal128, MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);
const orderId = new ObjectId(process.env.ORDER_ID);

try {
    const orders = client.db("shop").collection("orders");
    const result = await orders.updateOne(
        { _id: orderId, status: "draft" },
        {
            $set: {
                "items.$[item].discount": Decimal128.fromString("0.10"),
            },
        },
        {
            arrayFilters: [
                {
                    "item.category": "books",
                    "item.quantity": { $gte: 2 },
                },
            ],
        },
    );

    console.log(result.modifiedCount);
} finally {
    await client.close();
}
```

Resultado: todas as linhas do mesmo documento que passam `arrayFilters` recebem desconto.

### Exemplo 4 — replacement deliberado

```javascript
/**
 * @file Substitui uma configuração que é tratada como documento completo.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const settings = client.db("app").collection("tenantSettings");
    const replacement = {
        tenantKey: "tenant-pt",
        locale: "pt-PT",
        features: {
            search: true,
            recommendations: false,
        },
        version: 4,
        updatedAt: new Date(),
    };

    const result = await settings.replaceOne(
        { tenantKey: "tenant-pt", version: 3 },
        replacement,
    );

    if (result.modifiedCount !== 1) {
        throw new Error("Conflito de versão ou configuração inexistente.");
    }
} finally {
    await client.close();
}
```

Resultado: substituição integral apenas se a versão atual for 3.

### Exemplo 5 — delete protegido por estado

```javascript
/**
 * @file Elimina uma sessão apenas se já tiver expirado.
 */
import { MongoClient, ObjectId } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);
const sessionId = new ObjectId(process.env.SESSION_ID);

try {
    const sessions = client.db("app").collection("sessions");
    const result = await sessions.deleteOne({
        _id: sessionId,
        expiresAt: { $lt: new Date() },
    });

    console.log({ deleted: result.deletedCount });
} finally {
    await client.close();
}
```

Resultado: `deletedCount` 1 apenas para uma sessão existente e expirada.

---

## Explicação linha a linha

### Exemplo 1

1. Converte id string em `ObjectId`.
2. `$set` cria/atualiza um path nested.
3. `$currentDate` usa o relógio do servidor para a data.
4. `matchedCount` responde “foi encontrado?”.
5. `modifiedCount` responde “houve mudança efetiva?”.

### Exemplo 2

1. O unique index alinha unicidade com `userId`.
2. O filtro identifica a preferência.
3. `$set` aplica-se em update e insert.
4. `$setOnInsert` só se aplica no insert.
5. `upsertedId` é preenchido apenas quando houve upsert insert.

### Exemplo 3

1. O filtro limita o documento e o estado.
2. `$[item]` referencia o identificador filtrado.
3. `arrayFilters` seleciona linhas books com quantidade ≥ 2.
4. Todos os elementos elegíveis do documento são atualizados atomicamente.

### Exemplo 4

1. O replacement contém o estado completo desejado.
2. O filtro inclui versão para optimistic concurrency.
3. Campos antigos omitidos desaparecem.
4. A falha de `modifiedCount` é tratada como conflito.

### Exemplo 5

1. O filtro inclui identidade e condição de expiração.
2. `deleteOne` elimina no máximo um documento.
3. `deletedCount` 0 significa ausência ou condição não satisfeita.
4. O filtro evita uma leitura seguida de delete vulnerável a race.

---

## Casos Reais

- **Perfil:** `$set` altera preferências sem perder campos.
- **Contador:** `$inc` evita read-modify-write no cliente.
- **Carrinho:** `$[item]` atualiza linhas elegíveis.
- **Configuração versionada:** replacement + version implementa optimistic concurrency.
- **Sessões:** delete condicional por expiração.
- **Retenção:** `deleteMany` por data, idealmente em batches/política TTL quando aplicável.

---

## Performance

Updates que não alteram campos indexados evitam reescrever essas index keys, mas ainda têm custo de localizar e persistir o documento. `$inc` server-side reduz round trips e conflitos comparado com read + write.

`updateMany` e `deleteMany` sobre muitos documentos podem gerar I/O, replication lag e pressão de oplog. Processar em janelas quando o risco operacional o exige. Um TTL index é melhor para expiração automática simples do que um cron que faz scans completos.

Replacement de documentos grandes envia e escreve mais dados do que updates parciais. Contudo, não escolher `$set` se a regra é realmente substituir a configuração inteira: correção semântica vem primeiro.

---

## Armadilhas comuns

- **Confundir `$set` com `replaceOne`:** replacement apaga campos omitidos.
- **Tentar alterar `_id`:** é imutável.
- **Interpretar `modifiedCount: 0` como ausência:** pode já ter o valor.
- **Upsert sem unique index:** pode duplicar sob concorrência.
- **`updateMany({}, ...)` ou `deleteMany({})` acidental:** validar filtros.
- **Read-modify-write para incrementar:** usar `$inc`.
- **`$push` quando se quer evitar duplicados:** considerar `$addToSet`.
- **Assumir que `$addToSet` limpa duplicados existentes.**
- **Colocar `arrayFilters` dentro do update document:** é opção.
- **Confundir update pipeline `$set` com update operator `$set`.**
- **Soft delete sem filtrar todas as reads:** dados apagados reaparecem.

---

## O que costuma aparecer no exame

- `updateOne` versus `updateMany`.
- `$set` versus `replaceOne`.
- `matchedCount` versus `modifiedCount`.
- `upsert` e `$setOnInsert`.
- `$push` versus `$addToSet`.
- `$inc` e updates atómicos.
- `$`, `$[]` e `$[identifier]`.
- `arrayFilters`.
- `deleteOne` versus `deleteMany`.
- Resultado de delete e filtros vazios.

---

## Resumo

Updates parciais preservam campos; replacement substitui o documento. Operadores como `$set`, `$inc` e operadores de array expressam a mudança no servidor e mantêm atomicidade single-document. `matchedCount` e `modifiedCount` respondem a perguntas diferentes. Upsert elimina uma leitura prévia, mas precisa de unique index. Deletes devem ter filtros explícitos e resultados verificados.

Fontes oficiais: [updates no driver](https://www.mongodb.com/docs/drivers/node/current/crud/update/), [update operators](https://www.mongodb.com/docs/manual/reference/mql/update/), [arrays em updates](https://www.mongodb.com/docs/manual/reference/operator/update/positional-filtered/), [replaceOne](https://www.mongodb.com/docs/manual/reference/method/db.collection.replaceOne/) e [delete no driver](https://www.mongodb.com/docs/drivers/node/current/crud/delete/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — `updateOne()` altera campos através de operators/pipeline e devolve `UpdateResult`. `replaceOne()` substitui o documento inteiro, salvo `_id`. `findOneAndUpdate()` é escolhido quando é necessário obter o documento numa operação read+write atómica.

> ### ARMADILHA
>
> `matchedCount: 1` com `modifiedCount: 0` não significa necessariamente falha. O filtro encontrou um documento, mas o estado final já podia ser igual. Inversamente, validar apenas `acknowledged` não prova que a regra de negócio alterou um documento.

> ### DICA DE MEMORIZAÇÃO
>
> **`$set` faz patch; `replaceOne` troca a ficha. `$push` aceita repetidos; `$addToSet` evita novo igual.**

> ### COMPARAÇÃO
>
> O resultado pretendido separa métodos com nomes semelhantes.

| Método               | Efeito                   | Retorno          | Uso típico                   |
| -------------------- | ------------------------ | ---------------- | ---------------------------- |
| `updateOne()`        | altera primeiro match    | `UpdateResult`   | patch/contador               |
| `updateMany()`       | altera todos os matches  | `UpdateResult`   | mudança em massa deliberada  |
| `replaceOne()`       | substitui primeiro match | `UpdateResult`   | replacement completo         |
| `findOneAndUpdate()` | altera e lê atomicamente | documento/`null` | devolver estado before/after |
| `deleteOne()`        | remove no máximo um      | `DeleteResult`   | identidade/filtro único      |
| `deleteMany()`       | remove todos os matches  | `DeleteResult`   | limpeza explícita            |

> **Ligação entre capítulos:** filtros e arrays vêm do capítulo 05; retornos atuais no capítulo 08; unique indexes para upsert no capítulo 09.

### Fluxograma de escolha

```text
Preciso de modificar ou substituir?
  |-- substituir documento completo --> replaceOne
  `-- modificar campos --> um match?
                          |-- não --> updateMany
                          `-- sim --> preciso do documento final?
                                      |-- sim --> findOneAndUpdate
                                      `-- não --> updateOne
```

### Mini desafio

Sem executar, determina que fields permanecem em cada caso e que método devolve o documento final:

```javascript
await users.updateOne({ _id }, { $set: { name: "Ana" } });
await users.replaceOne({ _id }, { name: "Ana" });
```

---

## Resumo Rápido

- `updateOne()` faz alteração parcial; `replaceOne()` substitui.
- Result object não é documento atualizado.
- `matchedCount` e `modifiedCount` medem factos diferentes.
- Upsert precisa de filtro e unique constraint bem pensados.
- `$push` pode duplicar; `$addToSet` evita nova igualdade exata.
- Deletes afetam matches, não fields, e exigem filtros deliberados.

---

## Checklist

- [ ] Distingo update parcial de replacement.
- [ ] Interpreto todos os campos de `UpdateResult`.
- [ ] Sei quando usar `findOneAndUpdate()`.
- [ ] Sei construir um upsert seguro.
- [ ] Distingo `$push` de `$addToSet`.
- [ ] Uso `$[]` e `$[id]` conscientemente.
- [ ] Valido `deletedCount`.
- [ ] Protejo operações `Many` contra filtros demasiado amplos.

---

## Perguntas para confirmar conhecimentos

1. Que diferença semântica existe entre `updateOne()` e `replaceOne()`?
2. Porque `updateOne()` não devolve o documento alterado?
3. O que significa `matchedCount: 1, modifiedCount: 0`?
4. Quando se deve usar `findOneAndUpdate()`?
5. Como `returnDocument: "after"` altera o retorno?
6. Que risco concorrente existe num check-then-insert em vez de upsert com unique index?
7. Que diferença existe entre `$push` e `$addToSet`?
8. Para que servem `$[]`, `$[identifier]` e `arrayFilters`?
9. Que validação deve ser feita depois de `deleteOne()`?
10. Porque `updateMany({})` e `deleteMany({})` merecem proteção adicional?
