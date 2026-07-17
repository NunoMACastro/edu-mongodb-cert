# Introdução à certificação MongoDB Associate Developer (Node.js)

## Objetivos do capítulo

- Compreender o âmbito real do Learning Path e do exame.
- Preparar um ambiente de estudo coerente com o Node.js Driver atual.
- Distinguir MongoDB Server, Atlas, `mongosh` e o driver.
- Definir uma estratégia de estudo baseada em compreensão, prática e análise de planos.
- Saber como usar estes apontamentos sem memorizar APIs fora de contexto.

---

## Conceitos Fundamentais

### Âmbito do percurso

O percurso oficial cobre Atlas, modelo documental, connection strings, `mongosh`, ligação a partir de Node.js, CRUD, índices e aggregation. Inclui ainda transações e MongoDB Search como unidades eletivas. Estes apontamentos tratam as eletivas como matéria relevante porque fazem parte da estrutura solicitada e exercitam decisões arquiteturais importantes.

O exame Associate Developer Node.js é apresentado pela MongoDB University como uma prova de escolha múltipla, em inglês, com 53 perguntas e 75 minutos. Isto dá aproximadamente 85 segundos por pergunta. O objetivo não é decorar comandos raros: é reconhecer rapidamente a semântica correta de uma operação, o resultado provável e a opção adequada para o caso descrito.

### Baseline técnica destes apontamentos

Informação verificada em **16 de julho de 2026**:

| Componente     |                                              Baseline usada | Implicação                                       |
| -------------- | ----------------------------------------------------------: | ------------------------------------------------ |
| Node.js Driver |                      série 7.x; release notes atuais em 7.5 | APIs assíncronas, sem callbacks                  |
| Node.js        |                         20.19.0 ou superior para driver 7.x | usar ES Modules e `async`/`await`                |
| MongoDB Server | documentação corrente; exemplos compatíveis com Atlas atual | confirmar requisitos de funcionalidades recentes |
| Shell          |                                                   `mongosh` | não confundir com o antigo shell `mongo`         |

Não se deve inferir que uma funcionalidade do servidor existe apenas porque o método aparece no driver. A compatibilidade depende do trio **driver + server + topology**. Por exemplo, transações exigem replica set ou sharded cluster; MongoDB Search exige uma implementação suportada e um search index.

### Quatro componentes que aparecem juntos

- **MongoDB Server** executa comandos, persiste BSON, gere índices, replicação e transações.
- **MongoDB Atlas** gere deployments MongoDB e serviços associados na cloud.
- **`mongosh`** é uma interface interativa JavaScript para administração e exploração.
- **Node.js Driver** implementa descoberta de topologia, pools, BSON, autenticação e a API usada pela aplicação.

O driver não é um ORM. Ele expõe os conceitos do servidor de forma próxima do MongoDB Query Language (MQL). ODMs como Mongoose podem ser úteis num projeto, mas adicionam semântica própria e não substituem o conhecimento exigido no exame.

### Como estudar

Para cada operação, responder sempre a cinco perguntas:

1. Qual é o filtro e quantos documentos pode afetar?
2. O método devolve um documento, um result object ou um cursor?
3. A operação é atómica em que fronteira?
4. Que índice pode suportá-la?
5. Que custo existe em CPU, memória, I/O e rede?

Uma resposta sintaticamente válida pode estar errada por usar o método inadequado, projetar campos a mais, materializar resultados sem limite ou depender de uma atomicidade inexistente.

> **Importante para o exame:** não basta reconhecer sintaxe válida. Confirma sempre o tipo devolvido pelo método, a unidade de atomicidade e se a operação materializa um documento, um cursor ou um result object.

---

## Funcionamento Interno

Uma operação Node.js típica percorre esta cadeia:

1. O driver interpreta a URI, resolve DNS quando usa SRV e descobre a topologia.
2. O `MongoClient` seleciona um servidor elegível segundo tipo de operação, read preference e latência.
3. A operação obtém uma ligação do pool.
4. O driver serializa o documento JavaScript para BSON e envia um comando pelo wire protocol.
5. O servidor autentica e autoriza, planeia a query e executa-a.
6. A resposta BSON é desserializada para valores JavaScript e classes BSON do driver.
7. A ligação regressa ao pool; um cursor pode manter estado cliente/servidor e pedir batches adicionais.

Esta cadeia explica vários erros comuns: um timeout de seleção não significa necessariamente falha TCP; uma query lenta pode esperar pelo pool; `toArray()` pode consumir muita memória mesmo quando o servidor responde por batches.

### BSON e JSON não são equivalentes

BSON é o formato binário usado pelo servidor e pelo driver. Preserva tipos que JSON não representa diretamente, como `ObjectId`, `Decimal128`, `Long`, data, binário, expressão regular e timestamp. O objeto JavaScript que se vê no código é serializado; não é armazenado como texto JSON.

### Atomicidade

Cada write sobre um único documento é atómico, mesmo que altere vários campos ou elementos do documento. A atomicidade multi-documento exige uma transação quando a regra de negócio não pode tolerar estados intermédios. Um bom modelo documental reduz a necessidade de transações ao colocar dados alterados em conjunto no mesmo documento.

---

## Sintaxe

### Projeto mínimo

```json
{
    "name": "mongodb-certification-study",
    "private": true,
    "type": "module",
    "engines": {
        "node": ">=20.19.0"
    },
    "dependencies": {
        "mongodb": "^7.5.0"
    }
}
```

Instalação:

```bash
npm install
```

Variável de ambiente, sem colocar credenciais no código:

```bash
export MONGODB_URI='mongodb+srv://<user>:<password>@<cluster-host>/?retryWrites=true&w=majority'
```

Esqueleto oficial da API:

```javascript
import { MongoClient, ServerApiVersion } from "mongodb";

const uri = process.env.MONGODB_URI;

if (!uri) {
    throw new Error("A variável MONGODB_URI é obrigatória.");
}

const client = new MongoClient(uri, {
    appName: "mongodb-certification-study",
    serverApi: {
        version: ServerApiVersion.v1,
        strict: true,
        deprecationErrors: true,
    },
});

try {
    await client.connect();
    await client.db("admin").command({ ping: 1 });
} finally {
    await client.close();
}
```

`MongoClient(uri, options)` recebe a connection string e opções do cliente. `serverApi` opta pela Stable API v1; não é a versão do MongoDB Server nem do driver. `connect()` força o estabelecimento inicial e `command({ ping: 1 })` confirma uma round trip autorizada.

---

## Exemplos

### Exemplo 1 — verificar a ligação e consultar um documento

```javascript
import { MongoClient } from "mongodb";

const uri = process.env.MONGODB_URI;

if (!uri) {
    throw new Error("Defina MONGODB_URI antes de executar.");
}

const client = new MongoClient(uri, {
    appName: "certification-smoke-check",
    serverSelectionTimeoutMS: 5_000,
});

try {
    const movies = client.db("sample_mflix").collection("movies");
    const movie = await movies.findOne(
        { title: "The Matrix" },
        { projection: { _id: 0, title: 1, year: 1, genres: 1 } },
    );

    console.dir(movie, { depth: null });
} finally {
    await client.close();
}
```

Resultado esperado: um documento com os campos projetados, ou `null` se não existir correspondência. `findOne()` não devolve cursor.

### Exemplo 2 — iterar sem materializar tudo

```javascript
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const movies = client.db("sample_mflix").collection("movies");
    const cursor = movies
        .find({ genres: "Documentary", year: { $gte: 2015 } })
        .project({ _id: 0, title: 1, year: 1 })
        .sort({ year: -1, title: 1 })
        .limit(20);

    for await (const movie of cursor) {
        console.log(movie);
    }
} finally {
    await client.close();
}
```

Resultado esperado: no máximo 20 filmes, por ano descendente e título ascendente dentro do mesmo ano.

---

## Explicação linha a linha

### Exemplo 1

1. O import usa ES Modules e obtém `MongoClient` do driver oficial.
2. A URI vem do ambiente; a validação falha cedo em vez de passar `undefined` ao cliente.
3. `appName` torna a origem observável; `serverSelectionTimeoutMS` limita a espera pela seleção de servidor.
4. `db()` e `collection()` criam handles; não fazem por si uma query.
5. `findOne()` recebe filtro e opções separadas.
6. A projection exclui `_id` explicitamente e inclui só os campos necessários.
7. `await` resolve para documento ou `null`.
8. `finally` fecha o cliente mesmo perante erro.

### Exemplo 2

1. `find()` cria um `FindCursor`.
2. Comparar um campo array com `"Documentary"` corresponde se o array contiver esse valor.
3. `$gte` aplica um limite inferior inclusivo.
4. `project()` reduz o payload; `sort()` define ordem determinística com desempate.
5. `limit(20)` limita o conjunto antes da iteração.
6. `for await...of` pede batches à medida que são necessários, evitando um grande array em memória.

---

## Casos Reais

- **API de catálogo:** listar títulos recentes com filtros e projeção mínima.
- **Diagnóstico de deployment:** usar `ping` e `appName` para separar conectividade de problemas de query.
- **Job de processamento:** percorrer um cursor por batches em vez de carregar milhões de documentos.
- **Migração:** preservar `ObjectId`, datas e decimais como tipos BSON, em vez de converter tudo em strings.

---

## Performance

Sem índice, uma pesquisa pode realizar `COLLSCAN`, custo aproximadamente linear no número de documentos examinados. Com índice seletivo, o custo de navegação numa B-tree é aproximadamente logarítmico, acrescido das chaves/documentos lidos. Big-O é apenas uma aproximação: cache, seletividade, distribuição, working set e rede dominam frequentemente a latência real.

Reutilizar um `MongoClient` de longa duração permite reutilizar pools. Criar um cliente por pedido multiplica handshakes, sockets e pressão no servidor. Projeções reduzem rede e desserialização, mas uma projection não torna automaticamente a query coberta; o índice tem de conter filtro e campos devolvidos.

---

## Armadilhas comuns

- **Estudar APIs antigas:** callbacks, `useNewUrlParser` e `useUnifiedTopology` não pertencem ao driver 7 atual.
- **Confundir Stable API com versão do driver:** são eixos diferentes.
- **Criar `MongoClient` por request:** destrói os benefícios do pool.
- **Guardar a URI no Git:** expõe credenciais; usar secrets e environment variables.
- **Assumir que `find()` devolve array:** devolve cursor.
- **Aplicar `toArray()` sem limite:** pode esgotar a memória do processo.
- **Omitir `await`:** passa uma Promise adiante e altera completamente o tipo observado.
- **Fechar o cliente demasiado cedo:** um helper partilhado não deve fechar um cliente que não criou.
- **Confundir `null` com exceção em `findOne()`:** ausência de match devolve `null`.

---

## O que costuma aparecer no exame

- Diferença entre server, Atlas, `mongosh` e driver.
- Valor devolvido por `find()` versus `findOne()`.
- BSON versus JSON.
- Atomicidade single-document versus multi-document.
- Razão para reutilizar `MongoClient`.
- Efeito de projection, sort, limit e index.
- Significado de `_id` e `ObjectId`.
- Compatibilidade entre driver, server e topology.
- Identificação de exemplos com callbacks ou opções obsoletas.

---

## Resumo

O driver liga a aplicação ao servidor, gere topologia e pools, serializa BSON e expõe operações assíncronas. Atlas gere deployments e serviços; `mongosh` é uma interface interativa. `find()` devolve cursor, `findOne()` devolve documento ou `null`. Writes single-document são atómicos; transações resolvem invariantes multi-documento, com custo. Para o exame, raciocinar sobre tipo devolvido, cardinalidade, atomicidade, índice e custo.

Fontes oficiais: [Learning Path](https://learn.mongodb.com/learning-paths/mongodb-nodejs-developer-path), [página do exame](https://learn.mongodb.com/courses/mongodb-associate-developer-exam-nodejs), [Node.js Driver atual](https://www.mongodb.com/docs/drivers/node/current/), [release notes](https://www.mongodb.com/docs/drivers/node/current/reference/release-notes/) e [BSON types](https://www.mongodb.com/docs/manual/reference/bson-types/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — Muitas alternativas diferem apenas no tipo de retorno ou na fronteira de atomicidade. Antes de avaliar performance, identifica se o método devolve cursor, documento, `null` ou result object e se afeta um ou vários documentos.

> ### ARMADILHA
>
> Um `FindCursor` existir não significa que a query encontrou documentos. `find()` devolve imediatamente um cursor configurável; só a iteração, `next()` ou `toArray()` consome resultados. Confundir “tenho um cursor” com “tenho um match” leva a escolher código aparentemente plausível, mas semanticamente errado.

> ### DICA DE MEMORIZAÇÃO
>
> **V-C-R-A-I-C:** Verbo → Cardinalidade → Retorno → Atomicidade → Índice → Custo. Usa esta sequência para desmontar cada pergunta.

> ### COMPARAÇÃO
>
> Não troques a camada que executa dados pela ferramenta que a administra ou invoca.

| Componente     | Responsabilidade                  | Não confundir com |
| -------------- | --------------------------------- | ----------------- |
| MongoDB Server | executa comandos e persiste BSON  | Atlas             |
| Atlas          | gere deployments e serviços cloud | database user     |
| `mongosh`      | shell interativo                  | Node.js Driver    |
| Node.js Driver | API, BSON, discovery e pools      | ORM/ODM           |

> **Ligação entre capítulos:** os tipos de retorno são aprofundados nos capítulos 05–08; atomicidade e transações no capítulo 12; índices no capítulo 09.

### Mapa mental

```text
Aplicação Node.js
      |
      v
Node.js Driver -- BSON / pool / topology
      |
      v
MongoDB Server -- query / index / storage
      ^
      |
Atlas gere o deployment; mongosh é outra interface cliente
```

### Mini desafio

Sem executar, determina o tipo de `result`, quando a query é realmente enviada e que método usarias para saber se não existe nenhum match:

```javascript
const result = movies.find({ year: { $gte: 2020 } });
```

---

## Resumo Rápido

- MongoDB guarda BSON; JSON não preserva todos os tipos.
- Atlas, Server, `mongosh` e driver são camadas diferentes.
- `find()` devolve cursor; `findOne()` devolve documento ou `null`.
- Um write single-document é atómico.
- Transações multi-documento têm custo e exigem topologia compatível.
- Resolver perguntas por retorno, cardinalidade, atomicidade, índice e custo.

---

## Checklist

- [ ] Distingo Server, Atlas, `mongosh` e Node.js Driver.
- [ ] Sei explicar por que BSON não é apenas JSON binário.
- [ ] Reconheço documento, cursor e result object.
- [ ] Sei a fronteira da atomicidade single-document.
- [ ] Sei quando uma transação pode ser necessária.
- [ ] Consigo identificar a cardinalidade de uma operação.
- [ ] Verifico o tipo BSON antes de comparar valores.
- [ ] Aplico a sequência V-C-R-A-I-C a uma pergunta.

---

## Perguntas para confirmar conhecimentos

1. Que responsabilidades pertencem ao driver e quais pertencem ao servidor?
2. Porque não se deve tratar BSON e JSON como formatos semanticamente iguais?
3. Que tipo devolve `find()` antes de existir qualquer iteração?
4. Que valores pode devolver `findOne()`?
5. Qual é a fronteira de atomicidade garantida por um write normal?
6. Em que situação uma transação multi-documento é justificável?
7. Porque pode uma resposta sintaticamente válida continuar errada no exame?
8. Que custos devem ser considerados depois de confirmar a semântica da operação?
9. Porque não é correto assumir que uma funcionalidade do driver é suportada por qualquer server/topology?
10. Que seis perguntas da mnemónica V-C-R-A-I-C aplicarias a um snippet desconhecido?
