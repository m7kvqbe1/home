+++
title = 'LangChain, GPT and Messenger'
date = 2024-11-04T10:00:00+00:00
draft = false
+++

I built a conversational AI using LangChain and the OpenAI API that analyzes Facebook Messenger data. This enables you to have meaningful conversations with your historical chat data.

Here’s a quick walkthrough of the process and some insights.

Find the related source code here:

https://github.com/m7kvqbe1/llm-messenger-history

## Obtaining Historical Chat Data

Facebook allows you to download a copy of your data through the "Download Your Information" section in your account settings. After downloading, place your data in a `./data/username/messages/inbox` directory.

## Environment Setup

Before running the application, you'll need to set up a few environment variables:

```bash
OPENAI_API_KEY=your_api_key_here
USERNAME=your_facebook_username
```

## Parsing Messenger Data with LangChain

The application recursively walks through your Messenger directory to load all JSON chat files:

```python
documents = []
for root, dirs, files in os.walk(folder_path):
    for file in files:
        if file.endswith('.json'):
            file_path = os.path.join(root, file)
            loader = FacebookChatLoader(path=file_path)
            documents.extend(loader.load())
```

This loads your Facebook messages as documents ready for processing.

## Chunking Text for Processing

To handle large text blocks efficiently, I split them using LangChain’s `RecursiveCharacterTextSplitter`:

```python
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=200)
docs = text_splitter.split_documents(documents)
print(f"Split into {len(docs)} chunks.")
```

This ensures the data is manageable for embedding and retrieval.

## Embeddings and Vector Store with FAISS

Next, I created embeddings using OpenAI’s API and stored them in a FAISS vector store for efficient similarity search:

```python
embeddings = OpenAIEmbeddings(openai_api_key=OPENAI_API_KEY)
vectorstore = FAISS.from_documents(docs, embeddings)
print("Vector store created.")
```

The vector store acts as the knowledge base for our conversational AI.

## Building the Conversational Chain

With the data ready, I set up a conversational retrieval chain using LangChain:

```python
llm = OpenAI(temperature=0.7, openai_api_key=OPENAI_API_KEY)
qa = ConversationalRetrievalChain.from_llm(llm, vectorstore.as_retriever())
```

This chain handles context-aware responses, making interactions more fluid and natural.

## Smart Batch Processing with Rate Limiting

To handle large datasets efficiently while respecting OpenAI's rate limits, the application implements smart batch processing:

```python
tokens_per_chunk = 250
rate_limit_tpm = 1000000
batch_size = 100

# Calculate optimal wait time between batches
tokens_per_batch = batch_size * tokens_per_chunk
batches_per_minute = rate_limit_tpm / tokens_per_batch
optimal_wait_time = (60 / batches_per_minute) * 1.1
```

The system processes documents in batches with automatic retry logic and dynamic wait times if rate limits are hit.

## Model Selection

The application supports both GPT-3.5 Turbo and GPT-4:

```python
llm = ChatOpenAI(
    temperature=0.7,
    model_name=args.model,  # 'gpt-3.5-turbo' or 'gpt-4'
    openai_api_key=OPENAI_API_KEY
)
```

## Enhanced Chat Interface with Source Citations

The chat responses include source citations to show where the AI's information comes from:

```python
def chat():
    print("Chat with your Facebook Messenger AI (type 'exit' to quit):")
    chat_history = []
    while True:
        query = input("You: ")
        if query.lower() in ('exit', 'quit'):
            break

        result = qa({
            "question": query,
            "chat_history": chat_history
        })

        print(f"\nAI: {result['answer']}\n")

        if "source_documents" in result:
            sources = [doc.page_content[:400]
                      for doc in result["source_documents"][:3]]
            print(f"\nSources: {json.dumps({'sources': sources}, indent=2)}")
```

Example interaction:

```
Chat with your Facebook Messenger AI (type 'exit' to quit):
You: What did I talk about with John last Christmas?
AI: Last Christmas, you and John discussed holiday plans.

Sources: {
  "sources": [
    "December 25, 2023: Discussing holiday dinner plans with John...",
    "December 24, 2023: Coordinating gift exchange timing..."
  ]
}
```
