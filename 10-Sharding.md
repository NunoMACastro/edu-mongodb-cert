# Sharding e escalabilidade horizontal

## Objetivos do capítulo

- Compreender que problema o sharding resolve e quando não deve ser usado.
- Distinguir replicação, escalabilidade vertical e escalabilidade horizontal.
- Identificar shards, `mongos`, config servers, ranges e balancer.
- Avaliar shard keys por cardinalidade, frequência, monotonicidade e padrões de acesso.
- Comparar ranged, hashed e zoned sharding.
- Prever targeted queries, scatter-gather e o impacto no código Node.js.

---

## Conceitos Fundamentais

### Vocabulário de sharding

Sharding introduz uma camada de distribuição. Antes de analisar comandos, é necessário distinguir os elementos dessa camada:

| Conceito | Definição |
| -------- | --------- |
| Escalabilidade vertical | aumentar CPU, RAM, storage ou capacidade de uma máquina |
| Escalabilidade horizontal | acrescentar máquinas e distribuir dados ou trabalho entre elas |
| Sharding | particionamento horizontal de dados por vários shards |
| Sharded cluster | deployment composto por shards, routers e metadata de configuração |
| Shard | unidade que guarda uma parte dos dados; em produção é normalmente um replica set |
| Shard key | field ou conjunto ordenado de fields que determina a distribuição de uma collection |
| Range | intervalo do espaço de valores da shard key atribuído a um shard; historicamente chamado chunk |
| `mongos` | query router que recebe operações dos clientes e as encaminha para os shards necessários |
| Config server | membro do replica set que guarda metadata da topologia e da distribuição |
| Balancer | processo que observa desequilíbrios e migra ranges entre shards |
| Targeted query | operação encaminhada apenas para o shard ou subconjunto de shards relevante |
| Scatter-gather | operação enviada a todos os shards, seguida da reunião dos resultados |
| Hotspot | concentração de dados ou carga num range, shard ou valor da shard key |
| Primary shard | shard que guarda por defeito as collections não sharded de uma database |
| Refinement | extensão da shard key atual, acrescentando fields como sufixo |
| Resharding | redistribuição de uma collection segundo uma shard key nova ou uma redistribuição explícita |

O termo `primary shard` não significa o primary de um replica set. O primeiro é uma função ao nível da database num sharded cluster; o segundo é o membro que aceita writes dentro de um replica set.

### O problema que o sharding resolve

Uma única máquina tem limites físicos e económicos. Quando o dataset, o working set, o volume de writes ou o throughput de queries deixam de caber de forma sustentável num único replica set, sharding permite:

- dividir storage por várias máquinas;
- distribuir parte do trabalho de leitura;
- distribuir writes quando a shard key evita concentração;
- acrescentar shards à medida que a capacidade necessária cresce.

Sharding não torna automaticamente uma query mais rápida. Uma query que antes lia demasiados documentos pode passar a fazê-lo em vários shards. O ganho surge quando os dados e a carga ficam bem distribuídos e as operações conseguem ser encaminhadas de forma eficiente.

### Escalar verticalmente versus fazer sharding

| Estratégia | Como aumenta capacidade | Vantagem | Limitação |
| ---------- | ------------------------ | -------- | --------- |
| Vertical scaling | aumenta recursos do mesmo deployment | menor complexidade distribuída | limite físico, custo e janela de upgrade |
| Sharding | distribui dados por vários shards | capacidade horizontal de storage e workload | shard key, routing, migrations e operação mais complexos |

Escalar verticalmente costuma ser o primeiro passo quando ainda existe margem razoável. Sharding deve responder a uma necessidade medida, não à expectativa abstrata de crescimento.

### Replicação versus sharding

| Conceito | Pergunta que resolve | Forma dos dados |
| -------- | -------------------- | -------------- |
| Replica set | “Como continuar disponível perante falha de um membro?” | vários membros mantêm cópias do mesmo dataset |
| Sharded cluster | “Como distribuir dataset e workload por várias unidades?” | cada shard guarda uma parte dos dados sharded |

Um sharded cluster combina normalmente ambos: cada shard é um replica set. A replicação fornece redundância dentro do shard; o sharding distribui a collection entre shards.

### Arquitetura de um sharded cluster

```text
Aplicação Node.js
       |
       v
MongoDB Driver
       |
       v
     mongos  <------ metadata em cache ------ config server replica set
       |
       +------------------+------------------+
       |                  |                  |
       v                  v                  v
   Shard A            Shard B            Shard C
 replica set         replica set         replica set
 parte dos dados     parte dos dados     parte dos dados
```

| Componente | Responsabilidade | Não confundir com |
| ---------- | ---------------- | ----------------- |
| aplicação/driver | envia operações e monitoriza a topologia exposta | balancer |
| `mongos` | encaminha operações e combina resultados | processo `mongod` que guarda dados |
| config servers | guardam metadata de ranges, shards e configuração | backup do dataset |
| shard | executa operações sobre os dados que possui | um único servidor sem redundância |
| balancer | migra ranges para corrigir distribuição | query planner |

A aplicação liga-se ao router, não diretamente a um shard. Em Atlas, grande parte da infraestrutura é gerida pelo serviço, mas as consequências da shard key continuam a pertencer ao desenho da aplicação.

No modelo clássico, os config servers dedicam-se à metadata. MongoDB 8.0 introduziu a possibilidade de um **config shard**, no qual essa infraestrutura também guarda application data. Esta opção não elimina config metadata nem transforma todos os deployments atuais em config shards; deve ser tratada como uma variação explícita da topologia.

### Sharded e unsharded na mesma database

Uma database pode conter simultaneamente:

- collections sharded, cujos documentos estão repartidos por ranges;
- collections unsharded, localizadas num shard.

Por isso, “a database é sharded” é uma simplificação perigosa. A unidade que recebe a shard key é a collection. O primary shard da database aloja por defeito as collections que não estão distribuídas.

### A shard key é uma decisão de modelação

A shard key faz parte da identidade operacional de cada documento. É escolhida quando a collection é sharded e influencia:

- onde o documento é armazenado;
- que shard recebe um insert;
- quantos shards uma query tem de consultar;
- se writes se distribuem ou criam um hotspot;
- que unique indexes são possíveis;
- o custo de mudar a distribuição no futuro.

Não existe uma shard key universalmente “melhor”. A escolha depende simultaneamente dos dados e do workload.

### Características de uma shard key

| Característica | Pergunta | Risco quando inadequada |
| -------------- | -------- | ----------------------- |
| Cardinalidade | existem valores distintos suficientes? | poucos ranges possíveis e crescimento limitado |
| Frequência | alguns valores aparecem muito mais do que outros? | concentração de dados e carga |
| Monotonicidade | os novos valores crescem ou diminuem continuamente? | writes acumulados na extremidade do key space |
| Query alignment | as queries frequentes incluem a key ou um prefixo útil? | scatter-gather frequente |
| Write distribution | os inserts/updates repartem-se pelos shards? | um shard recebe a maior parte dos writes |
| Localidade | interessa manter ranges próximos? | hashed sharding pode destruir a ordem útil |

Alta cardinalidade é necessária, mas não suficiente. `_id` tem alta cardinalidade; se o workload procurar principalmente por `tenantId`, usar apenas `_id` como shard key pode distribuir bem os writes e continuar a produzir muitas queries broadcast.

### Ranged, hashed e zoned sharding

| Estratégia | Distribuição | Favorece | Compromisso |
| ---------- | ------------ | -------- | ----------- |
| Ranged | valores próximos ficam em ranges próximos | range queries e localidade | valores monotónicos podem concentrar writes |
| Hashed | usa o hash de um field da shard key | distribuição de valores monotónicos e equality | range queries perdem localidade natural |
| Zoned | associa ranges a grupos de shards | residência, geografia ou isolamento | configuração e capacidade por zona |

Hashed não significa aleatório no cliente. A aplicação envia o valor original; MongoDB calcula o hash necessário para routing e distribuição.

Zones não substituem ranged ou hashed. São uma restrição adicional que associa intervalos do espaço da shard key a shards pertencentes a uma zona.

### Compound shard keys e prefixos

Uma shard key pode ter vários fields:

```javascript
const shardKey = { tenantId: 1, orderId: 1 };
```

Neste exemplo:

- `tenantId` é prefixo da shard key;
- `tenantId + orderId` identifica um ponto mais específico;
- `orderId` isolado não é prefixo.

Uma query por `tenantId` pode ser encaminhada para o conjunto de ranges desse tenant. Uma query apenas por `orderId` não fornece o prefixo inicial e pode exigir broadcast.

A ordem dos fields deve equilibrar isolamento, distribuição e queries frequentes. Colocar `tenantId` primeiro favorece operações tenant-scoped, mas um tenant muito maior do que os restantes pode exigir um sufixo que permita dividir os seus dados por vários ranges.

### Quando não usar sharding

Sharding não é o primeiro remédio para:

- queries sem índices;
- documentos mal modelados;
- payloads excessivos;
- falta de paginação;
- pools mal configurados;
- agregações que multiplicam cardinalidade sem necessidade;
- um dataset pequeno que cabe confortavelmente num replica set.

Antes de shardear, medir CPU, memória, I/O, storage, latência, throughput e crescimento. Corrigir primeiro modelação, índices e queries quando estes forem a causa.

> **Importante para o exame:** replicação e sharding resolvem problemas diferentes. A shard key não é apenas um índice: determina distribuição e routing. Uma query que não contém informação suficiente da shard key pode ser enviada a todos os shards mesmo que cada shard use um índice local.

---

## Funcionamento Interno

### Routing de uma operação

Quando o driver envia uma operação:

1. `mongos` recebe o comando.
2. Consulta a metadata em cache sobre os ranges.
3. Analisa o filtro e a shard key.
4. Determina os shards que podem conter os dados.
5. Encaminha a operação para esses shards.
6. Cada shard planeia e executa localmente a sua parte.
7. `mongos` combina resultados quando mais de um shard participa.
8. O driver recebe uma resposta com a mesma API de alto nível.

Sharding é maioritariamente transparente ao código de ligação, mas não é transparente à performance nem à semântica de alguns writes.

### Targeted versus scatter-gather

Com a shard key `{ tenantId: 1, orderId: 1 }`:

```javascript
const targetedFilter = {
    tenantId: "tenant-42",
    orderId: "order-9001",
};

const potentiallyBroadcastFilter = {
    status: "pending",
};
```

O primeiro filtro contém a shard key completa e pode localizar um range específico. O segundo não contém a shard key; `mongos` pode ter de consultar todos os shards.

“Targeted” não significa necessariamente “usa um bom índice”. Depois do routing, o shard ainda precisa de um índice local apropriado. Existem, portanto, duas perguntas:

1. Quantos shards recebem a operação?
2. Quanto trabalho faz cada shard que a recebe?

### Inserts, reads e writes

Um insert inclui o documento e, por isso, fornece o valor da shard key. `mongos` calcula o destino correspondente. Num `insertMany()`, documentos diferentes podem ser encaminhados para shards diferentes.

Reads podem ser:

- single-shard;
- dirigidos a um subconjunto de shards;
- scatter-gather.

Writes de um único documento têm regras de targeting próprias. Incluir igualdade sobre a shard key completa é a forma mais explícita e previsível de identificar o destino. Operações como `findOneAndUpdate()` sobre collections sharded exigem particular atenção aos requisitos do filtro na versão usada.

### Ranges, splits e balancer

MongoDB divide o espaço da shard key em ranges. Um range tem limite inferior inclusivo e limite superior exclusivo. A metadata associa cada range a um shard.

O balancer:

- observa a distribuição de cada collection sharded;
- respeita zones configuradas;
- inicia migrations quando os critérios de desequilíbrio são atingidos;
- move ranges entre shards;
- atualiza a metadata depois da sincronização.

Migrations consomem rede, storage I/O e capacidade de replicação. A aplicação pode continuar a operar, mas “transparente” não significa “sem custo”.

### Ranges indivisíveis e jumbo

MongoDB não pode separar documentos que têm exatamente o mesmo valor completo da shard key por ranges diferentes. Se um valor muito frequente acumular uma quantidade desproporcional de dados, o range pode tornar-se indivisível ou problemático para migrations, frequentemente descrito na documentação operacional como **jumbo**. A correção estrutural é aumentar a granularidade da shard key, refinar ou reshardear segundo um desenho adequado; acrescentar shards sem corrigir a key não divide esse valor.

### Execução distribuída e merge

Cada shard tem o seu query planner e os seus índices. Numa operação multi-shard:

- os shards executam trabalho local;
- podem devolver batches parciais;
- `mongos` ou um shard designado pode fazer merge;
- sorts, groups e joins podem exigir coordenação adicional.

Uma aggregation pode começar com `$match` targeted e limitar os shards participantes. Sem filtro compatível com a shard key, a pipeline pode começar em todos os shards e reunir resultados mais tarde.

### Índices e unique constraints

O índice que suporta a shard key e os índices que suportam queries resolvem responsabilidades relacionadas, mas distintas.

Numa collection sharded, MongoDB só consegue impor globalmente um unique index quando a shard key completa é prefixo desse índice. Caso contrário, shards diferentes poderiam aceitar o mesmo valor sem coordenação global suficiente.

Exemplo de unicidade por tenant:

```javascript
const shardKey = { tenantId: 1, orderId: 1 };
const tenantUniqueKey = {
    tenantId: 1,
    orderId: 1,
    externalReference: 1,
};
```

O unique index começa pela shard key completa e pode acrescentar `externalReference`. A constraint aplica-se à combinação total; não torna `externalReference` globalmente única por si só. Não assumir que um unique index arbitrário mantém unicidade global depois de shardear.

### Primary shard e disponibilidade

Cada shard é normalmente um replica set:

- o primary desse replica set aceita os writes dirigidos ao shard;
- secondaries replicam os dados desse shard;
- uma eleição pode substituir o primary sem mover o range para outro shard.

Uma falha de membro e uma migration de range são eventos diferentes. A primeira é tratada pela replicação; a segunda altera a distribuição horizontal.

### Refinar versus reshardear

| Operação | Alteração | Quando considerar |
| -------- | --------- | ----------------- |
| Refine shard key | acrescenta fields ao fim da key atual | falta granularidade e o prefixo existente continua correto |
| Reshard collection | define outra key ou força redistribuição | estratégia atual já não representa o workload |

Refinar `{ tenantId: 1 }` para `{ tenantId: 1, orderId: 1 }` preserva o prefixo e acrescenta granularidade. Não transforma `tenantId` de ranged em hashed nem substitui os fields existentes.

Resharding é uma operação planeada e intensiva: exige capacidade de storage, acompanhamento e verificação dos requisitos da versão. Desde MongoDB 8.0, também pode redistribuir segundo a mesma shard key com `forceRedistribution`, mas isso não converte a operação numa manutenção sem custo.

---

## Sintaxe

Os comandos administrativos seguintes são executados contra `mongos` e exigem privilégios apropriados. Em Atlas, confirmar sempre as operações suportadas pelo tier e usar os workflows de administração recomendados.

### Inspecionar a distribuição

```mongosh
sh.status()

db.getSiblingDB("admin").runCommand({ listShards: 1 })
```

`sh.status()` resume shards, databases, collections sharded, shard keys e distribuição. `listShards` devolve os shards registados no cluster.

### Criar o índice de suporte e shardear por range

```mongosh
use sales

db.orders.createIndex(
    { tenantId: 1, orderId: 1 },
    { name: "tenant_order_shard_key" }
)

sh.shardCollection(
    "sales.orders",
    { tenantId: 1, orderId: 1 }
)
```

As duas chamadas resolvem problemas relacionados, mas diferentes:

- `createIndex()` cria a estrutura local que organiza `tenantId` e `orderId` nessa ordem;
- `sh.shardCollection(namespace, key)` declara ao cluster que a collection será distribuída segundo esse key pattern;
- `"sales.orders"` é um namespace completo `database.collection`, não apenas o nome da collection;
- `{ tenantId: 1, orderId: 1 }` é simultaneamente a shard key declarada e o prefixo exigido no índice de suporte;
- o `name` do índice é operacional e não é passado a `shardCollection()`.

Para uma collection já populada, a shard key requer um índice de suporte cujo prefixo corresponda à key. Criar o índice não ativa sharding por si só; shardear também não substitui a validação dos access patterns.

### Hashed sharding

```mongosh
use telemetry

db.events.createIndex(
    { deviceId: "hashed" },
    { name: "device_hashed_shard_key" }
)

sh.shardCollection(
    "telemetry.events",
    { deviceId: "hashed" }
)
```

Esta estratégia favorece distribuição por hash de `deviceId`. Não conserva a ordem natural para range queries por esse field.

`"hashed"` aparece no key pattern, não numa option `{ hashed: true }`. Cada valor de `deviceId` é transformado numa hash para distribuição. A igualdade pelo valor original continua a permitir targeting porque `mongos` calcula a mesma hash; um range original como `{ deviceId: { $gte: 1000 } }` não corresponde a um range contíguo das hashes.

### Configurar zones

```mongosh
sh.addShardToZone("shard-eu-01", "EU")
sh.addShardToZone("shard-us-01", "US")

sh.updateZoneKeyRange(
    "accounts.customers",
    { region: "EU", customerId: MinKey },
    { region: "EU", customerId: MaxKey },
    "EU"
)

sh.updateZoneKeyRange(
    "accounts.customers",
    { region: "US", customerId: MinKey },
    { region: "US", customerId: MaxKey },
    "US"
)
```

As boundaries têm a mesma forma da shard key. Zones exigem desenho completo para cobrir os ranges pretendidos e capacidade suficiente nos shards associados.

Leitura das chamadas:

- `addShardToZone(shardName, zoneName)` associa um shard existente a uma label de zone;
- `updateZoneKeyRange(namespace, min, max, zoneName)` associa o intervalo semiaberto `[min, max)` a essa zone;
- `MinKey` e `MaxKey` são valores BSON sentinela que representam os extremos possíveis do segundo field;
- `min` é inclusivo e `max` exclusivo, pelo que boundaries adjacentes devem ser desenhadas sem gaps ou overlaps inválidos;
- a ordem e todos os prefixos nos boundary documents têm de respeitar a shard key.

### Analisar uma shard key candidata

```mongosh
use sales

db.orders.createIndex({ tenantId: 1, orderId: 1 })

db.orders.analyzeShardKey(
    { tenantId: 1, orderId: 1 },
    {
        keyCharacteristics: true,
        readWriteDistribution: false,
        sampleSize: 10000
    }
)
```

`analyzeShardKey` pode medir cardinalidade, frequência e monotonicidade. Métricas de read/write distribution requerem query sampling representativo e privilégios adequados. Uma amostra enviesada produz uma decisão enviesada.

No segundo argumento, `keyCharacteristics: true` pede análise das propriedades dos valores. `readWriteDistribution: false` evita pedir métricas de workload nesta chamada. `sampleSize` limita a amostra pedida; não é “número de chunks a criar” nem garantia de representatividade estatística.

### Refinar ou reshardear

```mongosh
db.adminCommand({
    refineCollectionShardKey: "sales.orders",
    key: { tenantId: 1, orderId: 1 }
})

db.adminCommand({
    reshardCollection: "sales.orders",
    key: { region: 1, orderId: "hashed" }
})
```

Estes comandos representam mudanças operacionais importantes. Não devem ser executados como parte do startup normal da aplicação.

`refineCollectionShardKey` apenas acrescenta fields ao final da shard key existente e preserva o prefixo. `reshardCollection` pode escolher uma key diferente e redistribuir dados. Em ambos, `key` é um key pattern; não é uma query. A semelhança sintática não implica custo ou efeito equivalentes.

---

## Exemplos

### Exemplo 1 — confirmar a topologia e fazer uma query targeted

```javascript
/**
 * Demonstra como uma aplicação deteta um router mongos e executa
 * uma leitura que inclui a shard key completa.
 */
import { MongoClient } from "mongodb";

const uri = process.env.MONGODB_URI;

if (!uri) {
    throw new Error("MONGODB_URI é obrigatória");
}

const client = new MongoClient(uri, {
    appName: "sharding-targeted-query-example",
});

try {
    await client.connect();

    const hello = await client.db("admin").command({ hello: 1 });
    const isMongos = hello.msg === "isdbgrid";

    console.log({ isMongos });

    const orders = client.db("sales").collection("orders");
    const order = await orders.findOne({
        tenantId: "tenant-42",
        orderId: "order-9001",
    });

    console.log(order);
} finally {
    await client.close();
}
```

O código de ligação é igual ao usado noutros deployments. A diferença está no filtro: ao fornecer `tenantId` e `orderId`, a aplicação permite routing preciso para a shard key `{ tenantId: 1, orderId: 1 }`.

### Exemplo 2 — comparar evidência de routing com explain

```javascript
/**
 * Compara dois planos sem assumir que o nome exato dos stages é
 * igual em todas as versões do servidor.
 */
import { MongoClient } from "mongodb";

const uri = process.env.MONGODB_URI;

if (!uri) {
    throw new Error("MONGODB_URI é obrigatória");
}

const client = new MongoClient(uri, {
    appName: "sharding-explain-example",
});

try {
    await client.connect();

    const orders = client.db("sales").collection("orders");

    const targetedPlan = await orders
        .find({
            tenantId: "tenant-42",
            orderId: "order-9001",
        })
        .explain("executionStats");

    const broadcastCandidatePlan = await orders
        .find({
            status: "pending",
        })
        .explain("executionStats");

    console.dir(
        {
            targetedPlan,
            broadcastCandidatePlan,
        },
        { depth: null },
    );
} finally {
    await client.close();
}
```

O objetivo não é procurar uma única string decorada. Deve observar-se quantos shards participam, que planos locais usam, quantos documentos/keys examinam e quantos resultados devolvem.

### Exemplo 3 — repository que preserva tenant scope

```javascript
/**
 * Cria operações de encomendas que exigem sempre a parte tenant-scoped
 * da shard key. A collection é injetada para separar acesso a dados
 * do lifecycle do MongoClient.
 *
 * @param {import("mongodb").Collection} collection
 * @returns {{
 *   findOrder: (input: {tenantId: string, orderId: string}) => Promise<object|null>,
 *   updateOrderStatus: (input: {tenantId: string, orderId: string, status: string}) => Promise<object>
 * }}
 */
export function createOrderRepository(collection) {
    return {
        async findOrder({ tenantId, orderId }) {
            validateIdentifier("tenantId", tenantId);
            validateIdentifier("orderId", orderId);

            return collection.findOne({
                tenantId,
                orderId,
            });
        },

        async updateOrderStatus({ tenantId, orderId, status }) {
            validateIdentifier("tenantId", tenantId);
            validateIdentifier("orderId", orderId);
            validateIdentifier("status", status);

            return collection.updateOne(
                {
                    tenantId,
                    orderId,
                },
                {
                    $set: {
                        status,
                        updatedAt: new Date(),
                    },
                },
            );
        },
    };
}

/**
 * Rejeita identificadores vazios para impedir queries acidentalmente
 * sem tenant scope e para manter um contrato previsível.
 *
 * @param {string} fieldName
 * @param {unknown} value
 * @returns {asserts value is string}
 */
function validateIdentifier(fieldName, value) {
    if (typeof value !== "string" || value.trim() === "") {
        throw new TypeError(fieldName + " deve ser uma string não vazia");
    }
}
```

Exigir `tenantId` no contrato é simultaneamente uma decisão de autorização e de routing. A shard key nunca substitui verificação de permissões: um utilizador não fica autorizado a um tenant apenas porque forneceu o respetivo identificador.

### Exemplo 4 — analisar uma shard key candidata a partir do driver

```javascript
/**
 * Executa analyzeShardKey através do comando da database da collection.
 * Requer MongoDB compatível, índice de suporte e privilégios administrativos.
 */
import { MongoClient } from "mongodb";

const uri = process.env.MONGODB_URI;

if (!uri) {
    throw new Error("MONGODB_URI é obrigatória");
}

const client = new MongoClient(uri, {
    appName: "shard-key-analysis-example",
});

try {
    await client.connect();

    const result = await client.db("admin").command({
        analyzeShardKey: "sales.orders",
        key: {
            tenantId: 1,
            orderId: 1,
        },
        keyCharacteristics: true,
        readWriteDistribution: false,
        sampleSize: 10000,
    });

    console.dir(result.keyCharacteristics, { depth: null });
} finally {
    await client.close();
}
```

O comando recolhe evidência sobre a key candidata. Não escolhe a shard key pela aplicação nem substitui análise dos access patterns.

---

## Explicação linha a linha

### Exemplo 1

- O comentário inicial declara o objetivo e evita confundir deteção com provisionamento.
- A URI vem do ambiente para não expor credenciais.
- Um único `MongoClient` gere discovery e pools.
- `hello` permite observar se a ligação expõe um `mongos` através de `msg: "isdbgrid"`.
- `findOne()` recebe igualdade sobre a shard key completa.
- `finally` encerra o client mesmo perante erro.

### Exemplo 2

- Os dois filtros mantêm a mesma collection para tornar a comparação útil.
- O filtro targeted contém `tenantId` e `orderId`.
- O segundo filtro contém apenas `status` e pode exigir broadcast.
- `executionStats` acrescenta trabalho real do plano à estrutura explicativa.
- `console.dir()` usa profundidade ilimitada porque os planos são nested.
- Não se faz parsing rígido de nomes de stages que podem variar por versão.

### Exemplo 3

- A factory recebe a collection já configurada.
- O contrato de cada método exige `tenantId` e `orderId`.
- A validação ocorre antes de construir o filtro.
- Read e update usam a mesma forma de targeting.
- `$set` altera apenas os fields pretendidos.
- A função de validação documenta a garantia produzida.

### Exemplo 4

- O comando é executado na database `admin` e recebe o namespace completo `sales.orders`.
- `key` contém a shard key candidata.
- `keyCharacteristics` pede métricas dos valores.
- `readWriteDistribution: false` evita fingir que existe workload sampling.
- `sampleSize` torna explícita a dimensão máxima pretendida.
- O output é evidência para uma decisão humana, não uma decisão automática.

---

## Casos Reais

- **SaaS multi-tenant:** uma compound shard key iniciada por `tenantId` favorece isolamento lógico e queries tenant-scoped; tenants muito grandes exigem granularidade adicional.
- **Telemetria com muitos devices:** hashed sharding de um identificador pode distribuir writes, mas relatórios por intervalos temporais precisam de outro desenho de índice e podem consultar vários shards.
- **E-commerce global:** zones podem aproximar dados da região adequada e apoiar requisitos de residência, desde que a capacidade regional acompanhe a carga.
- **Catálogo muito grande:** sharding aumenta storage e throughput, mas reads por atributos que não pertencem à shard key podem continuar scatter-gather.
- **Jobs por organização:** manter `organizationId` no filtro de claims, jobs e updates melhora routing e reduz risco de operação cross-tenant.
- **Collection pequena de configurações:** mantê-la unsharded pode ser mais simples; nem todas as collections de um cluster precisam de distribuição.

---

## Performance

O custo de uma operação sharded deve ser analisado em duas dimensões:

1. **fan-out:** quantos shards recebem a operação;
2. **trabalho local:** quantas keys e documentos cada shard examina.

Uma targeted query com `COLLSCAN` local pode ser lenta. Uma scatter-gather query com bons índices locais pode ainda sofrer de fan-out, latência do shard mais lento e merge.

### Hotspots

Um hotspot pode resultar de:

- shard key monotónica em ranged sharding;
- um valor de frequência muito alta;
- um tenant desproporcionalmente grande;
- queries populares concentradas num range;
- zone com capacidade insuficiente.

Distribuição uniforme de documentos não garante distribuição uniforme de workload. Um shard com menos dados pode receber mais operações.

### Migrations

Enquanto o balancer migra ranges:

- há cópia de dados entre shards;
- writes concorrentes precisam de sincronização;
- replication e storage fazem trabalho adicional;
- caches podem sofrer pressão;
- a latência pode variar.

Adicionar um shard vazio não produz capacidade equilibrada instantaneamente. A redistribuição demora e deve ser acompanhada.

### Aggregation e transactions

Pipelines que começam com `$match` compatível com a shard key podem reduzir os participantes. `$lookup`, `$group` e `$sort` distribuídos podem exigir merge e memória adicional.

Transações cross-shard acrescentam coordenação distribuída ao custo normal de uma transação. Modelar e filtrar para manter invariantes num documento ou num único shard, quando correto, reduz essa coordenação.

### Medição

Usar em conjunto:

- `explain("executionStats")`;
- métricas de routing e profiler/Query Profiler;
- `analyzeShardKey` com sampling representativo;
- distribuição de storage e operações por shard;
- latência p50, p95 e p99;
- comportamento durante migrations e failovers.

---

## Armadilhas comuns

- **Confundir replicação com sharding:** uma replica dados; o outro distribui-os.
- **Shardear para corrigir uma query sem índice:** multiplica o problema.
- **Escolher apenas pela cardinalidade:** ignora frequência e padrões de query.
- **Usar field monotónico com ranged sharding:** pode concentrar inserts no último range.
- **Assumir que hashed suporta ranges naturais:** o hash destrói essa localidade.
- **Usar um array como hashed shard key:** hashed indexes não suportam fields array.
- **Omitir a shard key dos filtros frequentes:** aumenta scatter-gather.
- **Ligar a aplicação diretamente a um shard:** ignora o router e a visão global.
- **Confundir primary shard com replica set primary:** são níveis diferentes.
- **Assumir que todos os unique indexes continuam globais:** existem restrições próprias.
- **Desligar o balancer indefinidamente:** permite acumular desequilíbrio.
- **Adicionar shards e esperar equilíbrio imediato:** migrations precisam de tempo e recursos.
- **Executar resharding como rotina de aplicação:** é uma operação administrativa planeada.
- **Usar zone sharding como autorização:** localização de dados não substitui access control.
- **Acreditar que Atlas escolhe a shard key pela aplicação:** gestão de infraestrutura não elimina modelação.

---

## O que costuma aparecer no exame

No percurso Associate Developer, sharding tende a surgir como contexto de arquitetura, Atlas, índices e transações, não como administração profunda de cluster. É especialmente importante saber:

- escalabilidade vertical versus horizontal;
- replica set versus sharded cluster;
- função de shard, `mongos` e config servers;
- shard key e índice de suporte;
- cardinalidade, frequência e monotonicidade;
- ranged versus hashed;
- targeted query versus scatter-gather;
- impacto de incluir ou omitir a shard key;
- relação entre sharding e unique indexes;
- maior custo de operações cross-shard.

Não é razoável decorar todos os comandos administrativos para um exame de desenvolvimento. É necessário conseguir prever routing e reconhecer uma shard key inadequada.

---

## Resumo

Sharding distribui collections por vários shards para ultrapassar limites de storage e workload de um único replica set. `mongos` encaminha operações segundo metadata mantida pelos config servers. A shard key determina distribuição e routing, devendo ser avaliada por cardinalidade, frequência, monotonicidade e queries reais. Ranged preserva localidade; hashed favorece distribuição; zones restringem ranges a grupos de shards. Targeted queries reduzem fan-out; scatter-gather consulta vários shards. O balancer corrige distribuição através de migrations, que têm custo. Sharding complementa replicação, índices e modelação — não os substitui.

Fontes oficiais: [sharding](https://www.mongodb.com/docs/manual/sharding/), [componentes](https://www.mongodb.com/docs/manual/core/sharded-cluster-components/), [shard keys](https://www.mongodb.com/docs/manual/core/sharding-shard-key/), [escolher shard key](https://www.mongodb.com/docs/manual/core/sharding-choose-a-shard-key/), [routing com mongos](https://www.mongodb.com/docs/manual/core/sharded-cluster-query-router/), [balancer](https://www.mongodb.com/docs/manual/core/sharding-balancer-administration/), [zones](https://www.mongodb.com/docs/manual/tutorial/manage-shard-zone/), [restrições operacionais](https://www.mongodb.com/docs/manual/core/sharded-cluster-requirements/), [refinement](https://www.mongodb.com/docs/manual/core/sharding-refine-a-shard-key/), [resharding](https://www.mongodb.com/docs/manual/core/sharding-reshard-a-collection/) e [analyzeShardKey](https://www.mongodb.com/docs/manual/reference/command/analyzeShardKey/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Alta** — Uma shard key influencia simultaneamente distribuição e routing. Alta cardinalidade, baixa concentração e ausência de monotonicidade ajudam a distribuição; alinhamento com queries frequentes ajuda targeting. Nenhuma característica isolada prova que a key é adequada.

> ### ARMADILHA
>
> Uma query pode usar um índice em cada shard e continuar scatter-gather. `IXSCAN` responde à pergunta “como procurou este shard?”; a shard key responde primeiro a “que shards tiveram de procurar?”.

> ### DICA DE MEMORIZAÇÃO
>
> **Shard key decide onde; índice decide como procurar dentro. Router aponta, shard executa.**

> ### COMPARAÇÃO
>
> Antes de escolher a estratégia, separar objetivo, distribuição e padrão de query.

| Opção | Preserva ordem natural | Distribui monotonicidade | Caso típico |
| ----- | ---------------------- | ----------------------- | ----------- |
| ranged | sim | não necessariamente | range queries e localidade |
| hashed | não | normalmente melhor | equality e writes distribuídos |
| zoned | depende da key base | depende da key base | geografia, residência ou isolamento |

> **Ligação entre capítulos:** o modelo e access patterns são definidos no 02; filtros no 05; índices de suporte no 09; aggregation distribuída no 11–12; transactions cross-shard no 13.

### Mapa mental de sharding

```text
Necessidade de escala horizontal
|
+-- Arquitetura
|   +-- mongos -> routing
|   +-- config servers -> metadata
|   `-- shards -> partes dos dados + replica sets
|
+-- Shard key
|   +-- cardinalidade
|   +-- frequência
|   +-- monotonicidade
|   `-- query/write distribution
|
+-- Estratégia
|   +-- ranged -> localidade
|   +-- hashed -> distribuição
|   `-- zones -> placement
|
`-- Operação
    +-- targeted vs scatter-gather
    +-- ranges + balancer + migrations
    `-- refine vs reshard
```

### Fluxograma: devo shardear e com que key?

```text
O limite está medido num único replica set?
  |-- não --> medir e corrigir modelo/query/índice
  `-- sim --> workload pode ser distribuído por uma key?
                |-- não --> rever access patterns/modelo
                `-- sim --> avaliar candidatos
                              |
                              +-- cardinalidade suficiente?
                              +-- frequência equilibrada?
                              +-- evita hotspot monotónico?
                              +-- queries críticas são targeted?
                              |
                              `-> testar com dados + workload representativos
```

### Mini desafio

Uma collection `orders` recebe milhões de inserts por dia. As queries mais frequentes são “encomenda por tenant e orderId” e “últimas encomendas de um tenant”. Compara `{ orderId: "hashed" }`, `{ tenantId: 1 }` e `{ tenantId: 1, orderId: 1 }`. Explica distribuição, targeting, risco de tenant gigante e índices adicionais necessários.

---

## Resumo Rápido

- Sharding é escalabilidade horizontal; replication é redundância.
- Cada shard guarda parte dos dados e é normalmente um replica set.
- `mongos` encaminha; config servers guardam metadata.
- A shard key decide distribuição e routing.
- Ranged preserva ordem; hashed distribui; zones controlam placement.
- Targeted reduz fan-out; scatter-gather consulta vários shards.
- Balancer migra ranges e consome recursos.
- Shard key e índice local respondem a perguntas diferentes.
- Refine acrescenta sufixos; reshard muda ou redistribui a estratégia.

---

## Checklist

- [ ] Distingo vertical scaling, replication e sharding.
- [ ] Explico shard, `mongos`, config servers e balancer.
- [ ] Distingo primary shard de replica set primary.
- [ ] Avalio cardinalidade, frequência e monotonicidade.
- [ ] Comparo ranged, hashed e zoned sharding.
- [ ] Identifico prefixos de uma compound shard key.
- [ ] Prevejo targeted versus scatter-gather.
- [ ] Separo routing global de uso de índice local.
- [ ] Reconheço hotspots e custo de migrations.
- [ ] Distingo refinement de resharding.
- [ ] Sei quando um replica set simples é preferível.
- [ ] Incluo shard key e tenant scope nos contratos adequados.

---

## Perguntas para confirmar conhecimentos

1. Que limite resolve sharding que um replica set, por si só, não resolve?
2. Porque cada shard é normalmente um replica set?
3. Que responsabilidades pertencem a `mongos` e aos config servers?
4. Qual é a diferença entre primary shard e replica set primary?
5. Porque alta cardinalidade não basta para escolher uma shard key?
6. Como frequência e monotonicidade podem criar hotspots?
7. Que trade-off existe entre ranged e hashed sharding?
8. Quando uma query se torna scatter-gather?
9. Porque `IXSCAN` não prova que uma operação sharded é eficiente?
10. Como a shard key restringe unique indexes?
11. O que acontece conceptualmente durante uma range migration?
12. Qual é a diferença entre refinar e reshardear uma collection?
13. Porque zone sharding não substitui autorização?
14. Que medições recolherias antes de shardear uma collection?
