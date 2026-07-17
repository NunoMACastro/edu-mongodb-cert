# MongoDB Atlas

## Objetivos do capítulo

- Compreender a hierarquia organization, project e database deployment.
- Configurar acesso de rede e identidade sem permissões excessivas.
- Distinguir database user de Atlas user.
- Reconhecer replica sets, sharding, regiões e tiers no contexto Atlas.
- Carregar e usar os sample datasets oficiais.

---

## Conceitos Fundamentais

MongoDB Atlas é o serviço gerido que provisiona e opera deployments MongoDB. O modelo mental correto separa **control plane** e **data plane**:

- O control plane configura organizações, projetos, utilizadores Atlas, billing, deployments, alertas e políticas.
- O data plane contém bases de dados, collections, documentos e database users que autenticam ligações.

O percurso da primeira operação ajuda a não misturar estes planos:

```text
Atlas user entra no portal
    -> cria/configura um project e um database deployment
    -> cria um database user
    -> autoriza a origem de rede da aplicação
    -> copia a connection string
    -> a aplicação autentica-se como database user
    -> executa comandos sobre databases e collections do data plane
```

O Atlas user participa na configuração. Não é enviado pelo Node.js Driver como identidade da aplicação. Depois da ligação, `sample_mflix` é uma database normal do deployment e `movies` uma collection; a aplicação não consulta “o projeto Atlas” através de `find()`.

### Hierarquia

| Nível               | Responsabilidade                                                               |
| ------------------- | ------------------------------------------------------------------------------ |
| Organization        | propriedade, equipas e faturação agregada                                      |
| Project             | fronteira operacional para deployments, database users, access lists e alertas |
| Database deployment | processos MongoDB, storage e topologia                                         |
| Database            | namespace lógico dentro do deployment                                          |
| Collection          | conjunto de documentos e índices                                               |

Um **Atlas user** entra no portal/API de gestão. Um **database user** autentica-se no MongoDB Server. Dar acesso ao projeto a alguém não cria automaticamente credenciais de base de dados, e criar um database user não dá acesso ao portal.

### Região, provider, tier e topologia

Ao criar um database deployment, o Atlas pede decisões de infraestrutura que não alteram a forma dos documentos, mas influenciam capacidade, latência, disponibilidade e custo:

| Conceito          | O que representa                                                                                          |
| ----------------- | --------------------------------------------------------------------------------------------------------- |
| Cloud provider    | plataforma de infraestrutura onde o deployment é executado                                                |
| Region            | localização geográfica principal dos recursos                                                             |
| Availability zone | localização isolada dentro de uma region, usada para reduzir falhas correlacionadas                       |
| Tier              | perfil de capacidade e limites, incluindo CPU, memória, storage e funcionalidades aplicáveis              |
| Topology          | organização dos processos como replica set ou sharded cluster                                             |
| Multi-region      | distribuição de membros por mais de uma region para resiliência ou proximidade, com trade-offs de latência |

Escolher uma region próxima da aplicação reduz normalmente o round-trip time. Um tier maior oferece mais recursos, mas não corrige um schema inadequado, uma query sem índice ou uma operação que transfere dados desnecessários. Multi-region acrescenta resiliência geográfica, mas pode aumentar latência e custo.

### Acesso em duas condições

Uma ligação Atlas exige normalmente:

1. **Identidade válida:** database user e mecanismo de autenticação apropriado.
2. **Origem de rede autorizada:** IP access list, peering, private endpoint ou outra configuração suportada.

Adicionar `0.0.0.0/0` permite ligações de qualquer origem IPv4 e deve ser apenas uma decisão temporária e conscientemente protegida. A password continua necessária, mas a camada de restrição de origem desaparece.

### Replica set versus sharding

| Conceito        | Resolve principalmente           | Unidade                           |
| --------------- | -------------------------------- | --------------------------------- |
| Replica set     | disponibilidade e redundância    | cópias do mesmo conjunto de dados |
| Sharded cluster | escala horizontal e distribuição | partições por shard key           |

Um shard é tipicamente ele próprio um replica set. Replicação não divide o dataset; sharding não substitui redundância. Reads secundárias podem aliviar certos workloads, mas trazem semântica de consistência e não aumentam a capacidade de escrita do primary de um replica set. A arquitetura, as shard keys e o routing são desenvolvidos no capítulo 10.

### Sample datasets

Atlas disponibiliza datasets como `sample_mflix`, `sample_restaurants`, `sample_supplies`, `sample_analytics` e `sample_training`. Os apontamentos preferem estes namespaces para que os exercícios sejam repetíveis. O carregamento falha se os namespaces de exemplo já existirem no deployment.

> **Importante para o exame:** um Atlas user administra recursos na plataforma; um database user autentica operações no MongoDB. Autorizar um endereço em Network Access também não substitui autenticação nem autorização.

---

## Funcionamento Interno

Num replica set, um membro primary aceita writes. Os membros secondary replicam o oplog. Eleições promovem um membro elegível quando o primary fica indisponível. O driver monitoriza a topologia e seleciona servidores; a aplicação não deve fixar manualmente um hostname de membro numa URI Atlas SRV.

Num sharded cluster, `mongos` encaminha operações. A shard key determina a distribuição por chunks/ranges. Uma query que inclui a shard key pode ser direcionada; sem ela, pode exigir scatter-gather. Atlas automatiza infraestrutura, mas não elimina decisões de data model, index design e shard key.

### SRV e descoberta

A URI `mongodb+srv://` usa registos DNS SRV para obter hosts e pode obter opções via TXT. Isto permite rotação de membros sem reconfigurar a aplicação. TLS é ativado por defeito no formato SRV. O driver continua a fazer descoberta e monitorização após a ligação inicial.

### Backups e disponibilidade

Alta disponibilidade, backup e disaster recovery são objetivos diferentes:

- Réplicas protegem disponibilidade perante falha de membro, mas replicam deletes acidentais.
- Backups permitem restaurar um estado anterior, mas têm RPO/RTO.
- Multi-region melhora resiliência geográfica, com custo e considerações de latência.

---

## Sintaxe

### URI Atlas SRV

```text
mongodb+srv://<username>:<password>@<cluster-host>/<default-database>?retryWrites=true&w=majority
```

- `username` e `password` são credenciais do database user.
- Caracteres reservados nas credenciais devem usar percent-encoding.
- `cluster-host` é o hostname SRV fornecido por Atlas.
- O path pode indicar a default database; não limita o acesso.
- `retryWrites=true` ativa retryable writes quando suportado.
- `w=majority` pede acknowledgement segundo a maioria de voting data-bearing members.

### Carregar sample data

No Atlas UI, selecionar o deployment e a ação **Load Sample Dataset**. Depois, validar em `mongosh`:

```mongosh
show dbs
use sample_mflix
show collections
db.movies.findOne({}, { title: 1, year: 1 })
```

Leitura da sessão:

- `show dbs` pede ao shell uma lista de databases visíveis ao utilizador;
- `use sample_mflix` altera o handle `db` ativo, mas não cria a database até existir uma escrita;
- `show collections` lista collections da database ativa;
- `db.movies` seleciona a collection `movies` através do handle ativo;
- `{}` é um filtro vazio, logo não restringe matches;
- `{ title: 1, year: 1 }` é a projection; `_id` continua incluído por defeito;
- `findOne()` devolve um documento ou `null`, não um cursor.

`show` e `use` são helpers de `mongosh`, não métodos do Node.js Driver.

---

## Exemplos

### Exemplo 1 — health check seguro

```javascript
import { MongoClient, ServerApiVersion } from "mongodb";

const uri = process.env.MONGODB_URI;

if (!uri) {
    throw new Error("MONGODB_URI não está definida.");
}

const client = new MongoClient(uri, {
    appName: "atlas-health-check",
    serverApi: {
        version: ServerApiVersion.v1,
        strict: true,
        deprecationErrors: true,
    },
    serverSelectionTimeoutMS: 5_000,
});

try {
    await client.connect();
    const result = await client.db("admin").command({ ping: 1 });
    console.log(result);
} finally {
    await client.close();
}
```

Resultado: tipicamente `{ ok: 1 }`, se rede, DNS, TLS, autenticação e autorização permitirem a operação.

### Exemplo 2 — inventariar os sample datasets

```javascript
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const admin = client.db("admin").admin();
    const { databases } = await admin.listDatabases({
        nameOnly: true,
        authorizedDatabases: true,
    });

    const sampleDatabases = databases
        .map(({ name }) => name)
        .filter((name) => name.startsWith("sample_"))
        .sort();

    console.log(sampleDatabases);
} finally {
    await client.close();
}
```

Resultado: lista ordenada das databases `sample_*` que o utilizador pode listar.

### Exemplo 3 — primeira leitura sobre `sample_supplies`

```javascript
import { MongoClient } from "mongodb";

const client = new MongoClient(process.env.MONGODB_URI);

try {
    const sales = client.db("sample_supplies").collection("sales");
    const recentSales = await sales
        .find(
            { purchaseMethod: "Online" },
            {
                projection: {
                    _id: 0,
                    saleDate: 1,
                    storeLocation: 1,
                    customer: 1,
                },
            },
        )
        .sort({ saleDate: -1 })
        .limit(5)
        .toArray();

    console.dir(recentSales, { depth: null });
} finally {
    await client.close();
}
```

Resultado: até cinco vendas online, mais recentes primeiro.

---

## Explicação linha a linha

### Exemplo 1

1. Importa cliente e enum da Stable API.
2. Valida o secret antes de tentar ligar.
3. `appName` identifica a aplicação em observabilidade.
4. `serverApi.strict` rejeita comandos fora da Stable API declarada quando aplicável.
5. `serverSelectionTimeoutMS` limita a fase de seleção, não a duração de todas as queries.
6. `connect()` força o handshake; `ping` prova uma round trip.
7. `finally` encerra recursos deste script de curta duração.

### Exemplo 2

1. `db("admin").admin()` obtém a interface administrativa.
2. `nameOnly` evita detalhes desnecessários.
3. `authorizedDatabases` restringe a resposta ao que a identidade pode ver.
4. `map` extrai nomes, `filter` mantém samples e `sort` ordena no processo Node.

### Exemplo 3

1. Obtém a collection oficial `sample_supplies.sales`.
2. O filtro usa igualdade sobre `purchaseMethod`.
3. A projection reduz o payload.
4. Sort descendente coloca datas maiores primeiro.
5. Limit evita materializar mais de cinco documentos.
6. `toArray()` é seguro aqui por existir um limite pequeno.

---

## Casos Reais

- **SaaS multiambiente:** projetos separados para produção e não-produção reduzem blast radius.
- **Aplicação em cloud:** private endpoints evitam exposição pública e simplificam controlo de egress.
- **Catálogo global:** nós em várias regiões podem aproximar reads, desde que a read preference aceite a semântica.
- **Auditoria:** Atlas users gerem infraestrutura; database users seguem least privilege para workloads.
- **Recuperação:** backups point-in-time cobrem erros lógicos que a replicação propagaria.

---

## Performance

Escolher a região próxima da aplicação reduz round-trip time. Uma operação com várias viagens à base de dados amplifica latência; embedding e bulk operations podem reduzir round trips. Multi-region e cross-region reads/writes têm compromissos entre latência, durabilidade e consistência.

O tier define CPU, RAM, storage e limites. Um working set que não cabe em cache aumenta I/O. Escalar hardware sem corrigir `COLLSCAN`, índices redundantes ou schema inadequado apenas adia o problema.

Num sharded cluster, queries targeted escalam melhor do que scatter-gather. A shard key deve combinar cardinalidade, frequência, monotonicidade e padrões de query. Esta matéria ultrapassa parte do exame Associate, mas o princípio é relevante.

---

## Armadilhas comuns

- **Confundir Atlas user com database user:** autenticam planos diferentes.
- **Permitir `0.0.0.0/0` permanentemente:** aumenta desnecessariamente a superfície de ataque.
- **Dar `atlasAdmin` à aplicação:** viola least privilege.
- **Codificar a password na URI dentro do repositório:** segredos ficam no histórico.
- **Não fazer percent-encoding:** caracteres como `@` ou `:` quebram a URI.
- **Assumir que o nome da database na URI restringe permissões:** autorização vem das roles.
- **Confundir replica set com backup:** uma eliminação legítima replica-se.
- **Fixar um host de membro:** perde descoberta e failover do SRV.
- **Assumir que Atlas corrige queries:** operação gerida não substitui bons índices.

---

## O que costuma aparecer no exame

- Sequência: criar deployment, database user e network access.
- Diferença entre Atlas user e database user.
- Função de organization, project, database, collection.
- SRV versus URI standard.
- Replica set versus sharded cluster.
- Razão de sample datasets e namespaces corretos.
- Consequências de IP access list e least privilege.
- Primary, secondary e eleição.
- Porque replicação não é backup.

---

## Resumo

Atlas gere MongoDB na cloud, mas a aplicação continua a ligar-se com uma identidade de base de dados e uma origem de rede permitida. A hierarquia organization → project → deployment separa governação. Replica sets oferecem redundância; sharding distribui dados. SRV simplifica descoberta. Segurança exige secrets externos, roles mínimas e rede limitada. Performance depende de região, tier, working set, schema, índices e targeting.

Fontes oficiais: [Get Started with Atlas](https://www.mongodb.com/docs/atlas/getting-started/), [configurar acesso](https://www.mongodb.com/docs/atlas/security/quick-start/), [connection strings](https://www.mongodb.com/docs/manual/reference/connection-string/), [sample datasets](https://www.mongodb.com/docs/atlas/sample-data/), [replication](https://www.mongodb.com/docs/manual/replication/) e [sharding](https://www.mongodb.com/docs/manual/sharding/).

---

## Preparação orientada ao exame

> ### IMPORTANTE PARA O EXAME
>
> **Probabilidade: Muito Alta** — Uma aplicação Atlas precisa de três condições independentes: deployment disponível, origem de rede autorizada e database user com roles adequadas. Falhar uma delas impede a operação, mas cada falha ocorre numa camada diferente.

> ### ARMADILHA
>
> Criar um Atlas user não cria automaticamente credenciais para a connection string. Do mesmo modo, adicionar um IP à access list não concede permissões de leitura ou escrita. A rede permite chegar; a autenticação identifica; a autorização limita ações.

> ### DICA DE MEMORIZAÇÃO
>
> **Chegar → Entrar → Agir:** Network Access deixa chegar, database user permite entrar e roles determinam como agir.

> ### COMPARAÇÃO
>
> Identidade, rede e arquitetura resolvem problemas diferentes.

| Conceito        | Resolve                          | Não garante                        |
| --------------- | -------------------------------- | ---------------------------------- |
| Atlas user      | administração da plataforma      | login da aplicação na database     |
| Database user   | autenticação no MongoDB          | origem de rede permitida           |
| Network Access  | alcance de rede                  | autorização sobre collections      |
| Replica set     | redundância/alta disponibilidade | distribuição horizontal do dataset |
| Sharded cluster | distribuição do dataset          | backup histórico                   |

> **Ligação entre capítulos:** connection strings e `authSource` aparecem no capítulo 03; `MongoClient` e TLS no 04; índices no 09; sharded clusters, shard keys e targeting no 10.

### Fluxo de diagnóstico Atlas

```text
A aplicação chega ao hostname?
  |-- não --> DNS / região / Network Access
  `-- sim --> TLS completa?
                |-- não --> certificado / proxy / opções TLS
                `-- sim --> autentica?
                              |-- não --> database user / password / authSource
                              `-- sim --> operação autorizada?
                                            |-- não --> roles
                                            `-- sim --> analisar query/performance
```

### Mini desafio

Uma aplicação consegue resolver o hostname e abrir TLS, mas recebe “authentication failed”. Indica quais configurações Atlas são relevantes e quais não corrigem a causa apenas por serem alteradas.

---

## Resumo Rápido

- Organization contém projects; project contém deployments e configuração.
- Atlas user não é database user.
- Network Access não substitui autenticação nem roles.
- Replica set replica; sharding distribui.
- SRV facilita discovery e ativa TLS por defeito.
- Região, tier, schema e índices influenciam performance.

---

## Checklist

- [ ] Distingo organization, project e deployment.
- [ ] Distingo Atlas user de database user.
- [ ] Sei separar Network Access, autenticação e autorização.
- [ ] Explico replica set versus sharded cluster.
- [ ] Sei por que replicação não é backup.
- [ ] Reconheço a função de uma URI SRV.
- [ ] Sei aplicar least privilege às roles.
- [ ] Consigo diagnosticar uma falha Atlas por camadas.

---

## Perguntas para confirmar conhecimentos

1. Que recursos pertencem normalmente a uma organization, project e deployment?
2. Porque um Atlas user não deve ser usado como database user da aplicação?
3. O que acontece se a origem estiver autorizada, mas as credenciais estiverem erradas?
4. O que acontece se as credenciais forem válidas, mas a role não permitir o comando?
5. Qual é a diferença principal entre replica set e sharded cluster?
6. Porque a replicação não substitui um backup?
7. Que informação pode ser descoberta através de DNS numa URI SRV?
8. Que fatores Atlas e de aplicação influenciam a latência além do tier?
9. Porque uma regra de Network Access demasiado ampla aumenta risco mesmo com password forte?
10. Como isolarias DNS, TLS, autenticação e autorização numa pergunta de diagnóstico?
