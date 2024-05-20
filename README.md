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

