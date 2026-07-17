# Connection strings e MongoDB Shell

## Objetivos do capítulo

- Interpretar URIs `mongodb://` e `mongodb+srv://`.
- Distinguir autenticação, default database, `authSource` e autorização.
- Fazer percent-encoding de credenciais e proteger secrets.
- Usar `mongosh` para explorar, consultar e diagnosticar.
- Reconhecer diferenças entre helpers do shell e APIs do Node.js Driver.

---

## Conceitos Fundamentais

Uma connection string descreve **como descobrir e autenticar uma ligação**, não apenas “onde está a database”. Contém esquema, credenciais opcionais, seed hosts, default database e opções.

### Standard versus SRV

| Característica  | `mongodb://`        | `mongodb+srv://`           |
| --------------- | ------------------- | -------------------------- |
| Hosts           | listados na URI     | descobertos por DNS SRV    |
| Porta           | explícita/opcional  | não é indicada na URI      |
| Topologia       | seed list manual    | seed list obtida por DNS   |
| Opções DNS TXT  | não                 | sim                        |
| TLS por defeito | não necessariamente | sim                        |
| Uso Atlas       | possível            | formato normal recomendado |

Uma seed list não é uma lista de ligações fixas. O driver contacta seeds para descobrir a topologia real e monitoriza alterações.

### Anatomia

```text
mongodb://username:password@host1:27017,host2:27017/app?replicaSet=rs0&authSource=admin
```

- **scheme:** identifica o formato.
- **userinfo:** credenciais; deve ser omitido quando o mecanismo usa identidade externa.
- **hosts:** seeds, não necessariamente todos os destinos futuros.
- **path `/app`:** default database.
- **query options:** pares `name=value` separados por `&`.

### Default database versus authSource

A default database determina o resultado de `client.db()`/database inicial quando não se indica nome. `authSource` determina a database contra a qual as credenciais são autenticadas em mecanismos como SCRAM. Nenhuma delas concede autorização: as roles do database user decidem que ações são permitidas.

### Percent-encoding

Caracteres reservados em username/password — como `$ : / ? # [ ] @` — têm significado na URI e devem ser percent-encoded. No Node.js, `encodeURIComponent()` codifica cada componente; não se deve codificar a URI inteira porque destruiria separadores.

### mongosh

`mongosh` é o shell atual. Executa JavaScript num ambiente próprio e fornece:

- métodos `db.collection.find()`, `aggregate()`, `createIndex()`;
- helpers como `show dbs`, `show collections` e `use <db>`;
- formatação de BSON e funções de administração.

O shell é adequado a exploração e diagnóstico. Migrações repetíveis devem estar versionadas e ser idempotentes; copiar histórico interativo não é uma estratégia de deployment.

> **Importante para o exame:** a database indicada no path da URI é a database usada inicialmente pelo client; `authSource` indica onde as credenciais são verificadas. São conceitos diferentes, e caracteres reservados nas credenciais da URI têm de ser percent-encoded.

---

## Funcionamento Interno

### Resolução SRV

Para `mongodb+srv://cluster.example.com`, o cliente consulta um registo DNS SRV do serviço MongoDB e pode consultar TXT para opções permitidas. Depois valida regras do domínio, liga-se às seeds e executa discovery. Problemas DNS, TLS e autenticação ocorrem em fases diferentes.

### Handshake e autenticação

Após TCP/TLS, o cliente envia metadata e negoceia capacidades. O mecanismo de autenticação prova a identidade; SCRAM não transmite simplesmente a password em claro. TLS protege confidencialidade/integridade em trânsito e autentica o servidor conforme a cadeia de certificados.

### Server selection

O driver mantém uma visão da topologia. Para cada operação, filtra servidores elegíveis por tipo, read preference, staleness/tags e janela de latência. `serverSelectionTimeoutMS` limita quanto tempo tenta encontrar um servidor adequado. Não é o mesmo que:

- `connectTimeoutMS`: abrir um socket individual;
- `socketTimeoutMS`: inatividade de socket;
- `maxTimeMS`: tempo de execução imposto à operação no servidor;
- `waitQueueTimeoutMS`: espera por uma ligação livre no pool.

### Cursor no shell

`find()` devolve cursor também no shell. `mongosh` apresenta automaticamente um batch inicial, o que pode criar a ilusão de que devolveu um array completo. Métodos como `sort()` e `limit()` configuram o cursor antes da iteração.

---

## Sintaxe

### Formato standard

```text
mongodb://[username:password@]host1[:port1][,hostN[:portN]]/[defaultAuthDb][?options]
```

Os parênteses retos nesta documentação significam “parte opcional”; não são caracteres que se copiam para a URI. `hostN` indica que podem existir vários seed hosts separados por vírgula. A `/` antes da database e o `?` antes das options são separadores estruturais.

### Formato SRV

```text
mongodb+srv://[username:password@]host[/[defaultAuthDb][?options]]
```

No formato SRV escreve-se um hostname de serviço e não uma lista de hosts/portas. O driver usa DNS para obter os seeds. A parte `defaultAuthDb` mantém o mesmo papel de database inicial; autenticar noutra database continua a exigir `authSource` quando aplicável.

### Opções frequentes

| Opção            | Significado                   | Nota                                    |
| ---------------- | ----------------------------- | --------------------------------------- |
| `replicaSet`     | nome esperado do replica set  | relevante no formato standard           |
| `authSource`     | database de autenticação      | não é autorização                       |
| `retryWrites`    | retry de writes elegíveis     | depende da topologia/operação           |
| `retryReads`     | retry de reads elegíveis      | não torna lógica arbitrária idempotente |
| `w`              | write concern acknowledgement | `majority` privilegia durabilidade      |
| `readPreference` | routing de reads              | default `primary`                       |
| `appName`        | identificação observável      | aparece em logs/profiling               |
| `tls`            | TLS                           | implicitamente true em SRV              |

### Invocar mongosh

```bash
mongosh "$MONGODB_URI"
```

Não colocar a URI literal no histórico do shell. Se a password tiver sido exposta, removê-la do ficheiro não apaga o histórico: deve ser rodada.

### Métodos essenciais em mongosh

```mongosh
db.getName()
db.getCollectionNames()
db.movies.find({ year: { $gte: 2000 } }).limit(5)
db.movies.findOne({ title: "The Matrix" })
db.movies.countDocuments({ genres: "Drama" })
db.movies.explain("executionStats").find({ title: "The Matrix" })
```

Estas linhas não devolvem todas o mesmo tipo de resultado:

| Expressão                                      | Resultado/efeito principal                                                      |
| ---------------------------------------------- | ------------------------------------------------------------------------------- |
| `db.getName()`                                 | string com o nome da database ativa                                             |
| `db.getCollectionNames()`                      | array de nomes de collections                                                   |
| `db.movies.find(filter).limit(5)`              | cursor configurado para no máximo cinco documentos                              |
| `db.movies.findOne(filter)`                    | um documento ou `null`                                                          |
| `db.movies.countDocuments(filter)`             | número exato de matches segundo a query                                         |
| `db.movies.explain(...).find(filter)`          | documento de diagnóstico do plano, não os filmes                                |

`db` é o handle da database ativa; `movies` é o nome da collection; os documentos dentro de `find()`/`findOne()` são filtros MQL. O shell pode imprimir automaticamente os primeiros resultados de um cursor, mas isso não transforma o cursor num array.

---

## Exemplos

### Exemplo 1 — construir uma URI standard sem corromper componentes

```javascript
/**
 * Constrói uma URI de laboratório a partir de componentes controlados.
 * Em produção, é preferível receber a URI completa de um secret manager.
 */
function buildMongoUri({ username, password, hosts, authSource }) {
    const encodedUsername = encodeURIComponent(username);
    const encodedPassword = encodeURIComponent(password);
    const seedList = hosts.join(",");
    const params = new URLSearchParams({
        replicaSet: "rs0",
        authSource,
        retryWrites: "true",
        w: "majority",
    });

    return (
        "mongodb://" +
        encodedUsername +
        ":" +
        encodedPassword +
        "@" +
        seedList +
        "/app?" +
        params.toString()
    );
}

const uri = buildMongoUri({
    username: "app-user",
    password: "p@ss:word/with?reserved",
    hosts: ["db1.example.net:27017", "db2.example.net:27017"],
    authSource: "admin",
});

console.log(uri);
```

Resultado: os caracteres reservados nas credenciais aparecem codificados; os separadores estruturais mantêm-se.

### Exemplo 2 — exploração segura em mongosh

```mongosh
use sample_mflix

const filter = {
  genres: "Drama",
  year: { $gte: 2010, $lte: 2020 },
  "imdb.rating": { $gte: 8 }
};

db.movies
  .find(filter, { _id: 0, title: 1, year: 1, "imdb.rating": 1 })
  .sort({ "imdb.rating": -1, title: 1 })
  .limit(10);
```

Resultado: até dez dramas do intervalo, com rating elevado, ordenados por rating e título.

### Exemplo 3 — diagnosticar um plano

```mongosh
use sample_mflix

const plan = db.movies
  .find({ title: "The Matrix" }, { _id: 0, title: 1, year: 1 })
  .explain("executionStats");

printjson({
  namespace: plan.queryPlanner.namespace,
  winningPlan: plan.queryPlanner.winningPlan,
  totalKeysExamined: plan.executionStats.totalKeysExamined,
  totalDocsExamined: plan.executionStats.totalDocsExamined,
  nReturned: plan.executionStats.nReturned
});
```

Resultado: resumo do plano e métricas reais dessa execução. Um `COLLSCAN` examina documentos; um `IXSCAN` usa índice, frequentemente seguido de `FETCH`.

### Exemplo 4 — ligação shell sem expor password no argumento

```bash
mongosh "mongodb+srv://cluster.example.net/sample_mflix" \
  --username "study-user" \
  --authenticationDatabase "admin"
```

Resultado: `mongosh` pede a password interativamente sem a incluir na URI nem no argumento. Para automação, usar um mecanismo e um secret manager suportados em vez de prompts.

---

## Explicação linha a linha

### Exemplo 1

1. A função recebe componentes explicitamente.
2. Codifica username e password separadamente.
3. Junta hosts, que já são valores internos validados.
4. `URLSearchParams` codifica os valores das opções.
5. A concatenação preserva `://`, `:`, `@`, `/` e `?` estruturais.
6. O exemplo mostra por que codificar a URI completa seria incorreto.

### Exemplo 2

1. `use` seleciona `sample_mflix` no shell.
2. Um único filter document combina igualdade, ranges e dot notation.
3. A projection é inclusiva e exclui `_id`.
4. O sort usa rating descendente e título como desempate.
5. O limit aplica-se antes de o shell esgotar o cursor.

### Exemplo 3

1. `explain("executionStats")` executa e recolhe métricas.
2. `queryPlanner` descreve a escolha do plano.
3. `totalKeysExamined` e `totalDocsExamined` quantificam trabalho.
4. `nReturned` permite comparar trabalho com resultado.
5. `printjson` limita a saída aos campos de diagnóstico relevantes.

### Exemplo 4

1. A URI não contém password.
2. Username e auth database são parâmetros distintos.
3. Ao receber username sem password numa sessão de terminal, `mongosh` solicita-a interativamente.
4. O secret não aparece no histórico como parte do comando.

---

## Casos Reais

- **Atlas com rotação de hosts:** SRV evita alterar a configuração da app.
- **Replica set self-managed:** URI standard lista várias seeds e `replicaSet`.
- **Incidente:** separar DNS, TCP/TLS, autenticação, autorização e query reduz diagnóstico por tentativa.
- **Script operacional:** `mongosh --file` executa um script versionado.
- **Performance:** `explain("executionStats")` confirma trabalho real antes de criar um índice.

---

## Performance

DNS SRV acrescenta resolução inicial, mas simplifica descoberta e failover. Pools e monitoring mantêm a topologia atualizada; não se deve resolver SRV manualmente e fixar o resultado.

Um timeout curto demais cria falsos negativos durante eleições; um timeout infinito prolonga falhas. Definir time budgets por fase e pelo SLO. `maxTimeMS` protege o servidor de operações longas, mas não cobre necessariamente toda a espera cliente, pool e rede.

No `explain`, comparar `totalDocsExamined` e `totalKeysExamined` com `nReturned`. Razões grandes sugerem baixa seletividade ou índice inadequado; zero docs examined pode indicar covered query.

---

## Armadilhas comuns

- **Codificar a URI inteira:** transforma separadores em dados.
- **Não codificar password:** `@`, `:` e `/` alteram parsing.
- **Confundir default database com `authSource`.**
- **Assumir que autenticação implica autorização.**
- **Usar `mongodb://` com um só membro de replica set:** reduz seeds para discovery.
- **Desativar TLS/validar certificados em produção:** permite ataques man-in-the-middle.
- **Colar secrets no terminal, logs ou screenshots.**
- **Confundir output automático do shell com array completo.**
- **Aplicar sort/limit depois de iterar um cursor:** configuração tardia não se aplica como esperado.
- **Interpretar `serverSelectionTimeoutMS` como timeout total da query.**

---

## O que costuma aparecer no exame

- Partes de uma connection string.
- `mongodb://` versus `mongodb+srv://`.
- Percent-encoding de credenciais.
- `authSource` versus default database.
- Seed list, discovery e replica set.
- Helpers `show`/`use` versus métodos de collection.
- Cursor devolvido por `find()`.
- Leitura de `explain`: `COLLSCAN`, `IXSCAN`, `FETCH` e métricas.
- Diferenças entre os principais timeouts.

---

## Resumo

SRV usa DNS para discovery e é o formato normal em Atlas; o formato standard lista seeds. Credenciais são componentes URI e exigem percent-encoding. A database do path, `authSource` e roles resolvem problemas diferentes. `mongosh` é excelente para exploração e explain, mas não é a API do driver. Diagnosticar uma ligação por fases evita confundir DNS, TLS, autenticação, autorização, pool e execução.

Fontes oficiais: [connection strings](https://www.mongodb.com/docs/manual/reference/connection-string/), [formatos](https://www.mongodb.com/docs/manual/reference/connection-string-formats/), [opções](https://www.mongodb.com/docs/manual/reference/connection-string-options/), [mongosh](https://www.mongodb.com/docs/mongodb-shell/) e [explain](https://www.mongodb.com/docs/manual/reference/explain-results/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — A database no path da URI seleciona o handle inicial; `authSource` indica onde as credenciais são autenticadas; as roles decidem autorização. Alterar uma destas partes não substitui as restantes.

> ### ARMADILHA
>
> Colocar uma password com `@`, `:`, `/` ou outros caracteres reservados diretamente na URI altera a interpretação dos componentes. Percent-encoding deve ser aplicado aos componentes individuais, não à connection string completa.

> ### DICA DE MEMORIZAÇÃO
>
> **SRV descobre; path escolhe; authSource autentica; role autoriza.**

> ### COMPARAÇÃO
>
> O esquema da URI muda discovery e defaults, não a linguagem de query.

| Aspeto       | `mongodb://`              | `mongodb+srv://`              |
| ------------ | ------------------------- | ----------------------------- |
| seeds        | explícitas na URI         | descobertas por DNS SRV       |
| opções DNS   | não                       | pode usar TXT                 |
| TLS default  | depende da opção/contexto | ativado por defeito           |
| porta na URI | pode ser explícita        | não se indica no hostname SRV |
| uso Atlas    | possível                  | formato habitual              |

> **Ligação entre capítulos:** Network Access e database users estão no capítulo 01; pools e timeouts do client no capítulo 04; `explain` e índices no capítulo 09.

### Mapa de interpretação da URI

```text
mongodb+srv://user:password@host/app?retryWrites=true&w=majority
|              |             |    |   `-- options
|              |             |    `------ default database
|              |             `----------- discovery host
|              `------------------------- credentials
`---------------------------------------- scheme

authSource = origem da autenticação, não autorização nem default database
```

### Mini desafio

Uma URI termina em `/sales?authSource=admin`. Explica que database `client.db()` usa quando o nome é omitido, onde as credenciais são verificadas e porque isso não prova que o utilizador pode escrever em `sales`.

---

## Resumo Rápido

- `mongodb://` usa seeds explícitas; `mongodb+srv://` usa DNS discovery.
- Credenciais reservadas exigem percent-encoding por componente.
- Default database, `authSource` e roles são conceitos diferentes.
- TLS, autenticação e autorização falham em fases diferentes.
- `mongosh` é shell interativo; o driver tem API própria.
- Secrets não pertencem ao código, histórico ou linha de comandos partilhada.

---

## Checklist

- [ ] Interpreto todos os componentes de uma URI.
- [ ] Distingo standard de SRV.
- [ ] Sei aplicar percent-encoding a username/password.
- [ ] Distingo default database de `authSource`.
- [ ] Distingo autenticação de autorização.
- [ ] Conheço o papel de TLS e das opções principais.
- [ ] Sei usar `mongosh` sem confundi-lo com o driver.
- [ ] Diagnostico DNS, TCP, TLS, auth e comando por ordem.

---

## Perguntas para confirmar conhecimentos

1. Como difere a descoberta de servidores entre `mongodb://` e `mongodb+srv://`?
2. Porque uma URI SRV normalmente não contém uma porta explícita?
3. Que componentes das credenciais exigem percent-encoding?
4. Porque não se deve codificar a URI inteira com `encodeURIComponent()`?
5. Qual é a diferença entre a database do path e `authSource`?
6. Que elemento determina as operações que o database user pode executar?
7. O que prova um `ping` bem-sucedido e o que não prova?
8. Porque scripts de migração não devem depender do histórico interativo de `mongosh`?
9. Como distingues uma falha de TLS de uma falha de autenticação?
10. Que risco existe em passar a connection string diretamente na linha de comandos?
