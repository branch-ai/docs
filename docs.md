# Branch AI API Documentation

## Table of Contents
- [Tutorials](#tutorials)
  - [Installation](#installation)
  - [Hello World w/Search (Simple)](#hello-world-wsearch-simple)
  - [Hello World w/Search (Advanced)](#hello-world-wsearch-advanced)
  - [Hello World w/Search (Multiple Sources)](#hello-world-wsearch-multiple-sources)
  - [Chatbot (for Website Visitors)](#chatbot-for-website-visitors)
  - [Recommendations - Content (for People Matching)](#recommendations---content-for-people-matching)
  - [Recommendations - Collaborative (for Ecommerce)](#recommendations---collaborative-for-ecommerce)
- [Core Concepts](#core-concepts)
  - [Source](#source)
  - [Document](#document)
  - [Transform](#transform)
  - [Pipeline](#pipeline)
  - [Destination](#destination)
  - [Queries](#queries)
  - [Model](#model)
- [Authentication](#authentication)
  - [POST /auth/login](#post-auth-login)
- [Source](#source-1)
  - [GET /source/{source_id}](#get-sourcesource_id)
  - [POST /source](#post-source)
  - [PUT /source/{source_id}](#put-sourcesource_id)
  - [DELETE /source/{source_id}](#delete-sourcesource_id)
- [Transform](#transforms-1)
  - [GET /transform/{transform_id}](#get-transformtransform_id)
  - [POST /transform](#post-transform)
  - [PUT /transform/{transform_id}](#put-transformtransformid)
  - [DELETE /transform/{transform_id}](#delete-transformtransformid)
- [Pipeline](#pipeline-1)
  - [GET /pipelines](#get-pipelines)
  - [GET /pipeline/{pipeline_id}](#get-pipelinepipeline_id)
  - [POST /pipeline](#post-pipeline)
  - [PUT /pipeline/{pipeline_id}](#put-pipelinepipeline_id)
  - [DELETE /pipeline/{pipeline_id}](#delete-pipelinepipeline_id)
  - [POST /pipeline/start/{pipeline_id}](#post-pipelinestartpipeline_id)
  - [POST /pipeline/stop/{pipeline_id}](#post-pipelinestoppipeline_id)
- [Destination](#destination-1)
  - [GET /destination/{destination_id}](#get-destinationdestination_id)
  - [POST /destination](#post-destination)
  - [PUT /destination/{destination_id}](#put-destinationdestination_id)
  - [DELETE /destination/{destination_id}](#delete-destinationdestination_id)
- [Queries](#queries-1)
  - [POST /search](#post-search)
  - [POST /generate](#post-generate)
- [Recipes](#recipes)
  - [Web Page to Vector DB](#web-page-to-vector-DB) 

## Tutorials

### Installation
```
git clone https://github.com/branch-ai/branch-ai.git
cd branch-ai
./run-branch-ai.sh
```


### Hello World w/Search (Simple)

In this example we:
- Step 1: Connect to a Google Drive source, by supplying credentials (to be obtained from Google)
- Step 2: Pull documents every 5 minutes
- Step 3: Uses a transform template DAG known as "split_embed_index" which
    - Splits ingested text
    - Embeds it using default models
    - Indexes it into the default local vector DB
- Step 4: Starts the pipeline
- Step 5: Sets up a search endpoint
- Step 6: Sends a query to obtain an answer


```
import branchai

client = branchai.client('localhost:8000')
source = client.source(
        type="gdrive",
        credentials=credentials
    )
transform = branchai.transform(type="split_embed_index")
pipeline = client.pipeline(
        sources=[source],
        transform=transform,
        schedule="5 * * * *"
    )
status = pipeline.start()
adapter = client.adapter.search(pipeline)
answer = adapter.query("Hello world, how are you?")
```

### Hello World w/Search (Advanced)

In the previous example, we used template recipe magic for `transform` and `destination`. In this example, we take full control over pipeline definition to achieve the same outcome.

```
import branchai

client = branchai.client('localhost:8000')
source = client.source(
        type="gdrive", 
        credentials=credentials
    )
destination = client.destination(
        type="weaviate",
        connection_details={"url": "http://localhost:8080"}
    )
transform = branchai.transform(
    yaml=f"""
        info:
          version: 0.0.1
        sources:
          - {source.id}
        destinations:
          - {destination.id}
        pipeline:
          stages:
            text_split:
              stage: text_split
              params:
                method: paragraph
            generate_embedding:
              stage: langchain_generate_embedding
              params:
                type: huggingface
            index_document:
              stage: weaviate_index_document
              params:
                class_name: demo
              connections:
                - {destination.id}
          dag:
            origin:
              - {source.id}
            {source.id}:
              - text_split
            text_split:
              - generate_embedding
            generate_embedding:
              - index_document
        """
)
pipeline = client.pipeline(
        sources=[source],
        transform=transform,
        schedule="5 * * * *"
    )
status = pipeline.start()
adapter = client.adapter.search(pipeline)
documents = adapter.query("Hello world, how are you?", top_k=5)
```

### Hello World w/Search (Multiple Sources)

In this example, we revert to recipe magic, and evolve the code to support multiple sources. This can be used to build internal knowledge search, for example.

```
import branchai

client = branchai.client('localhost:8000')
gdrive_source = client.source(
        type="gdrive",
        params={"folder": "1Sxd3AdCRfMAehyo4QNQPs23fst68xhBF7VXfnaq04FA"}
        credentials=gdrive_credentials
    )
notion_source = client.source(
        type="notion", 
        credentials=notion_credentials
    )
transform = branchai.transform(type="split_embed_index")
pipeline = client.pipeline(
        sources=[gdrive_source, notion_source],
        transform=transform,
        schedule="5 * * * *"
    )
status = pipeline.start()
adapter = client.adapter.search(pipeline)
documents = adapter.query("Projects for ACME Corp", top_k=10)
```



### Chatbot (for Website Visitors)

Goal: Build a chatbot indexed on your company's website using GPT-4. Branch AI pipelines will adapt to changes made to your website to ensure that you chatbot is never out of sync with your content.

```
import branchai

client = branchai.client('localhost:8000')
source = client.source(
        type="crawler",
        params={"seed_url" : "www.example.com", "max_pages" : 100}
    )
destination = client.destination(
        type="weaviate",
        connection_details={"url": "http://localhost:8080"}
    )
transform = branchai.transform(
    yaml=f"""
        info:
          version: 0.0.1
        sources:
          - {source.id}
        destinations:
          - {destination.id}
        pipeline:
          stages:
            html_parser:
              stage: html_parser
            text_split:
              stage: text_split
              params:
                method: section
            generate_embedding:
              stage: langchain_generate_embedding
              params:
                type: text-embedding-ada-002
            index_document:
              stage: weaviate_index_document
              params:
                class_name: demo
              connections:
                - {destination.id}
          dag:
            origin:
              - {source.id}
            {source.id}:
              - html_parser
            html_parser:
              - text_split
            text_split:
                generate_embedding
            generate_embedding:
              - index_document
        """
)
pipeline = client.pipeline(
        sources=[source],
        transform=transform,
        schedule="5 * * * *"
    )
status = pipeline.start()
adapter = client.adapter.generate(pipeline, model="gpt-4")
answer = adapter.query("What are the opening hours for the store?")
```

### Recommendations - Content (for People Matching)

Goal: Match people with similar interests based on their profiles

```
import branchai
import json

client = branchai.client('localhost:8000')
source = client.source(
        type="postgresql",
        params={
            "schema": "private",
            "table": "users",
            "time_column": "updated_at"
            "id_column": "user_id",
            "data_columns": ["preference", "description", "personal_details"]
        },
        credentials=credentials
    )
destination = client.destination(
        type="weaviate",
        connection_details={"url": "http://localhost:8080"}
    )
transform = branchai.transform(
    yaml=f"""
        info:
          version: 0.0.1
        sources:
          - {source.id}
        destinations:
          - {destination.id}
        pipeline:
          stages:
            run_text_prepare:
              stage: custom_python
              main: prepare_text
              code: |
                def prepare_text(data: Document) -> str:
                    return json.dumps(data.content)
            truncate_context_window:
              stage: truncate_context_window
              params:
                model: text-embedding-ada-002
            generate_embedding:
              stage: langchain_generate_embedding
              params:
                type: text-embedding-ada-002
            index_document:
              stage: weaviate_index_document
              params:
                class_name: demo
              connections:
                - {destination.id}
          dag:
            origin:
              - {source.id}
            {source.id}:
              - run_text_prepare
            run_text_prepare:
              - truncate_context_window
            truncate_context_window:
              - generate_embedding
            generate_embedding:
              - index_document
        """
)
pipeline = client.pipeline(
        sources=[source],
        transform=transform,
        schedule="* 1 * * *"
    )
status = pipeline.start()
adapter = client.adapter.recommend(pipeline)
similar_matches = adapter.query(json.dumps({'user_id': 1001, "description": "...", "personal_details": {}, "preferences": {}}))
```


### Recommendations - Collaborative (for Ecommerce)

Goal: Recommend e-commerce products to customers using their past purchase transactions

```
import branchai

client = branchai.client('localhost:8000')

# Pipeline for generating product embeddings
source = client.source(
        type="postgresql",
        params={
            "schema": "private",
            "table": "products",
            "time_column": "updated_at"
            "id_column": "id",
            "data_columns": ["name", "description", "category", "details"]
        },
        credentials=credentials
    )
destination = client.destination(
        type="weaviate",
        connection_details={"url": "http://localhost:8080"}
    )
redis_conn = client.destination(
        type="redis",
        connection_details={"host": "localhost", "port": 6379}
    )
transform = branchai.transform(
    yaml=f"""
        info:
          version: 0.0.1
        sources:
          - {source.id}
        destinations:
          - {destination.id}
          - {redis_conn.id}
        pipeline:
          stages:
            run_text_prepare_1:
              stage: custom_python
              main: prepare_text
              code: |
                def prepare_text(data: dict) -> str:
                    return str(data)
            truncate_context_window:
              stage: truncate_context_window
              params:
                model: text-embedding-ada-002
            generate_embedding:
              stage: langchain_generate_embedding
              params:
                type: text-embedding-ada-002
            index_document:
              stage: weaviate_index_document
              params:
                class_name: products
              connections:
                - {destination.id}
            cache_redis:
              stage: redis_insert
              connections:
                - {redis_conn.id}
          dag:
            origin:
              - {source.id}
            {source.id}:
              - run_text_prepare_1
            run_text_prepare_1:
              - truncate_context_window
            truncate_context_window:
                generate_embedding
            generate_embedding:
              - index_document
              - cache_redis
        """
)
product_pipeline = client.pipeline(
        sources=[source],
        transform=transform,
        schedule="* 1 * * *"
    )

# Pipeline for generating user embeddings based on past purchases
source = client.source(
        type="postgresql",
        params={
            "schema": "private",
            "table": "purchases",
            "time_column": "updated_at"
            "id_column": "user_id",
            "sql": "select user_id, array_agg(product_id order by updated_at desc) as products from purchases group by user_id"
        },
        credentials=credentials
    )
transform = branchai.transform(
    yaml=f"""
        info:
          version: 0.0.1
        sources:
          - {source.id}
        destinations:
          - {destination.id}
          - {redis_conn.id}
        pipeline:
          stages:
            run_prepare:
              stage: custom_python
              file: generate_user_embedding.py
              main: generate_user_embedding
              connections:
                - {redis_conn.id}
            index_document:
              stage: weaviate_index_document
              params:
                class_name: users
              connections:
                - {destination.id}
          dag:
            origin:
              - {source.id}
            {source.id}:
              - run_prepare
            run_prepare:
              - index_document
        """
user_pipeline = client.pipeline(
        sources=[source],
        transform=transform,
        schedule="5 * * * *"
    )
status = pipeline.start()

# Generate recommendations
adapter = client.adapter.recommend(product_pipeline)
user_embedding = client.adapter.fetch(user_pipeline).fetch(id="<user_id>")
recommendations = adapter.query(embedding=user_embedding)
```


## Core Concepts

These are the key objects (serialized into JSON) that our application works with:

### Source

A source is a thin wrapper over a data source, be it an external SaaS tool (e.g. Notion, Confluence, Gmail), an internal database (e.g. PostgreSQL, Redshift) or website (e.g. www.example.com). Please use our UI or contact us for a list of supported sources, details on how to configure each of them and requests for additional support.
```
class Source:
    id: Optional[int] = None
    name: Optional[str] = ""
    user_id: str
    type: str
    typeid: Optional[str] = ""
    credentials: Optional[dict] = {}
    description: Optional[str] = ""
    params: Optional[dict] = {}
```

### Document

The data to process is represented by a Document object. The [transforms](#transform) are run on Document object. 
```
class Document:
    stage: str
    id: Optional[str] = None
    parent_ids: Optional[str] = None
    root_id: Optional[str] = None
    embedding: Optional[numpy.ndarray] = None
    content: List[Element] = field(default_factory=list)
    attributes: defaultdict(dict) = field(default_factory=dict)
```

### Transform

Each stage in the processing pipeline is represented as a transform. A transform takes in [Document(s)](#document) and applies the pre-defined logic to output [Document(s)](#document). We offer out-of-the-box recipes (e.g. `split_embed_index` for split -> embed -> index) as well as fundamental operators like text splitter, embedding generator, etc for user to build their own processing logic. You can also define your own transform logic as custom Python code (idempotent functions).
```
class Transform:
    id: int
    user_id: str
    name: str
    description: str
    params: dict
```

### Pipeline

A pipeline comprises a list of sources, corresponding transforms and destination, and is associated with a given user. All configurations for the pipeline, with the exception of the schedule, can be found in the YAML. The YAML contains the DAG representing the processing logic for the pipeline. 

```
class Pipeline:
    id: Optional[int] = None
    name: Optional[str] = ""
    user_id: str
    source_ids: List[int]
    destination_ids: List[int]
    yaml: str
    schedule: str
    status: Optional[str] = None

```

### Destination

A destination is a thin wrapper over a data store that is able to store the outputs of a pipeline. This could be a vector DB, a relational database, or blob storage.

```
class Destination:
    id: Optional[int] = None
    name: Optional[str] = ""
    user_id: str
    type: str
    typeid: Optional[str] = ""
    connection_details: dict
    description: Optional[str] = ""
    params: Optional[dict] = {}
```

### Adapter

An adapter is a thin wrapper around a `destination` that applies a pre-configured transformation on the input query (often bootstrapping onto the pipelines that were used to send data to vector indexes in the first place)

```
class Adapter:
    id: Optional[int] = None
    name: Optional[str] = ""
    type: str
    destination_id: int
    pipeline_id: int
```

### SearchQuery

Search query to be sent as parameter for [/search](#)

```
class SearchQuery:
    user_id: Optional[str]
    query: str
    pipeline_id: Optional[int] = None
    destination_id: Optional[int] = None
    top_k: Optional[int] = 1
    certainty: Optional[float] = 0.5
```

### SearchResult

Matching record representing the search result of a query. List[SearchResult] is returned in response for a `search` query. 

```
class SearchResult:
    text: str
    score: Optional[float] = None
    url: Optional[str] = None
    section_url: Optional[str] = None
    document_id: Optional[str] = None
    document_title: Optional[str] = None
    root_id: Optional[str] = None
```

### GenerateQuery

Generate query to be sent as parameter for [/generate](#)

```
class GenerateQuery:
    user_id: Optional[str]
    query: str
    system_prompt: Optional[str]
    pipeline_id: Optional[int] = None
    destination_id: Optional[int] = None
    top_k: Optional[int] = 1
    certainty: Optional[float] = 0.5
    model: str
    temperature: Optional[float] = 0
    model: Optional[str] = "gpt-3.5-turbo"
```

### Answer

Record containing the answer generated by ChatGPT for a `generate` query. `documents` contain the list of matching documents for the query. 

```
class Answer:
    text: str
    query: str
    score: Optional[float] = None
    documents: Optional[List[SearchResult]] = None
```

### Model

A model is a special type of resource that can be registered with us, in order to run model inference operations in transform DAGs, and eventually to support out of the box fine-tuning and pre-training of models using the datasets that we generate.




## Authentication

### POST /auth/login
Authenticate the user and return a JWT.

#### Input
- Username and password base64 encoded in the HTTP headers, as per [fastapi](https://fastapi.tiangolo.com/advanced/security/http-basic-auth/) docs.

#### Output
- `{ "jwt": <generated_jwt> }`
- This JWT needs to be authenticated upon every subsequent request for access to the application.


## Source

### GET /source/{source_id}
Fetch an existing source.

#### Input
- `source_id`: Source ID provided in the URL path.

#### Output
- A source's details in JSON. The details of the class can be found [here](#source).

### POST /source
Create a new source.

#### Input
- Source object in the JSON body.

#### Output
- A source's details in JSON. The details of the class can be found [here](#source).
- Any errors should reflect as `404` status code with `detail` field.

### PUT /source/{source_id}
Update an existing source.

#### Input
- `source_id` in the URL path.
- The new source object in the JSON body.

#### Output
- `{ "id": <source_id>, "comment": <outcome> }`
- Any errors should be reflected in the comment field.

### DELETE /source/{source_id}
Delete an existing source.

#### Input
- `source_id` in the URL path.

#### Output
- `{ "id": <source_id>, "comment": <outcome> }`
- Any errors should be reflected in the comment field.


## Transform

### GET /transform/{transform_id}
Fetch an existing transform

#### Input
- `transform_id` in the url path. The details of the class can be found [here](#transform).

#### Output
- A transform’s details in JSON

### POST /transform
Create a new transform.

#### Input
- Transform object in the JSON body.

#### Output
- `{ "id": <transform_id>, "comment": <outcome> }`
- Any errors should be reflected in the `comment` field.

### PUT /transform/{transform_id}
Modify an existing transform.

#### Input
- `id`: transform_id provided in the URL path.
- Transform object in the JSON body.

#### Output
- `{ "id": <transform_id>, "comment": <outcome> }`
- Any errors should be reflected in the `comment` field.

### DELETE /transform/{transform_id}
Delete a transform.

#### Input
- `id`: transform_id provided in the URL path.

#### Output
- `{ "id": <transform_id>, "comment": <outcome> }`
- Any errors should be reflected in the `comment` field.


## Pipeline

### GET /pipelines
Returns the pipelines that a user has created. Each pipeline is identified by a unique ID, and is configured by a YAML string. This can be used to generate a preview of the pipelines that a user has.

#### Input
- None.

#### Output
- A list of pipeline configurations as a single JSON:
```[{ "id": <pipeline_id>, "yaml": <pipeline_yaml> }, ...]```


### GET /pipeline/{pipeline_id}
Return details for a given pipeline.

#### Input
- `id`: pipeline_id provided in the URL path.

#### Output
- A pipeline's details in JSON. The details of the class can be found [here](#pipeline).

### POST /pipeline
Create a new pipeline.

#### Input
- Pipeline object in the JSON body.

#### Output
- `{ "id": <pipeline_id>, "comment": <outcome> }`
- Any errors should be reflected in the `comment` field.

### PUT /pipeline/{pipeline_id}
Modify an existing pipeline.

#### Input
- `id`: pipeline_id provided in the URL path.
- Pipeline object in the JSON body.

#### Output
- `{ "id": <pipeline_id>, "comment": <outcome> }`
- Any errors should be reflected in the `comment` field.

### DELETE /pipeline/{pipeline_id}
Remove a pipeline.

#### Input
- `id`: pipeline_id provided in the URL path.

#### Output
- `{ "id": <pipeline_id>, "comment": <outcome> }`
- Any errors should be reflected in the `comment` field.


### POST /pipeline/start/{pipeline_id}
Sets the pipeline to `active` state.

#### Input
- `id`: pipeline_id provided in the URL path.

#### Output
- `{ "id": <pipeline_id>, "comment": <outcome> }`
- Any errors should be reflected in the `comment` field.


### POST /pipeline/stop/{pipeline_id}
Sets the pipeline to `inactive` state.

#### Input
- `id`: pipeline_id provided in the URL path.

#### Output
- `{ "id": <pipeline_id>, "comment": <outcome> }`
- Any errors should be reflected in the `comment` field.



## Destination

### GET /destination/{destination_id}
Fetch an existing destination.

#### Input
- `destination_id` in the URL path.

#### Output
- A destination's details in JSON.
- The details of the class can be found [here](#destination).

### POST /destination
Create a new destination.

#### Input
- Destination object in the JSON body.

#### Output
- `{"id": <destination_id>, "comment": <outcome>}`
- Any errors should be reflected in the comment field.

### PUT /destination/{destination_id}
Update an existing destination

#### Input
- `destination_id` in the url path and the new destination object in the JSON body.

#### Output
- `{"id": <destination_id>, "comment": <outcome>}`
- Any errors should be reflected in the comment field.

### DELETE /destination/{destination_id}
Delete an existing destination

#### Input
- `destination_id` in the url path

#### Output
- `{"id": <destination_id>, "comment": <outcome>}`
- Any errors should be reflected in the comment field.


## Queries

### POST /search

Search for matching documents that have been indexed

#### Input
- query to search for relevant documents. The details of the class can be found [here](#searchquery)

#### Output
- List of records containing matching documents. Each record will be of type [SearchResult](#searchresult)

### POST /generate

Generate an answer from chatgpt using matching documents for a query

#### Input
- query to generate answer using chatgpt based on matching documents. The details of the class can be found [here](#generatequery)

#### Output
- Record with answer generated by chatgpt and matching documents sent to chatgpt. Record will be of type [Answer](#answer)

### POST /query

Endpoint for chatGPT Retrieval Plugin integration.

#### Input
- query sent by chatgpt

#### Output
- List of records containing matching documents. Each record will be of type [SearchResult](#searchresult)


## Recipes
### Web page to Vector DB
Use this recipe to crawl pages and index the webpage content into a vector DB. The following steps are executed:
- Spider using given domain name and seed urls to download the HTML of each page
- Split the HTML into meaningful sections using heading tags (h1 .. h6)
- Embed each section of the page using specified model
- Write the embedding along with meta data to a given vector db


#### Using API
```
payload = {
    "user_id": user_id, 
    "source": {"domain": "example.com", "seed_urls": "seed_urls": ["https://example.com/"], "patterns": ["/page"], "enable_spider": True},
    "destination": {"type": "weaviate", "params": {"index_name": "Pages"}, "connection_details": {"url": "http://172.17.0.1:8080"} },
    "transform": {"embedding": {"model": "openai/text-embedding-ada-002", "api_key": "<openai api key>"} },
    "schedule": "12 hours"
}
response = requests.post("{}/recipes/webpage".format(server), json=payload)
response.json()

# Response
{
  'sources': [{'id': 1}],
  'destinations': [{'id': 1}],
  'pipeline': {'id': 1},
  'comment': 'Pipeline has been setup'
}
```


#### Using Library
```
import branchai
client = branchai.Client(f"{server_url}")
payload = {
    "user_id": user_id, 
    "source": {"domain": "example.com", "seed_urls": ["https://www.example.com"], "patterns": ["page"], "enable_spider": False, "css_selectors": ["body"]},
    "destination": {"type": "weaviate", "params": {"index_name": "Page"}, "connection_details": {"url": "<weaviate server url>" },
    "transform": {"embedding": {"model": "openai/text-embedding-ada-002", "api_key": "<openai api key>"}},
    "schedule": "12 hours"
}
pipeline = client.with_recipe("webpage").create_pipeline(payload)

```

#### Parameters

Parent | Attribute | Type | Description | Required (Y/N)
--- | --- | --- | --- | --- 
source | domain | string | Domain to be crawled | Y
source | seed_urls | List[string] | List of seed urls to crawl | Y
source | patterns | List[string] | List of url patterns to match for spidering | N
source | enable_spider | Boolean | False - only seed_urls will be crawled. True - enable spidering starting from seed urls | Y
source | css_selectors | List[string] | List of css selectors to select text from html page. If empty then entire text from `<body>` tag will be extracted | N
source | params | dict | Source specific parameters. <br /> 1. `max_url_count` - Once the count has been reached, spider will send a signal to stop crawling further. We may end up with slight more urls than `max_url_count`. Defaults to 100.  | N 
destination | type | string | Type of vector db, only weaviate is supported now | Y
destination | params | dict | Parameters for destination. `index_name` - Index name for vector db | Y
destination | connection_details | dict | Connection parameters for destination | Y
transform | embedding | dict | Parameters for embedding model - <br />1. `model` - Model to be used for embedding generation. <br />a. `huggingface/sentence-transformers/all-mpnet-base-v2` [default]  <br />b. `openai/text-embedding-ada-002` <br />2. `api_key` - API Key for model. Eg. OpenAI API Key  | N
schedule | | string | Frequency for the pipeline. Eg. "X minutes", "Y hours", "Z days". Default is adhoc | Y

#### Generates source connector
```
payload = {
    "user_id": user_id, 
    "type": "webcrawl", 
    "description": "crawl source connector",
    "params": {"domain": "example.com", "seed_urls": ["https://example.com/"], "patterns": ["/page"], "max_url_count": 100, "enable_spider": true} },
    "credentials": {}
}
response = requests.post("{}/source".format(server), json=payload)
crawl = response.json()
crawl
```

#### Generate destination connector
```
payload = {
  "user_id": user_id,
  "type": "weaviate",
  "connection_details": {"url": "http://localhost:8080"}
}
response = requests.post("{}/destination".format(server), json=payload)
weaviate = response.json()
weaviate
```

#### Sample YAML for pipeline setup
Running the recipe will setup the source and destination connectors. `recipes.webpage_embed_vectordb` will generate the following YAML for pipeline - 
```
info:
  version: 0.0.1
sources:
  - webcrawl_15
destinations:
  - weaviate_15
pipeline:
  stages:
    html_parse:
      stage: parse_html
      params:
        enable_chunking: False
        css_selectors: ['main']
    generate_embed:
      stage: langchain_generate_embedding
      params:
        type: openai/text-embedding-ada-002
        api_key: encrypted_openai_api_key
    index_document:
      stage: weaviate_index_document
      params:
        index_name: Page
      connections:
        - weaviate_15
  dag:
    origin:
      - webcrawl_15
    webcrawl_15:
      - html_parse
    html_parse:
      - generate_embed
    generate_embed:
      - index_document
```


