# CRUD completo com o MongoDB Node.js Driver

## Objetivos do capítulo

- Organizar código CRUD com um cliente partilhado e APIs atuais.
- Tratar result objects, cursores e compound operations corretamente.
- Usar `bulkWrite()` quando operações diferentes pertencem ao mesmo batch.
- Distinguir erros de input, ausência, conflito e falha de infraestrutura.
- Aplicar práticas seguras de filtros, projection, timeouts e retries.

---

## Conceitos Fundamentais

### Mapeamento das APIs

| Intenção | Método | Resultado |
|---|---|---|
| criar um | `insertOne` | `InsertOneResult` |
| criar vários | `insertMany` | `InsertManyResult` |
| ler um | `findOne` | documento/`null` |
| ler vários | `find` | `FindCursor` |
| alterar campos | `updateOne/Many` | `UpdateResult` |
| substituir | `replaceOne` | `UpdateResult` |
| eliminar | `deleteOne/Many` | `DeleteResult` |
| ler + alterar atomicamente | `findOneAndUpdate` | documento/`null` por defeito |
| lote heterogéneo | `bulkWrite` | `BulkWriteResult` |

No driver atual, compound operations devolvem por defeito o documento diretamente porque `includeResultMetadata` é `false`. Com `includeResultMetadata: true` devolvem um `ModifyResult` com `value` e metadata. Exemplos antigos que usam sempre `result.value` podem estar errados para a versão atual.

### Result object não é o documento

`updateOne()` não devolve o documento atualizado. Se a aplicação precisa dele atomicamente, usar `findOneAndUpdate()` com `returnDocument: "after"`. Fazer update seguido de find abre uma janela concorrente.

### CRUD repository

Um repository pode centralizar acesso à collection e invariantes de persistência, mas não deve:

- criar/fechar `MongoClient` em cada método;
- aceitar filtros arbitrários diretamente de HTTP;
- esconder `matchedCount`/conflitos relevantes;
- transformar todo erro em “not found”.

Validação de input deve ocorrer antes de construir MQL. Chaves iniciadas por `$` provenientes do utilizador não devem ser espalhadas num filtro sem allowlist.

### Bulk operations

`bulkWrite(models, { ordered })` combina inserts, updates, replacements e deletes numa chamada. Reduz round trips e devolve contadores. Não é atomicidade global. Ordered para no primeiro erro; unordered tenta continuar operações possíveis.

### Compound operations

`findOneAndUpdate`, `findOneAndReplace` e `findOneAndDelete` são read+write atómicos sobre um documento. `returnDocument` aceita `"before"` (default em updates/replaces) ou `"after"`.

> **Importante para o exame:** nas compound operations do driver atual, `includeResultMetadata` é `false` por defeito: recebe-se documento ou `null`. Só com `true` se recebe um `ModifyResult` e se consulta `result.value`.

---

## Funcionamento Interno

O driver converte cada método num comando. Operações elegíveis podem receber session id e transaction number para retryable writes. O servidor garante que a repetição reconhecida não aplica o mesmo write duas vezes, dentro das regras de retryable writes.

### Cursor e backpressure

`FindCursor` guarda options e estado do batch. `for await...of` chama `getMore` conforme progride. `stream()` integra Node streams; async iteration é geralmente mais simples. Misturar `next()`, `toArray()` e async iteration no mesmo cursor leva a estado consumido inesperado.

### Bulk write

O driver agrupa models compatíveis em comandos/batches respeitando limites. `ordered: false` permite reordenamento/execução independente; não assumir que o índice do resultado representa ordem temporal.

### Erros

Erros importantes:

- `MongoServerError`: resposta do servidor, por exemplo duplicate key code 11000.
- `MongoNetworkError`: falha de rede.
- `MongoServerSelectionError`: não encontrou servidor elegível no prazo.
- `MongoBulkWriteError`: batch com erro e resultados parciais.
- erros de validação da aplicação: devem ser separados antes de chamar o driver.

Não basear lógica só em texto da mensagem. Usar classes/codes documentados e preservar causa.

---

## Sintaxe

### findOneAndUpdate atual

~~~javascript
const updatedDocument = await collection.findOneAndUpdate(
  filter,
  update,
  {
    returnDocument: "after",
    projection: { sensitiveField: 0 },
    upsert: false,
    includeResultMetadata: false
  }
);
~~~

`updatedDocument` é documento ou `null`. Com `includeResultMetadata: true`, o documento fica em `result.value`.

### bulkWrite

~~~javascript
const result = await collection.bulkWrite(
  [
    { insertOne: { document } },
    { updateOne: { filter, update, upsert: true } },
    { deleteOne: { filter } },
    { replaceOne: { filter, replacement } }
  ],
  { ordered: false }
);
~~~

### Hint, timeout e comment

~~~javascript
const cursor = collection.find(filter, {
  projection,
  hint: { status: 1, createdAt: -1 },
  maxTimeMS: 2_000,
  comment: "list-recent-active-orders"
});
~~~

`hint` força um índice e pode falhar se não existir. Usar apenas quando existe evidência e gestão coordenada do ciclo de vida do índice.

---

## Exemplos

### Exemplo 1 — repository com validação e optimistic concurrency

~~~javascript
/**
 * @file Repository de perfis com atualização concorrente por versão.
 */
import { ObjectId } from "mongodb";

/**
 * Cria operações de perfil sobre uma collection já configurada.
 *
 * @param {import("mongodb").Collection} profiles Collection partilhada.
 */
export function createProfileRepository(profiles) {
  return {
    /**
     * Altera o display name se a versão observada ainda for atual.
     */
    async updateDisplayName({ profileId, expectedVersion, displayName }) {
      if (!ObjectId.isValid(profileId)) {
        throw new TypeError("profileId inválido.");
      }

      const normalizedName = displayName.trim();

      if (normalizedName.length < 2 || normalizedName.length > 80) {
        throw new RangeError("displayName deve ter entre 2 e 80 caracteres.");
      }

      return profiles.findOneAndUpdate(
        {
          _id: new ObjectId(profileId),
          version: expectedVersion
        },
        {
          $set: {
            displayName: normalizedName,
            updatedAt: new Date()
          },
          $inc: { version: 1 }
        },
        {
          returnDocument: "after",
          projection: { _id: 1, displayName: 1, version: 1, updatedAt: 1 }
        }
      );
    }
  };
}
~~~

Resultado: perfil atualizado ou `null` quando id/versão não corresponde; não existe janela entre update e read.

### Exemplo 2 — paginação e cursor com cleanup

~~~javascript
/**
 * @file Exporta filmes progressivamente sem os carregar todos.
 */
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
  const movies = client.db("sample_mflix").collection("movies");
  const cursor = movies
    .find(
      { year: { $gte: 1990, $lt: 2000 } },
      { batchSize: 100, maxTimeMS: 10_000 }
    )
    .project({ _id: 0, title: 1, year: 1 })
    .sort({ year: 1, title: 1 });

  try {
    for await (const movie of cursor) {
      process.stdout.write(JSON.stringify(movie) + "\n");
    }
  } finally {
    await cursor.close();
  }
} finally {
  await client.close();
}
~~~

Resultado: JSON Lines por filme; memória limitada pelo batch e processamento.

### Exemplo 3 — bulkWrite heterogéneo

~~~javascript
/**
 * @file Sincroniza produtos por SKU numa única chamada bulk.
 */
import { Decimal128, MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
  const products = client.db("shop").collection("products");
  await products.createIndex({ sku: 1 }, { unique: true });

  const operations = [
    {
      updateOne: {
        filter: { sku: "BOOK-001" },
        update: {
          $set: {
            name: "MongoDB Study Guide",
            price: Decimal128.fromString("39.90"),
            active: true,
            updatedAt: new Date()
          },
          $setOnInsert: { createdAt: new Date() }
        },
        upsert: true
      }
    },
    {
      updateOne: {
        filter: { sku: "BOOK-002" },
        update: {
          $set: { active: false, updatedAt: new Date() }
        }
      }
    },
    {
      deleteOne: {
        filter: { sku: "TEMP-REMOVED", temporary: true }
      }
    }
  ];

  const result = await products.bulkWrite(operations, { ordered: false });
  console.log({
    inserted: result.insertedCount,
    matched: result.matchedCount,
    modified: result.modifiedCount,
    deleted: result.deletedCount,
    upserted: result.upsertedCount
  });
} finally {
  await client.close();
}
~~~

Resultado: contadores agregados das operações executadas; não há all-or-nothing global.

### Exemplo 4 — tratar duplicate key sem esconder outras falhas

~~~javascript
/**
 * @file Cria conta e converte apenas duplicate key num conflito de domínio.
 */
import { MongoClient, MongoServerError } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
  const accounts = client.db("app").collection("accounts");
  await accounts.createIndex({ email: 1 }, { unique: true });

  try {
    const result = await accounts.insertOne({
      email: "ana@example.com",
      status: "active",
      createdAt: new Date()
    });

    console.log(result.insertedId);
  } catch (error) {
    if (error instanceof MongoServerError && error.code === 11000) {
      console.error("Já existe uma conta com esse email.");
      process.exitCode = 2;
    } else {
      throw error;
    }
  }
} finally {
  await client.close();
}
~~~

Resultado: duplicate key é tratado como conflito; rede e outros erros continuam a propagar.

---

## Explicação linha a linha

### Exemplo 1

1. O repository recebe a collection; não possui o client.
2. Valida a forma de ObjectId e as regras do nome.
3. O filtro contém id e versão observada.
4. `$set` muda dados e `$inc` avança a versão atomicamente.
5. `returnDocument: "after"` devolve o novo estado.
6. `null` representa ausência ou conflito; a camada de serviço decide a resposta.

### Exemplo 2

1. `batchSize` regula o batch, não limita o total.
2. Projection e sort são definidos antes da iteração.
3. `for await` aplica backpressure natural.
4. Cada linha é serializada independentemente.
5. `cursor.close()` liberta recursos quando a iteração é interrompida.
6. O client fecha no fim do script.

### Exemplo 3

1. Unique SKU protege o upsert concorrente.
2. O primeiro model faz update-or-insert.
3. O segundo não faz upsert.
4. O delete tem duas condições defensivas.
5. `ordered: false` permite continuar após erro independente.
6. O result object agrega contadores.

### Exemplo 4

1. Unique index é a garantia, não uma leitura prévia.
2. O `try` interno cobre só o insert.
3. A classe e code 11000 identificam duplicate key.
4. Só esse erro é convertido em conflito.
5. O `throw` preserva falhas inesperadas.

---

## Casos Reais

- **HTTP PATCH:** `findOneAndUpdate` devolve o recurso final atomicamente.
- **Optimistic locking:** filtro por version evita lost updates.
- **Export:** cursor + JSON Lines mantém memória controlada.
- **Sincronização ERP:** `bulkWrite` reduz round trips.
- **Registo:** unique index + tratamento 11000 resolve concorrência.
- **Admin:** projection impede devolver campos sensíveis.

---

## Performance

Bulk operations reduzem round trips, mas um batch gigantesco aumenta latência, memória e dimensão do retry. Agrupar em tamanhos operacionais e observar partial failures. Ordered é previsível para dependências sequenciais; unordered tende a melhor throughput para operações independentes.

`findOneAndUpdate` faz read+write atómico e evita uma segunda round trip. Projection reduz a resposta. Cursor `batchSize` equilibra viagens de rede e memória; valor pequeno cria muitos `getMore`, valor grande aumenta buffers.

Validar filtros evita queries acidentais amplas. `maxTimeMS` e comments ajudam controlo/observabilidade. Hints rígidos podem tornar deployments frágeis durante alterações de índices.

---

## Armadilhas comuns

- **Usar `result.value` por hábito antigo:** por defeito atual, compound methods devolvem o documento.
- **Esperar documento de `updateOne`:** devolve `UpdateResult`.
- **Update seguido de find para obter estado:** janela concorrente.
- **Espalhar `request.query` num filtro:** operator injection e queries caras.
- **Engolir todos os erros como 404:** perde conflitos e outages.
- **Fazer retry manual de duplicate key:** não resolve a constraint.
- **Assumir bulk all-or-nothing:** não é transação.
- **Misturar paradigmas num cursor.**
- **Executar chamadas concorrentes sobre o mesmo cursor.**
- **Usar `ObjectId.isValid` sem depois construir `ObjectId`:** validação não converte.
- **Fechar client no repository:** viola ownership.

---

## O que costuma aparecer no exame

- Tipo devolvido por cada método CRUD.
- `findOneAndUpdate` e `returnDocument`.
- `includeResultMetadata`.
- Cursor, `batchSize`, `toArray` e async iteration.
- Ordered/unordered em `bulkWrite`.
- Partial failures e contadores.
- Duplicate key code 11000.
- Retryable operations.
- Projection e options no driver.
- Diferença entre ausência, conflito e erro.

---

## Resumo

O driver atual expõe Promises e cursores. Cada método tem um tipo de retorno específico: result object, documento/`null` ou cursor. Compound operations devolvem documento por defeito e resolvem read+write atomicamente. `bulkWrite` reduz round trips, sem atomicidade global. Repositories devem receber collections, validar input e preservar conflitos/erros. Pools, cursor backpressure, projections e timeouts mantêm a implementação segura e escalável.

Fontes oficiais: [CRUD do driver](https://www.mongodb.com/docs/drivers/node/current/crud/), [compound operations](https://www.mongodb.com/docs/drivers/node/current/crud/compound-operations/), [bulk operations](https://www.mongodb.com/docs/drivers/node/current/crud/bulk-write/), [cursor](https://www.mongodb.com/docs/drivers/node/current/crud/query/cursor/) e [API do driver](https://mongodb.github.io/node-mongodb-native/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — No driver atual, `findOneAndUpdate()` e restantes compound operations devolvem documento ou `null` por defeito. `result.value` só se aplica quando `includeResultMetadata: true` devolve um `ModifyResult`.

> ### ARMADILHA
>
> `bulkWrite()` reduz round trips, mas não é uma transação. Em ordered bulk, operações anteriores ao erro podem já ter sido aplicadas; em unordered bulk, outras operações podem continuar. Tratar qualquer erro como “nada aconteceu” corrompe a recuperação.

> ### DICA DE MEMORIZAÇÃO
>
> **CRUD devolve três famílias:** result object, documento/`null` ou cursor. Identifica a família antes de ler propriedades.

> ### COMPARAÇÃO
>
> A API escolhida define o objeto que o código pode interpretar.

| Operação | Retorno principal | Propriedade/consumo |
|---|---|---|
| `insertOne()` | `InsertOneResult` | `insertedId` |
| `updateOne()` | `UpdateResult` | `matchedCount`/`modifiedCount` |
| `deleteOne()` | `DeleteResult` | `deletedCount` |
| `findOneAndUpdate()` | documento/`null` | fields do documento |
| `find()` | `FindCursor` | iterar/`next()`/`toArray()` |
| `bulkWrite()` | `BulkWriteResult` | contadores + erros parciais |

> **Ligação entre capítulos:** métodos CRUD detalhados nos capítulos 05–07; pool no 04; índices no 09; transações no 12.

### Mapa mental do repository

~~~text
HTTP input
   |
   v
validar + autorizar + construir allowlist
   |
   v
repository recebe Collection partilhada
   |
   +-> método correto
   +-> projection / timeout
   `-> interpretar retorno e erro
            |
            v
resposta de domínio sem expor dados internos
~~~

### Mini desafio

Um exemplo antigo faz `const updated = result.value` depois de `findOneAndUpdate()` sem configurar metadata. Indica o comportamento esperado no driver atual e duas correções possíveis, conforme o retorno realmente pretendido.

---

## Resumo Rápido

- Cada método tem um tipo de retorno específico.
- Compound operations devolvem documento/`null` por defeito.
- `includeResultMetadata: true` muda o shape do retorno.
- `bulkWrite()` reduz round trips, não cria all-or-nothing.
- Repositories não criam nem fecham o client partilhado.
- Input externo exige validação e allowlist antes de MQL.

---

## Checklist

- [ ] Identifico o retorno de cada método CRUD.
- [ ] Não uso `result.value` por hábito antigo.
- [ ] Distingo compound operation de update seguido de find.
- [ ] Trato duplicate key por code/classe.
- [ ] Interpreto partial failures de bulk.
- [ ] Não aceito filtros/stages arbitrários de HTTP.
- [ ] Reutilizo `Collection`/`MongoClient` corretamente.
- [ ] Aplico projection, timeout e limites no repository.

---

## Perguntas para confirmar conhecimentos

1. Que três famílias de retorno aparecem no CRUD do driver?
2. O que devolve `findOneAndUpdate()` por defeito no driver atual?
3. Quando `result.value` volta a ser aplicável?
4. Porque update seguido de find não equivale sempre a uma compound operation?
5. Que informação útil existe num `MongoBulkWriteError`?
6. Como diferem ordered e unordered bulk depois de um erro?
7. Porque `bulkWrite()` não garante atomicidade global?
8. Que responsabilidades pertencem a um repository e quais não?
9. Porque não se deve construir um filtro espalhando diretamente um objeto HTTP?
10. Como distingues not found, conflito de unicidade e falha de infraestrutura?
