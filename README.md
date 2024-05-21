# Retrieval Augmented Generation (RAG) Example
<a href="https://app.takeshape.io/add-pattern?repo=https://github.com/takeshape/rag-example/tree/main"><img alt="Deploy To TakeShape" src="https://images.takeshape.io/2cccc825-70be-431c-9ba0-10ab38ecd3a7/dev/8e2f7bda-0e08-4ede-a546-6df59be6a8bb/Deploy%20to%20TakeShape%402x.png?auto=format%2Ccompress" width="205" height="38" style="max-width:100%;"></a>
<img width="1772" alt="Screenshot 2024-05-20 at 1 36 07 PM" src="https://github.com/takeshape/rag-example/assets/428120/45985091-5175-4f6e-bb49-1302672af161">

## Instructions

### Create a new TakeShape Project
1. Click [Deploy to TakeShape](https://app.takeshape.io/add-pattern?repo=https://github.com/takeshape/rag-example/tree/main).
1. Select "Create new project" from the dropdown
1. Enter a name for the new project or leave the default
1. Click "Add to TakeShape"

### Add your API keys
<img width="600" alt="Connect Services" src="https://github.com/takeshape/rag-example/assets/428120/fccdda1f-3466-4dd9-b776-9d16c84f2071">

#### OpenAI
<img width="600" alt="Connect OpenAI" src="https://github.com/takeshape/rag-example/assets/428120/72a15246-1d34-4409-9378-980eba715945">

1. Create an OpenAI API Key https://platform.openai.com/api-keys with at least the "Model capabilities" permissions this example uses `/v1/embeddings` and `/v1/chat/completions`
2. Copy/paste your API key into the service configuration dialog
3. CLick "Save"

#### Shopify Admin
<img width="600" alt="Connect Shopify Admin" src="https://github.com/takeshape/rag-example/assets/428120/66227bda-8bbc-4e53-a9cc-c4a14f5a1649">

Follow the directions in the [TakeShape Shopify documentation](https://app.takeshape.io/docs/services/providers/shopify) to connect Shopify Admin API

### Try it out!
Once your services are connected now try out the API in the API Explorer

```graphql
{
  completion(text: "what shoes should I buy?")
}
```

```graphql
{
  getRelatedProductList(text: "what shoes should I buy?" size: 3) {
    items {
      _id
      title
      descriptionHtml
    }
  }
}
```

## How it works

This example demonstrates how to use TakeShape's vector capabilities combined with API Indexing to enable the RAG use-case.

### Add vector property
The first step is to add a vector property to our `Shopify_Product` shape:
```json
{
  "vector": {
    "type": "array",
    "items": {
      "type": "number"
    },
    "title": "Vector",
    "@tag": "vector"
  }
}
```
Vectors are arrays of floating point numbers which is easy to express using the JSON Schema syntax. Using the `@tag` annotation with the value of `vector` tells TakeShape to treat this array property as a vector and to enable vector (knn) search on it. In this property we want to store the OpenAI calculated embedding of the text content of the `Shopify_Product`. To specify this we use an `@resolver` annotation to tell TakeShape how to get the embedding from OpenAI:
```json
{
  "vector": {
    ...
    "@resolver": {
      "name": "delegate",
      "to": "openai:Mutation.createEmbedding",
      "options": {
        "selectionSet": "{ data { embedding } }"
      },
      ...
    }
  }
}
```
We use the `delegate` resolver to delegate to the `createEmbedding` mutation provided by our OpenAI service. To send the text content of `Shopify_Product` to `createEmbedding` we use the `args` mapping and `@dependencies` annotation:
```json
{
  "vector": {
    ...
    "@resolver": {
      ...
      "args": {
        "ops": [
          {
            "path": "input",
            "value": {
              "model": "text-embedding-3-small",
              "encoding_format": "float"
            }
          },
          {
            "path": "input.input",
            "mapping": [
              [
                "get",
                {
                  "path": [
                    "$source.title",
                    "$source.description"
                  ]
                }
              ],
              [
                "format",
                {
                  "template": "%s %s"
                }
              ]
            ]
          }
        ]
      }
    }
  }
}
```
We use `args.ops` to set the `input` arg with an object that looks like:
```json
{
  "model": "text-embedding-3-small",
  "encoding_format": "float",
  "input": "{title} {description}"
}

```
Using `get` and `format` directives we concatenate the `title` and `description` properties and and assign the result to `input.input`. To ensure that we always always load the `title` and `description` when reading the `vector` field we use `@depdendencies`:
```json
{
  "vector": {
    ...
    "@dependencies": "{title description}",
    "@resolver": {
      ...
    }
  }
}
```
To grab the vector value from the response of `createEmbedding` we use `results` mapping:
```json
{
  "vector": {
    ...
    "@resolver": {
      "name": "delegate",
      "to": "openai:Mutation.createEmbedding",
      "options": {
        "selectionSet": "{ data { embedding } }"
      },
      ...
      "results": {
        "ops": [
          {
            "path": "$",
            "mapping": "$finalResolver.data[0].embedding"
          }
        ]
      }
    }
  }
}
```
`options.selectionSet` defines the fragment that `createEmbedding` returns and `results.ops[0]` defines how to map that response to our vector property `$`.

### Enable API Indexing
After adding the vector property to `Shopify_Product` the next step is to enable indexing:
```json
{
  "Shopify_Product": {
    ...
    "cache": {
      "enabled": true,
      "triggers": [
        {
          "type": "schedule",
          "loader": "list",
          "interval": 1440
        },
        {
          "type": "webhook",
          "loader": "get",
          "service": "shopify",
          "events": [
            "products/create",
            "products/update",
            "products/delete"
          ]
        }
      ],
      "fragment": {
        "maxDepth": 2
      }
    },
    "loaders": {
      "list": {
        "query": "shopify:Query.products"
      },
      "get": {
        "query": "shopify:Query.product"
      }
    },
    "schema": {...}
  }
}
```
Above is the default indexing configuration for `Shopify_Product` generated by checking the "Indexing" checkbox in the Shape Editor UI. This configuration will index all products every 24hrs using the the list loader and will index single products in response to a webhook.

### Create `getRelatedProductList` query
Once indexing is enabled the next step is to create the `getRelatedProductList` query. This query composes OpenAI's `createEmbedding` mutation and TakeShape's `vectorSearch` resolver:
```json
{
  "queries": {
    "getRelatedProductList": {
      "shape": "PaginatedList<shopify:Product>",
      "resolver": {
        "compose": [
          {
            "name": "delegate",
            "id": "createEmbedding",
            "to": "openai:Mutation.createEmbedding",
            ...
          },
          {
            "shapeName": "Shopify_Product",
            "name": "takeshape:vectorSearch",
            "service": "takeshape",
            ...
          }
        ]
      },
      "args": {
        "type": "object",
        "properties": {
          "text": {
            "type": "string"
          },
          "size": {
            "type": "integer"
          }
        },
        "required": [
          "text"
        ]
      }
    }
  }
}
```
This query uses a `compose` resolver to delegate to `createEmbedding` and then pass the resulting embedding vector to `takeshape:vectorSearch` which searches our indexed `Shopify_Product`. First let's look at the resolver to fetch the embedding:
```json
{
  "name": "delegate",
  "id": "createEmbedding",
  "to": "openai:Mutation.createEmbedding",
  "args": {
    "ops": [
      {
        "path": "input",
        "value": {
          "model": "text-embedding-3-small",
          "encoding_format": "float"
        }
      },
      {
        "path": "input.input",
        "mapping": "$args.text"
      }
    ]
  },
  "results": {
    "ops": [
      {
        "path": "vector",
        "mapping": "$currentResolver.data[0].embedding"
      }
    ]
  }
}
```
This resolver looks very similar to the resolver in the previous section, but it is creating an embedding of th query you pass in the `text` arg. Then we take the resulting vector and pass it into the `vector.value` arg of the `takeshape:vectorSearch` resolver:
```json
{
  "shapeName": "Shopify_Product",
  "name": "takeshape:vectorSearch",
  "service": "takeshape",
  "args": {
    "ops": [
      {
        "path": "vector.name",
        "value": "vector"
      },
      {
        "path": "vector.value",
        "mapping": "$resolvers.createEmbedding.vector"
      },
      {
        "path": "size",
        "mapping": "$args.size"
      }
    ]
  }
}
```
Note that the `vector.name` is set to `"vector"` to correspond with the property name on `Shopify_Product`. The `size` argument determines the number of related (k in k-nearest neighbors).

### Create `completion` query
The last step is to put it all together with the `completion` query. To generate text in response to a users prompt we are composing `getRelatedProductList` with OpenAI's `createChatCompletion`:

```json
{
  "queries": {
    "completion": {
      "shape": "string",
      "resolver": {
        "compose": [
          {
            "name": "delegate",
            "id": "getRelatedProductList",
            "to": "Query.getRelatedProductList",
            ...
          },
          {
            "name": "delegate",
            "to": "openai:Mutation.createChatCompletion",
            ...
          }
        ]
      },
      "args": {
        "type": "object",
        "properties": {
          "text": {
            "type": "string"
          }
        },
        "required": [
          "text"
        ]
      }
    }
  }
}
```
First we take the `text` arg and fetch top related product using `getRelatedProductList`:
```json
{
  "name": "delegate",
  "id": "getRelatedProductList",
  "to": "Query.getRelatedProductList",
  "options": {
    "selectionSet": "{ items {title description} }"
  },
  "args": {
    "ops": [
      {
        "path": "text",
        "mapping": "$args.text"
      },
      {
        "path": "size",
        "value": 1
      }
    ]
  },
  "results": {
    "ops": [
      {
        "path": "$",
        "mapping": "$currentResolver.items[0]"
      }
    ]
  }
}
```
Then we combine the text from the top related product and the user's original prompt and send it to GPT 3.5 via OpenAI's `createChatCompletion` mutation:
```json
{
  "name": "delegate",
  "to": "openai:Mutation.createChatCompletion",
  "options": {
    "selectionSet": "{ choices { message { role content } } }"
  },
  "args": {
    "ops": [
      {
        "path": "input",
        "value": {
          "model": "gpt-3.5-turbo",
          "messages": [
            {
              "role": "system",
              "content": "You are a helpful assistant."
            },
            {
              "role": "user",
              "content": ""
            }
          ]
        }
      },
      {
        "path": "input.messages[1].content",
        "mapping": [
          [
            "get",
            {
              "path": [
                "$args.text",
                "$resolvers.getRelatedProductList.title",
                "$resolvers.getRelatedProductList.description"
              ]
            }
          ],
          [
            "format",
            {
              "template": "Answer the following question:\n %s\n by using the following text:\n %s %s"
            }
          ]
        ]
      }
    ]
  },
  "results": {
    "ops": [
      {
        "path": "$",
        "mapping": "$currentResolver.choices[0].message.content"
      }
    ]
  }
}
```
The key to RAG is using the user's prompt and relevant data from a vector search and combining it into a new prompt for the LLM:
```json
{
  "path": "input.messages[1].content",
  "mapping": [
    [
      "get",
      {
        "path": [
          "$args.text",
          "$resolvers.getRelatedProductList.title",
          "$resolvers.getRelatedProductList.description"
        ]
      }
    ],
    [
      "format",
      {
        "template": "Answer the following question:\n %s\n by using the following text:\n %s %s"
      }
    ]
  ]
}
```