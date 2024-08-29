
+++
title= "RAG Chatbot: Mise en place du Projet (2/3)"
subtitle= "Mise en Place d'un Projet d'Indexation de Documents avec Pinecone et MongoDB"
author = 'writememory1337'
date= 2024-08-13
description= "Apprenez à mettre en place un pipeline complet pour l'indexation de documents PDF en utilisant Pinecone, MongoDB, et AWS S3."
tags= ["Pinecone", "MongoDB", "AWS S3", "OpenAI", "Node.js"]
featuredImage = "ada.jpg"
categories= ["documentation"]
index = false


+++

Dans cet article, nous détaillerons la mise en place du **RAG**, on y apprendra à 
1) **Extraire** du texte des fichiers **PDF** 
2) **Générer** des embeddings avec **OpenAI**
3) Les **Stocker** dans **Pinecone**.

J'ai réalisé plusieurs projets avec ces techos, certains utilisaient **Nuxt** d'autres **Next**. 

Cet article détaillera une utilisation plus générale.
Le but de l'article est simplement d'expliquer le fonctionnement des différentes librairies dont nous aurons besoin. Au lecteur d'ajuster le code à ses besoins (par exemple si vous avez implémenté l'**authentification** et la **multi-tenancy**). 

## Pré-requis

Avant de commencer, assurez-vous d'avoir installé et configuré les éléments suivants :

- **Node.js** et **npm**
- **MongoDB** pour stocker les métadonnées des documents
- **AWS S3** pour le stockage des fichiers PDF
- **Pinecone** pour l'indexation et la recherche vectorielle
- **OpenAI API** pour la génération des embeddings
- Un ficher **.env** avec toutes les uri et clés API nécessaires au fonctionnement de ces services
### 1. Configuration du Projet Node.js

Voilà de quoi nous aurons besoin
```bash
npm install express mongodb aws-sdk pdfjs-dist pinecone-client openai node-fetch h3
```
### 2. Extraction du Texte à partir des PDFs

L'une des étapes clés est l'extraction du texte à partir des fichiers PDF. Nous utilisons pdfjs-dist pour cela :

```js
import * as pdfjsLib from 'pdfjs-dist';

async function extractTextFromPDF(file) {
  const loadingTask = pdfjsLib.getDocument({ data: atob(file.split(',')[1]) });
  const pdf = await loadingTask.promise;
  let textContent = '';

  for (let i = 1; i <= pdf.numPages; i++) {
    const page = await pdf.getPage(i);
    const content = await page.getTextContent();
    const pageText = content.items
      .filter(item => item.str)
      .map(item => item.str)
      .join(' ');
    textContent += pageText + ' ';
  }

  return textContent.trim();
}
```

>Nous avons extrait de notre PDF des données exploitables.

### 3. Stockage des Fichiers dans AWS S3

Les fichiers PDF sont stockés dans un bucket AWS S3 pour une gestion centralisée des documents. Voici comment on configure cela :

```js
import AWS from 'aws-sdk';

const s3 = new AWS.S3({
  accessKeyId: process.env.AWS_ACCESS_KEY_ID,
  secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
  region: process.env.AWS_REGION,
});

const bucketName = process.env.S3_BUCKET_NAME!;

async function uploadToS3(file, fileName) {
  const s3Params = {
    Bucket: bucketName,
    Key: `uploads/${fileName}`,
    Body: Buffer.from(file.split(',')[1], 'base64'),
    ContentType: 'application/pdf',
  };
  const uploadResult = await s3.upload(s3Params).promise();
  return uploadResult.Location;
}
```

{{< admonition warning "Concernant S3 " >}}
La manière d'utiliser S3 avait changé entre 2 versions du projet, pensez à suivre son évolution pour éviter les erreurs.
{{< /admonition >}}

### 4. Stockage des Métadonnées dans MongoDB

Les informations sur chaque document, comme le nom du fichier, la date de téléchargement et l'URL S3, sont stockées dans MongoDB pour faciliter la gestion et la recherche des documents.

```js
import { MongoClient } from 'mongodb';

async function storeMetadataInMongoDB(fileName, textContent, fileUrl) {
  const client = new MongoClient(process.env.MONGODB_URI);
  await client.connect();
  const db = client.db('chatbot-rag');
  const collection = db.collection('documents');
  
  const result = await collection.insertOne({
    fileName,
    uploadDate: new Date(),
    content: textContent,
    fileUrl,
  });
  
  await client.close();
  return result.insertedId.toString();
}
```

Rien de particulier ici, MongoDB est très pratique à utiliser


### 5. Vérification et Création de Namespace dans Pinecone

Avant d'insérer des embeddings dans Pinecone, on vérifie si le namespace (index) existe, sinon, il faut le créer.

#
{{< admonition question "Pourquoi Pinecone ?" >}}
Pinecone est une BDD vectorielle performante, idéale pour la recherche de similarité, donc parfaite pour la recherche de documents.
{{< /admonition >}}

### 6. Génération et Insertion des Embeddings

Enfin, nous utilisons OpenAI pour générer des embeddings à partir du texte extrait et les insérons dans Pinecone pour permettre des recherches rapides et précises.

```js
import OpenAI from 'openai';

async function generateAndInsertEmbeddings(textChunks, documentId, fileName) {
  const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY! });

  const embeddings = await Promise.all(textChunks.map(async (chunk, index) => {
    const embeddingResponse = await openai.embeddings.create({
      model: 'text-embedding-ada-002',
      input: [chunk],
    });
    return {
      id: `${documentId}_${index}`,
      values: embeddingResponse.data[0].embedding,
      metadata: {
        fileName,
        chunkIndex: index,
        documentId,
      },
    };
  }));

//ajout des embeddings à Pinecone
  await pinecone.upsert({ vectors: embeddings, namespace: documentId });
      console.log('Embeddings ajourés à Pinecone');

}
```

## Conclusion

Notre pipeline pour l'extraction, le stockage, et l'indexation de documents PDF est complet. Reste à implémenter le processus de réponse du chatbot.

{{< admonition bug "Vous avez besoin d'aide ?" >}}
Je reste disponible (*quand le temps me le permet*) et vous pouvez me joindre par mail si vous avez des idées, des questions ou des difficultés à mettre en place votre projet.
{{< /admonition >}}
