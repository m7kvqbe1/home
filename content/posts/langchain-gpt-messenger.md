+++
title = 'LangChain, GPT and Messenger'
date = 2024-11-04T10:00:00+00:00
draft = false
+++

> LangChain is a software framework that helps facilitate the integration of large language models (LLMs) into applications.

I built a conversational AI using LangChain and the OpenAI API that parses Facebook Messenger data. This enables you to have meaningful conversations with your historical chat data.

Here’s a quick walkthrough of the process and some insights.

Find the source code here:

https://github.com/your-repo/facebook-chat-ai

## Obtaining Historical Chat Data

Facebook allows you to download a copy of your data through the "Download Your Information" section in your account settings. Ensure that you select the option to include your Messenger data.

## Parsing Messenger Data with LangChain

LangChain's `FacebookChatLoader` made it straightforward to import and handle chat data. Start by pointing it to your Messenger directory:

```python
loader = FacebookChatLoader(path="./data/username/messages/inbox")
documents = loader.load()
print(f"Loaded {len(documents)} documents.")
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

## Chatting with Your AI

Finally, I created an interactive loop to chat with the AI:

```python
def chat():
    print("Chat with your Facebook Messenger AI (type 'exit' to quit):")
    chat_history = []
    while True:
        query = input("You: ")
        if query.lower() in ('exit', 'quit'):
            print("Exiting chat.")
            break
        if not query.strip():
            continue
        result = qa({"question": query, "chat_history": chat_history})
        answer = result["answer"]
        print(f"AI: {answer}")
        chat_history.append((query, answer))

chat()
```

```
Chat with your Facebook Messenger AI (type 'exit' to quit):
You: What did I talk about with John last Christmas?
AI: Last Christmas, you and John discussed holiday plans.
You: Who else was involved in the conversation?
AI: You also mentioned inviting Sarah and Mike to join the holiday gathering.
You: exit
Exiting chat.
```
