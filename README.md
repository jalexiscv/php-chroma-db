<p align="center"><img src="./art/header.png"></p>

# ChromaDB PHP API Client

[![Latest Version on Packagist](https://img.shields.io/packagist/v/helgesverre/chromadb.svg?style=flat-square)](https://packagist.org/packages/helgesverre/chromadb)
[![Total Downloads](https://img.shields.io/packagist/dt/helgesverre/chromadb.svg?style=flat-square)](https://packagist.org/packages/helgesverre/chromadb)

[ChromaDB](https://github.com/chroma-core/chroma) is an open-source vector database that allows you to store and query
vector embeddings. This package provides a PHP client for the ChromaDB API.

## Installation

You can install the package via composer:

```bash
composer require helgesverre/chromadb
```

You can publish the config file with:

```bash
php artisan vendor:publish --tag="chromadb-config"
```

This is the contents of the published `config/chromadb.php` file:

```php
return [
    'token' => env('CHROMADB_TOKEN'),
    'host' => env('CHROMADB_HOST', 'localhost'),
    'port' => env('CHROMADB_PORT', '19530'),
];
```

## Usage

```php
$chromadb = new \HelgeSverre\Chromadb\Chromadb(
    token: 'test-token-chroma-local-dev',
    host: 'http://localhost',
    port: '8000'
);

// Create a new collection with optional metadata
$chromadb->collections()->create(
    name: 'my_collection',
);

// Count the number of collections
$chromadb->collections()->count();

// Retrieve a specific collection by name
$chromadb->collections()->get(
    collectionName: 'my_collection'
);

//Para obtener el ID de una colección conociendo su nombre, puedes usar el método get del recurso collections y luego acceder al ID de la colección desde la respuesta.
$collectionName = 'my_collection';
$collection = $chromadb->collections()->get(collectionName: $collectionName);
$collectionId = $collection->json('id');
echo "Collection ID: " . $collectionId;


// Delete a collection by name
$chromadb->collections()->delete(
    collectionName: 'my_collection'
);

// Update a collection's name and/or metadata
$chromadb->collections()->update(
    collectionId: '3ea5a914-e2ab-47cb-b285-8e585c9af4f3',
    newName: 'new_collection_name',
);

// Add items to a collection with optional embeddings, metadata, and documents
$chromadb->items()->add(
    collectionId: '3ea5a914-e2ab-47cb-b285-8e585c9af4f3',
    ids: ['item1', 'item2'],
    embeddings: ['embedding1', 'embedding2'],
    documents: ['doc1', 'doc2']
);

// Update items in a collection with new embeddings, metadata, and documents
$chromadb->items()->update(
    collectionId: '3ea5a914-e2ab-47cb-b285-8e585c9af4f3',
    ids: ['item1', 'item2'],
    embeddings: ['new_embedding1', 'new_embedding2'],
    documents: ['new_doc1', 'new_doc2']
);

// Upsert items in a collection (insert if not exist, update if exist)
$chromadb->items()->upsert(
    collectionId: '3ea5a914-e2ab-47cb-b285-8e585c9af4f3',
    ids: ['item'],
    metadatas: [['title' => 'metadata']],
    documents: ['document']
);

// Retrieve specific items from a collection by their IDs
$chromadb->items()->get(
    collectionId: '3ea5a914-e2ab-47cb-b285-8e585c9af4f3',
    ids: ['item1', 'item2']
);

// Delete specific items from a collection by their IDs
$chromadb->items()->delete(
    collectionId: '3ea5a914-e2ab-47cb-b285-8e585c9af4f3',
    ids: ['item1', 'item2']
);

// Count the number of items in a collection
$chromadb->items()->count(
    collectionId: '3ea5a914-e2ab-47cb-b285-8e585c9af4f3'
);

// Query items in a collection based on embeddings, texts, and other filters
$chromadb->items()->query(
    collectionId: '3ea5a914-e2ab-47cb-b285-8e585c9af4f3',
    queryEmbeddings: [createTestVector(0.8)],
    include: ['documents', 'metadatas', 'distances'],
    nResults: 5
);

```

## Example: Semantic Search with ChromaDB and OpenAI Embeddings

This example demonstrates how to perform a semantic search in ChromaDB using embeddings generated from OpenAI.

Full code available in [SemanticSearchTest.php](./tests/Feature/SemanticSearchTest.php).

### Prepare Your Data

First, create an array of data you wish to index. In this example, we'll use blog posts with titles, summaries, and
tags.

```php
$blogPosts = [
    [
        'title' => 'Exploring Laravel',
        'summary' => 'A deep dive into Laravel frameworks...',
        'tags' => ['PHP', 'Laravel', 'Web Development']
    ],
    [
        'title' => 'Introduction to React',
        'summary' => 'Understanding the basics of React and how it revolutionizes frontend development.',
        'tags' => ['JavaScript', 'React', 'Frontend']
    ],
];
```

### Generate Embeddings

Use OpenAI's embeddings API to convert the summaries of your blog posts into vector embeddings.

```php
$summaries = array_column($blogPosts, 'summary');
$embeddingsResponse = OpenAI::client('sk-your-openai-api-key')
    ->embeddings()
    ->create([
        'model' => 'text-embedding-ada-002',
        'input' => $summaries,
    ]);

foreach ($embeddingsResponse->embeddings as $embedding) {
    $blogPosts[$embedding->index]['vector'] = $embedding->embedding;
}
```

### Create ChromaDB Collection

Create a collection in ChromaDB to store your blog post embeddings.

```php
$createCollectionResponse = $chromadb->collections()->create(
    name: 'blog_posts',
);

$collectionId = $createCollectionResponse->json('id');
```

### Insert into ChromaDB

Insert these embeddings, along with other blog post data, into your ChromaDB collection.

```php
foreach ($blogPosts as $post) {
    $chromadb->items()->add(
        collectionId: $collectionId,
        ids: [$post['title']],
        embeddings: [$post['embedding']],
        metadatas: [$post]
    );
}
```

### Creating a Search Vector with OpenAI

Generate a search vector for your query, akin to how you processed the blog posts.

```php
$searchEmbedding = getOpenAIEmbedding('laravel framework');
```

### Searching using the Embedding in ChromaDB

Use the ChromaDB client to perform a search with the generated embedding.

```php
$searchResponse = $chromadb->items()->query(
    collectionId: $collectionId,
    queryEmbeddings: [$searchEmbedding],
    nResults: 3,
    include: ['metadatas']
);

// Output the search results
foreach ($searchResponse->json('results') as $result) {
    echo "Title: " . $result['metadatas']['title'] . "\n";
    echo "Summary: " . $result['metadatas']['summary'] . "\n";
    echo "Tags: " . implode(', ', $result['metadatas']['tags']) . "\n\n";
}
```

## Ver todos los documentos(items) en una colección
```php
$collection = $collections->get(collectionName: $collection_name);
$collection_id = $collection->json('id');
$all_items = [];
$offset = 0;
$limit = 5;
$items = $chromadb->items()->get(collectionId: $collection_id,ids: array(),include: ['documents', 'metadatas'],limit: $limit,offset: $offset,);
$data = $items->json();
if($data !== null && isset($data['ids'])){
    $all_items = array_merge($all_items, $data['ids']);
    //$all_documents = array_merge($all_documents, $data['documents']??[]);
    //$all_metadatas = array_merge($all_metadatas, $data['metadatas']??[]);
    $offset += $limit;
}else{
    echo "No se encontraron items";
}

```


## Running ChromaDB in Docker

To quickly get started with ChromaDB, you can run it in Docker

```bash
# Download the docker-compose.yml file
wget https://github.com/HelgeSverre/chromadb/blob/main/docker-compose.yml

# Start ChromaDB
docker compose up -d
```

The auth token is set to `test-token-chroma-local-dev` by default.

You can change this in the `docker-compose.yml` file by changing the `CHROMA_SERVER_AUTH_CREDENTIALS` environment
variable

To stop ChromaDB, run `docker compose down`, to wipe all the data, run `docker compose down -v`.

> **NOTE**
>
> The `docker-compose.yml` file in this repo is provided only as an example and should not be used in
> production.
>
> Go to the ChromaDB [deployment documentation](https://docs.trychroma.com/deployment) for more information on deploying
> Chroma in production.

## Testing

```bash
cp .env.example .env

docker compose up -d
 
composer test
composer analyse src
```

## License

The MIT License (MIT). Please see [License File](LICENSE.md) for more information.
