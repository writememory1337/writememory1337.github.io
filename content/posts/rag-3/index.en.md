+++
title = "Créer un chatbot de type RAG (3/3) : Pour aller plus loin"
subtitle = "Chapitre 3 : Solutions Open Source et Améliorations Avancées"
author = 'writememory1337'
date = 2024-08-15
description = "Découvrez comment améliorer votre chatbot RAG avec des solutions open source, une meilleure sécurité et des fonctionnalités avancées."
tags = ["Foss","Ollama", "PostgreSQL", "pgvector"]
featuredImage = "vector3.jpg"
categories = ["documentation"]
index = false
+++
Dans cet article bonus à notre série sur les Chatbot RAG nous passerons en revue différentes possiblités d'amélioration. 

Selon les besoins de vos utilisateurs, il est possible de privilégier la **sécurité**, la **privacy** ou la **simplicité**.

Notez que certains modèles open source peuvent être assez gourmands et que pour héberger ces solutions en local, du matériel performant est requis.
## Transition vers des solutions open source

Certains de vos utilisateurs auront peut-être des contraintes spécifiques en matière de respect des données et de la vie privée, il est possible de déployer chez vos utilisateurs un système de Chatbot RAG indépendant d'openAI, de Pinecone et de MongoDB à l'aide, par exemple d'un conteneur faisant tourner **Ollama**, **PostgreSQL** avec **pgvector**, et le récent modèle **Llama3**.


## 1.Utiliser un modèle open source: Llama3 et ollama-js
Ollama-js est, selon moi, le moyen le plus simple d'intégrer différents modèles via [ollama](https://github.com/ollama/ollama) dans vos projets Javascript.

Une fois installé:
```bash
npm i ollama
```
son utilisation est très simple: 
```javascript
import ollama from 'ollama'

const response = await ollama.chat({
  model: 'llama3.1',
  messages: [{ role: 'user', content: 'Why is the sky blue?' }],
})
console.log(response.message.content)
```

#### Changer de modèle / Créer un modèle
{{<admonition tip "Exemple " >}} **ollama-js** permet d'utiliser différents modèles, meme des modèles de vision tels que **Llava** pour peu que vous les ayez installés.
{{< /admonition >}}

```javascript
import ollama from 'ollama'

const imagePath = './examples/multimodal/cat.jpg'
const response = await ollama.generate({
  model: 'llava',
  prompt: 'describe this image:',
  images: [imagePath],
  stream: true,
})
for await (const part of response) {
  process.stdout.write(part.response)
}
```
{{<admonition Example " Streamer les réponses" >}} Ollama-js propose l'option **stream responses** pour que la réponse soit affichée au fur et à mesure de la génération (idem ChatGPT ou Claude).
{{< /admonition >}}
```javascript
import ollama from 'ollama'

const message = { role: 'user', content: 'Why is the sky blue?' }
const response = await ollama.chat({ model: 'llama3.1', messages: [message], stream: true })
for await (const part of response) {
  process.stdout.write(part.message.content)
}
```

#### Aller plus loin 
Intégrer ollama dans vos Web Apps: cet article explique comment construire une Web App autour de ollama-js avec express.js

```javascript
import express from 'express';
import { Ollama } from 'ollama';

const app = express();
const ollama = new Ollama({ host: 'http://100.74.30.25:11434' });

app.get('/', async (req, res) => {
    try {
        const output = await ollama.generate({
            model: 'llama3:8b',
            prompt: "Give me a list of cities in the state of Wisconsin with a population of over 100,000 people in the form of a JSON array. It should look something like '[\"City 1\", \"City 2\", \"City 3\"]' in the output. Don't include any other text beyond the JSON."
        });

        // Parsing du JSON renvoyé vers un vrai tableau de villes
        const cityArray = JSON.parse(output.response);

        // Construction de l'<ul> depuis ce tableau
        let htmlList = '<ul>';
        for (let city of cityArray) {
            htmlList += `<li>${city}</li>`;
        }
        htmlList += '</ul>';

        res.send(htmlList);
    } catch (error) {
        res.status(500).send('Error processing your request: ' + error.message);
    }
});

const PORT = 8080;
app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

{{<admonition info "Info " >}}  À  chaque rechargemment de la page, un nouveau call api est fait, ce qui permettra d'avoir des répon,ses différentes à chaque fois (selon le prompt et si le modèle a + ou - tendance à halluciner. )
{{< /admonition >}}

Credits: [Blog](https://jws.dev/) de Joe Steinbring


### Base de données et Vectorisation

Plutôt que MongoDB et Pinecone, il est possible d'utiliser PostgreSQL avec l'extension
[pgvector](https://github.com/pgvector/pgvector-node) qui fournit différents exemples d'implémentation selon les technos utilisées par votre solution.

{{<admonition note  "Vérification " >}} Ce snippet vérifie que tout fonctionne correctement: la création de tables, l'insertion de données et la recherche optimisée par index {{< /admonition >}}


```javascript
import assert from 'node:assert';
import test from 'node:test';
import pg from 'pg';
import pgvector from 'pgvector/pg';
import { SparseVector } from 'pgvector';

test('pg example', async () => {
  const client = new pg.Client({database: 'pgvector_node_test'});
  await client.connect();

  await client.query('CREATE EXTENSION IF NOT EXISTS vector');
  await pgvector.registerTypes(client);

  await client.query('DROP TABLE IF EXISTS pg_items');
  await client.query('CREATE TABLE pg_items (id serial PRIMARY KEY, embedding vector(3), half_embedding halfvec(3), binary_embedding bit(3), sparse_embedding sparsevec(3))');

  const params = [
    pgvector.toSql([1, 1, 1]), pgvector.toSql([1, 1, 1]), '000', new SparseVector([1, 1, 1]),
    pgvector.toSql([2, 2, 2]), pgvector.toSql([2, 2, 2]), '101', new SparseVector([2, 2, 2]),
    pgvector.toSql([1, 1, 2]), pgvector.toSql([1, 1, 2]), '111', new SparseVector([1, 1, 2]),
    null, null, null, null
  ];
  await client.query('INSERT INTO pg_items (embedding, half_embedding, binary_embedding, sparse_embedding) VALUES ($1, $2, $3, $4), ($5, $6, $7, $8), ($9, $10, $11, $12), ($13, $14, $15, $16)', params);

  const { rows } = await client.query('SELECT * FROM pg_items ORDER BY embedding <-> $1 LIMIT 5', [pgvector.toSql([1, 1, 1])]);
  assert.deepEqual(rows.map(v => v.id), [1, 3, 2, 4]);
  assert.deepEqual(rows[0].embedding, [1, 1, 1]);
  assert.deepEqual(rows[0].half_embedding, [1, 1, 1]);
  assert.deepEqual(rows[0].binary_embedding, '000');
  assert.deepEqual(rows[0].sparse_embedding.toArray(), [1, 1, 1]);

  await client.query('CREATE INDEX ON pg_items USING hnsw (embedding vector_l2_ops)');

  await client.end();
});

test('pool', async () => {
  const pool = new pg.Pool({database: 'pgvector_node_test'});
  pool.on('connect', async function (client) {
    await client.query('CREATE EXTENSION IF NOT EXISTS vector');
    await pgvector.registerType(client);
  });

  await pool.query('DROP TABLE IF EXISTS pg_items');
  await pool.query('CREATE TABLE pg_items (id serial PRIMARY KEY, embedding vector(3))');

  const params = [
    pgvector.toSql([1, 1, 1]),
    pgvector.toSql([2, 2, 2]),
    pgvector.toSql([1, 1, 2]),
    null
  ];
  await pool.query('INSERT INTO pg_items (embedding) VALUES ($1), ($2), ($3), ($4)', params);

  const { rows } = await pool.query('SELECT * FROM pg_items ORDER BY embedding <-> $1 LIMIT 5', [pgvector.toSql([1, 1, 1])]);
  assert.deepEqual(rows.map(v => v.id), [1, 3, 2, 4]);
  assert.deepEqual(rows[0].embedding, [1, 1, 1]);
  assert.deepEqual(rows[1].embedding, [1, 1, 2]);
  assert.deepEqual(rows[2].embedding, [2, 2, 2]);

  await pool.query('CREATE INDEX ON pg_items USING hnsw (embedding vector_l2_ops)');

  await pool.end();
});
```


### Utilisation de sentence-transformers pour les embeddings

Pour information, plusieurs solutions d'embedding open source sont disponibles, BERT, GTE et ses différentes versions , Multilingual etc.

##### Installation

Commencez par installer Embeddings.js :

```bash
npm install @themaximalist/embeddings.js
```

Pour utiliser les embeddings locaux, installez également le modèle :

```bash
npm install @xenova/transformers
```

Nous ne détaillerons pas leur utilisation ici mais vous pouvez retrouver la doc [ici](https://embeddingsjs.themaximalist.com) ainsi qu'un super [article](https://blanchardjulien.com/posts/transformers_js/) de blog sur le sujet.

Jetez aussi un oeil à cette vidéo de [Coding Train](https://youtu.be/1mwguqeEz8c)







## 3. Utiliser Langchain

Une autre approche serait d'utiliser [`Langchain`](https://js.langchain.com/v0.1/docs/get_started/introduction)qui dispose d'une version Node.js, voir l'installation [`ici`](https://js.langchain.com/v0.1/docs/get_started/installation/).

{{<admonition tip "L'avantage" >}} Langchain fournit plusieurs classes toutes plus pratiques les unes que les autres, citons par exemple [YouTube-Transcripts](https://js.langchain.com/v0.2/docs/integrations/document_loaders/web_loaders/youtube/) dont le nom est assez explicite ou encore [GithubRepoLoader](https://js.langchain.com/v0.2/docs/integrations/document_loaders/web_loaders/github/)

{{< /admonition >}}

## Recommendations

Quelques liens vers des blogs et articles utiles aux curieux:

Un [framework](https://github.com/llm-tools/embedJs) à jour pour travailler avec différents LLMs et embeddings

Un [article](https://betterprogramming.pub/building-a-question-answer-bot-with-langchain-vicuna-and-sentence-transformers-b7f80428eadc) sur l'utilisation de **sentence-transformers** pour les embeddings.

Un autre [article](https://medium.com/credera-engineering/build-a-simple-rag-chatbot-with-langchain-b96b233e1b2a) sur la conception d'un chatbot rag (en python) avec pas mal d'explications

La récente version typescript de [LlamaIndex](https://ts.llamaindex.ai/)

Le [blog](https://jws.news/2024/how-to-write-a-javascript-app-that-uses-ollama/) de Joe et son article sur la conception d'une app JS intégrant Ollama.

 Dans les prochains articles, nous explorerons des techniques avancées d'optimisation et d'intégration pour créer un écosystème RAG complet et puissant.