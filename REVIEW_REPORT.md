# Relatório de revisão técnica

## Identificação

- **Material:** MongoDB Associate Developer (Node.js)
- **Data da revisão:** 16 de julho de 2026
- **Ficheiros de estudo revistos:** 15
- **Âmbito:** conteúdo técnico, exemplos, APIs, performance, segurança, consistência, terminologia, formatação e preparação para exame
- **Baseline:** MongoDB Node.js Driver 7.5, Node.js 20.19.0 ou superior e documentação MongoDB corrente

Este relatório é o único ficheiro novo criado, por ser o artefacto final pedido. Os apontamentos existentes não foram reescritos: as alterações são localizadas e preservam a estrutura, o nível técnico e o estilo global.

> **Nota de estado — 16 de julho de 2026:** este relatório documenta a revisão técnica concluída antes do enriquecimento pedagógico para certificação. As contagens de secções, separadores, tabelas, palavras e blocos abaixo pertencem a esse snapshot. O estado posterior do manual, incluindo novas caixas, mapas, checklists e perguntas, está documentado em [`CERTIFICATION_ENHANCEMENT_REPORT.md`](./CERTIFICATION_ENHANCEMENT_REPORT.md). O conteúdo técnico validado foi preservado.

> **Nota posterior — 17 de julho de 2026:** foi criado `10-Sharding.md` e os antigos capítulos 10–13 foram renumerados para 11–14. A tabela abaixo usa os nomes atuais para manter os links legíveis, mas as métricas 15/15 continuam a descrever o snapshot anterior à criação de sharding. A validação do conjunto atual de 16 capítulos está no relatório de enriquecimento.

> **Nota de reconstrução pedagógica — 17 de julho de 2026:** depois deste snapshot técnico, os 16 capítulos receberam uma revisão estrutural integral. A afirmação histórica abaixo de que “os apontamentos não foram reescritos” já não descreve o estado pedagógico atual: conceitos e sintaxe foram reordenados e ampliados para construir primeiro o modelo mental. O âmbito e as métricas atuais estão em [`CERTIFICATION_ENHANCEMENT_REPORT.md`](./CERTIFICATION_ENHANCEMENT_REPORT.md).

## Resultado global

O conjunto estava tecnicamente sólido e já usava APIs modernas. A revisão não encontrou exemplos ativos baseados em callbacks do driver, `collection.count()`, `cursor.count()`, `remove()`, `save()` ou `update()` legado. As referências a APIs antigas permanecem apenas quando servem para avisar que não devem ser usadas.

Foram corrigidas duas lacunas técnicas materiais, um problema de lifecycle num exemplo Node.js, referências antigas da documentação de Search e a ausência transversal das notas pedidas para o exame. Não ficaram contradições técnicas conhecidas entre capítulos após a revisão.

## Ficheiros revistos

| Ficheiro | Foco da revisão | Alteração |
|---|---|---|
| `00-Introducao.md` | baseline, camadas, tipos de retorno e atomicidade | nota de exame sobre retorno, cursor/result object e fronteira atómica |
| `01-Atlas.md` | control plane, database users, Network Access, SRV e TLS | distinção examinável entre Atlas user, database user e acesso de rede |
| `02-Document-Model.md` | BSON, `ObjectId`, embedding, references e limite documental | reforço de comparação por tipo e limite BSON de 16 MiB |
| `03-Connection-Strings-e-MongoShell.md` | URI, SRV, percent-encoding, `authSource` e `mongosh` | distinção entre default database, autenticação e autorização |
| `04-Connecting-From-NodeJS.md` | `MongoClient`, pools, timeouts, retries e shutdown | correção significativa do pool e do graceful shutdown |
| `05-CRUD-Insert-Find.md` | inserts, filtros, arrays, cursor e tipos BSON | reforço de `find()`/`findOne()` e `$elemMatch` |
| `06-CRUD-Update-Replace-Delete.md` | update operators, replace, upsert, delete e contadores | reforço de replacement e `matchedCount`/`modifiedCount` |
| `07-CRUD-Sort-Limit-Projection.md` | sort, projection, paginação, batch e counts | reforço das regras de projection e de `limit()`/`batchSize()` |
| `08-CRUD-NodeJS.md` | result objects, compound operations, bulk e erros | semântica atual de `includeResultMetadata` destacada |
| `09-Indexes.md` | compound, ESR, multikey, coverage e `explain` | restrições multikey e uso de `$elemMatch` em index bounds reforçados |
| `11-Aggregation.md` | stages, cardinalidade, blocking, otimização e memória | correção significativa dos limites próprios de `$facet` |
| `12-Aggregation-NodeJS.md` | `AggregationCursor`, batches, BSON e builders | reforço de cursor, `toArray()` e iteração progressiva |
| `13-Transactions.md` | sessions, retries, concerns, side effects e contenção | ausência de paralelismo e passagem da mesma session destacadas |
| `14-Atlas-Search.md` | search indexes, mappings, analyzers, scoring e operadores | distinção de índices reforçada e links oficiais atualizados |
| `99-Resumo-Final.md` | coerência transversal, tabelas, checklist e revisão | nota sobre uso legítimo das prioridades de exame |

## Principais correções

### 1. Contabilização de ligações no pool

O capítulo 04 apresentava corretamente `maxPoolSize` como limite por pool, mas não explicitava que o `MongoClient` pode manter até duas ligações adicionais de monitorização por servidor da topologia. A omissão podia levar a estimativas incorretas do total de sockets.

Foi acrescentada a distinção entre ligações usadas por operações e ligações de monitorização. Mantém-se também o alerta para calcular capacidade por processo, pool e servidor, em vez de aumentar `maxPoolSize` isoladamente.

### 2. Graceful shutdown do servidor HTTP

O exemplo anterior chamava `server.close()` e fechava imediatamente o `MongoClient`, sem aguardar a callback de fecho do servidor. Pedidos ainda ativos podiam, por isso, perder acesso ao client partilhado.

O exemplo passou a:

- impedir execuções concorrentes do shutdown;
- deixar de aceitar novos pedidos;
- aguardar o fecho do servidor e dos pedidos ativos;
- fechar depois o `MongoClient`;
- preservar falhas através de `process.exitCode`;
- tratar `SIGINT` e `SIGTERM` sem criar uma Promise ignorada pelo event emitter.

### 3. Limites de `$facet`

A explicação geral de spill para disco estava correta, mas demasiado ampla para `$facet`. Foi documentada a exceção:

- o documento produzido durante `$facet` está limitado a 100 MB;
- `$facet` não pode fazer spill para disco;
- `allowDiskUse` não remove este limite;
- o documento final continua sujeito ao limite BSON de 16 MiB.

Esta correção impede a inferência incorreta de que `allowDiskUse: true` resolve qualquer limite de memória de aggregation.

### 4. Referências atuais de MongoDB Search

Os links antigos sob `/docs/atlas/atlas-search/` foram substituídos pelos caminhos correntes sob `/docs/search/` para a página principal, pesquisa, query reference, analyzers e `autocomplete`. A terminologia mantém “Atlas Search” no título apenas como designação histórica reconhecível, explicando que o nome corrente é MongoDB Search.

### 5. Notas de exame

Nenhum dos 15 ficheiros tinha o destaque literal pedido. Foi acrescentada uma nota `Importante para o exame`, contextual e não repetitiva, em cada capítulo. As notas esclarecem distinções que geram erros frequentes sem alegar acesso a perguntas reais ou informação confidencial.

## Conceitos aprofundados

Foram reforçados, no respetivo contexto:

- documento, cursor e result object;
- atomicidade single-document versus transação multi-documento;
- Atlas user, database user e Network Access;
- igualdade por valor e tipo em BSON, incluindo `ObjectId`;
- default database versus `authSource`;
- pool de operações versus sockets de monitorização;
- `find()` versus `findOne()`;
- `updateOne()` versus `replaceOne()`;
- `matchedCount` versus `modifiedCount`;
- projection inclusiva/exclusiva;
- `limit()` versus `batchSize()`;
- retorno atual de `findOneAnd*()`;
- compound indexes, multikey restrictions e `$elemMatch`;
- stage, expression e accumulator;
- limites próprios de `$facet`;
- `AggregationCursor` versus array materializado;
- session, retries e proibição de paralelismo numa transação;
- B-tree index, text index clássico e MongoDB Search index.

## APIs e compatibilidade

### Confirmadas como atuais

- ES Modules e top-level `await` em ficheiros `.mjs` ou projetos com `"type": "module"`;
- `MongoClient` partilhado e API baseada em Promises;
- `countDocuments()` e `estimatedDocumentCount()`;
- `findOneAndUpdate()` com `returnDocument` e `includeResultMetadata`;
- `bulkWrite()` e os result objects atuais;
- `AggregationCursor` com `for await...of` e `toArray()`;
- `ClientSession.withTransaction()`;
- `createSearchIndex()` e aggregation com `$search`/`$searchMeta`.

### APIs obsoletas

Não foi necessário substituir APIs obsoletas em código ativo. `cursor.count()` e `collection.count()` aparecem apenas numa explicação que as identifica como antigas/deprecated. Não foram encontradas utilizações ativas dos métodos legados pesquisados.

## Consistência e formatação

Todos os capítulos mantêm a mesma sequência de secções:

1. Objetivos do capítulo
2. Conceitos Fundamentais
3. Funcionamento Interno
4. Sintaxe
5. Exemplos
6. Explicação linha a linha
7. Casos Reais
8. Performance
9. Armadilhas comuns
10. O que costuma aparecer no exame
11. Resumo

Foram preservados os títulos, separadores, tabelas, code fences, termos técnicos em inglês e estilo de português europeu. Cada ficheiro contém agora pelo menos uma nota com o mesmo formato.

## Validação executada

- **15/15 ficheiros** com a estrutura de 11 secções esperada.
- **15/15 ficheiros** com exatamente 10 separadores principais.
- **15/15 ficheiros** com pelo menos uma nota `Importante para o exame`.
- **103/103 blocos JavaScript** aprovados por `node --check` com Node.js v24.17.0.
- Fences de código equilibradas em todos os ficheiros.
- Nenhum trailing whitespace detetado.
- Pesquisa por métodos CRUD legados sem utilizações ativas.
- **81 links técnicos** limitados a `mongodb.com`, `mongodb.github.io` e `learn.mongodb.com`.

A validação de JavaScript foi sintática e a compatibilidade das APIs foi confrontada com documentação oficial. Os exemplos não foram executados contra um cluster real, porque isso exigiria credenciais, dados, índices e configuração de topologia externos. Resultados dependentes de `sample_mflix`, `sample_supplies`, transações ou MongoDB Search devem ser confirmados num deployment de laboratório antes de uma publicação executável.

Na reconstrução pedagógica posterior, a validação sintática subiu para **142/142 blocos JavaScript**, os 16 capítulos mantiveram fences equilibradas e os 15 links internos foram resolvidos sem destinos em falta. Estes números pertencem ao estado atual e não alteram as métricas históricas acima.

## Capítulos com alterações significativas

### `04-Connecting-From-NodeJS.md`

Alteração significativa por corrigir lifecycle concorrente no shutdown e completar o modelo de capacidade dos pools.

### `11-Aggregation.md`

Alteração significativa por documentar a exceção de memória de `$facet` e a relação correta com `allowDiskUse` e o limite BSON.

### `14-Atlas-Search.md`

Alteração moderada por atualizar o encaminhamento documental e tornar explícita a separação entre database indexes e search indexes.

Nos restantes capítulos, as alterações são deliberadamente pequenas: validação técnica, reforço examinável e uniformização, sem reescrita do conteúdo correto.

## Pontos para nova revisão futura

1. Confirmar a release corrente do Node.js Driver e o requisito mínimo de Node.js antes de cada edição/publicação.
2. Rever alterações de retorno, opções e tipos nas release notes de futuras versões major do driver.
3. Confirmar alterações ao formato, duração e blueprint público do exame na MongoDB University.
4. Rever a evolução de MongoDB Search, sobretudo nomes, disponibilidade por deployment, field types e paginação.
5. Executar os exemplos num cluster de laboratório com sample datasets e capturar `explain("executionStats")` representativo.
6. Testar transações num replica set e Search com índices em estado `READY`; estes comportamentos não podem ser demonstrados num standalone sem configuração própria.
7. Revalidar limites operacionais que podem depender de versão, tier Atlas ou configuração do servidor.

## Fontes oficiais principais

- [MongoDB Node.js Driver](https://www.mongodb.com/docs/drivers/node/current/)
- [Release notes do Node.js Driver](https://www.mongodb.com/docs/drivers/node/current/reference/release-notes/)
- [MongoDB Manual](https://www.mongodb.com/docs/manual/)
- [Connection pools](https://www.mongodb.com/docs/drivers/node/current/connect/connection-options/connection-pools/)
- [CRUD compound operations](https://www.mongodb.com/docs/drivers/node/current/crud/compound-operations/)
- [Aggregation pipeline optimization](https://www.mongodb.com/docs/manual/core/aggregation-pipeline-optimization/)
- [`$facet`](https://www.mongodb.com/docs/manual/reference/operator/aggregation/facet/)
- [Transactions no Node.js Driver](https://www.mongodb.com/docs/drivers/node/current/crud/transactions/)
- [MongoDB Search](https://www.mongodb.com/docs/search/)
- [MongoDB University — Node.js Developer Path](https://learn.mongodb.com/learning-paths/mongodb-nodejs-developer-path)
