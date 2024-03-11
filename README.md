# Quantitative Data Analysis Using LLMs - Text To SQL Guide

LLMs are great at processing semantic information and can even complete logical tasks in many cases. Unfortunately, LLMs struggle to do math and accurately process large amounts of data. One solution to this challenge is a pipeline that makes use of SQL and several prompts to provide a final result.

This guide will show how to implement a very basic Text to SQL pipeline that makes use of PGVector for vector searches and GPT as the LLM.

As an example we will have a list of purchases made on a credit card

## Process

### The Initial Text to SQL Call

```typescript
async function getQuery(businessId: string) {
  const openai = new OpenAI({
    apiKey: process.env.OPENAI_API_KEY,
  });

  const prompt = `
  You are a Postgres SQL generator. You are provided with a question about business finances. Your job is to generate an SQL query that will pull all information relating to this question.

  For queries that search for a category or vendor, do not directly search. Instead insert EMBEDDINGSEARCHWHERE(type, query). Pick vendor, category or item for search type. This will later be parsed and repalced by the backend.

  - Filter queries to businessid - ${businessId}
  - The current date and time is ${new Date().toLocaleString()}

  transactions table

  Column | Type 
  businessid         | text
  vendor             | text
  vendorcategory     | text
  transactionid      | text
  date               | date
  time               | int
  total              | numeric
  categoryEmbedding  | vector(1536)
  vendorEmbedding    | vector(1536)
  `;

  const response = await openai.chat.completions.create({
    model: "gpt-4-0125-preview",
    messages: [
      {
        role: "system",
        content: prompt,
      },
      {
        role: "user",
        content: question,
      },
    ],
  });
  const sql = response.choices[0].message.content;
}
```

The prompt has been stripped down from it's original to make it easier to understand. The specific notes I used may not work for your use case so notes and rules should be custom made based on what provides you the best output.

There is a critical step in ensuring that questions that include semantic searches are included. For example if you search for transactions that are related to electronics stores. This here isn't actual PGVector syntax. The reason for this format is that it's easy for GPT to understand and provide consistent queries. To parse this syntax into a proper query, we use the following code.

```typescript
const embeddingSearch = sql!.match(/EMBEDDINGSEARCHWHERE\((.+), (.+)\)/g);

// find the first opening bracket and then get the string inside up to the comma
const searchType = embeddingSearch[0].match(/\((.+),/g)![0].slice(1, -1);
const searchQuery = embeddingSearch[0].match(/, (.+)\)/g)![0].slice(2, -2);

// from searchType remove anything that is not a letter
const cleanSearchType = searchType.replace(/[^a-zA-Z]/g, "");
const cleanSearchQuery = searchQuery.replace(/[^a-zA-Z]/g, "");

// Turn the word we are searching for into an embedding using OpenAI's embedding API
const searchQueryEmbedding = await getEmbedding(cleanSearchQuery);

// The proper SQL query
let newSql = "";

if (cleanSearchType === "category") {
  newSql = sql!.replace(
    /EMBEDDINGSEARCHWHERE\((.+), (.+)\)/g,
    "categoryEmbedding <-> $$1 < 0.5"
  );
}

if (cleanSearchType === "vendor") {
  newSql = sql!.replace(
    /EMBEDDINGSEARCHWHERE\((.+), (.+)\)/g,
    "vendorEmbedding <-> $$1 < 0.5"
  );
}
```

Once we have the query we run it and get all relevant data. Here is an example of what our data looks up to this point

- Question: `What was my total spending for the month on electronics`
- Original query: `SELECT SUM(total) FROM transactions WHERE businessid = '1234' AND EMBEDDINGSEARCH("category", "electronics")`
- Real query: `SELECT SUM(total) FROM transactions WHERE businessid = '1234' AND categoryEmbeding <-> $$1 < 0.5`;

Note the reason for $$1 instead of $1 is having a single dollar sign messes up the regex.

### Providing Context

Once this is done we provide that output from the query as context into a new prompt. This prompt is much simpler. All you really need to do is tell GPT what sort of data this is and to answer the user's question.

```typescript
const prompt = `
    Context
    ${JSON.stringify(cleanedData)}
    
    You are a ... Your job is to answer ... questions based on the above data.
    
    The goal is to provide the user with a clean and easy to understand output. While the context may contain detailed information, the user should only see information that is directly relevant to their question.
    
    Notes
    `;
const res = await model.chat.completions.create({
  messages: [
    {
      role: "system",
      content: getChatbotPrompt(cleanedData),
    },
    ...history,
    {
      role: "user",
      content: question,
    },
  ],
  model: "gpt-4-0125-preview",
});
```

Once again, this prompt has been stripped from the original. Replace elipses with information about your data and add rules as needed.

## Summary

This is basically it. It's a super simple process that makes use of two separate prompts. In reality, you will likely use several extra steps for retrieving and processing data. At [Monitico](https://www.moniti.co/) the pipeline for our chatbot currently has three prompts with multiple processing steps inbetween and is growing by the day.

If you have any questions or would like to discuss on how to optimize these processes, please reach out to me on [Twitter](https://twitter.com/notavp)
