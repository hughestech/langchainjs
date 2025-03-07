# Agents with Vector Stores

This notebook covers how to combine agents and vector stores. The use case for this is that you’ve ingested your data into a vector store and want to interact with it in an agentic manner.

The recommended method for doing so is to create a VectorDBQAChain and then use that as a tool in the overall agent. Let’s take a look at doing this below. You can do this with multiple different vector databases, and use the agent as a way to choose between them. There are two different ways of doing this - you can either let the agent use the vector stores as normal tools, or you can set `returnDirect: true` to just use the agent as a router.

First, you'll want to import the relevant modules:

```typescript
import { OpenAI } from "langchain/llms/openai";
import { initializeAgentExecutor } from "langchain/agents";
import { SerpAPI, ChainTool } from "langchain/tools";
import { Calculator } from "langchain/tools/calculator";
import { VectorDBQAChain } from "langchain/chains";
import { HNSWLib } from "langchain/vectorstores/hnswlib";
import { OpenAIEmbeddings } from "langchain/embeddings/openai";
import { RecursiveCharacterTextSplitter } from "langchain/text_splitter";
import * as fs from "fs";
```

Next, you'll want to create the vector store with your data, and then the QA chain to interact with that vector store.

```typescript
const model = new OpenAI({ temperature: 0 });
/* Load in the file we want to do question answering over */
const text = fs.readFileSync("state_of_the_union.txt", "utf8");
/* Split the text into chunks */
const textSplitter = new RecursiveCharacterTextSplitter({ chunkSize: 1000 });
const docs = await textSplitter.createDocuments([text]);
/* Create the vectorstore */
const vectorStore = await HNSWLib.fromDocuments(docs, new OpenAIEmbeddings());
/* Create the chain */
const chain = VectorDBQAChain.fromLLM(model, vectorStore);
```

Now that you have that chain, you can create a tool to use that chain. Note that you should update the name and description to be specific to your QA chain.

```typescript
const qaTool = new ChainTool({
  name: "state-of-union-qa",
  description:
    "State of the Union QA - useful for when you need to ask questions about the most recent state of the union address.",
  chain: chain,
});
```

Now you can construct and using the tool just as you would any other!

```typescript
const tools = [new SerpAPI(), new Calculator(), qaTool];

const executor = await initializeAgentExecutor(
  tools,
  model,
  "zero-shot-react-description"
);
console.log("Loaded agent.");

const input = `What did biden say about ketanji brown jackson is the state of the union address?`;

console.log(`Executing with input "${input}"...`);

const result = await executor.call({ input });

console.log(`Got output ${result.output}`);
```

You can also set `returnDirect: true` if you intend to use the agent as a router and just want to directly return the result of the VectorDBQAChain.

```typescript
const qaTool = new ChainTool({
  name: "state-of-union-qa",
  description:
    "State of the Union QA - useful for when you need to ask questions about the most recent state of the union address.",
  chain: chain,
  returnDirect: true,
});
```
