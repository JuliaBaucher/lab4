# Julia Baucher – AI CV Chatbot (OpenAI + AWS + GitHub Pages)

This project provides a secure AI assistant embedded into Julia’s public online CV.  
The chatbot answers questions about her profile **without exposing the OpenAI API key on the frontend**.

👉 Live website: **https://juliabaucher.github.io/lab4**

---

## ✨ Core Features

| Feature | Description |  
|--------|-------------|
| **Conversational Chatbot** | Uses GPT-4o-mini from OpenAI |  
| **Secure Back-End on AWS** | API key stored safely in Lambda (not public) |  
| **Public Frontend on GitHub Pages** | No hosting cost |  
| **CORS Protected** | Only Julia’s website can call the API |  
| **Fully Serverless** | No servers, no maintenance |  

---  

## 🏛️ System Architecture (Simple + Secure)

GitHub Pages (index.html)  
│  
│ fetch POST https://xxxx.execute-api.eu-north-1.amazonaws.com/prod/chat  


▼  
API Gateway (HTTP API)  
▼  
AWS Lambda: JuliaBaucher_CV-backend  
│ - Validates message  
│ - Sends request to OpenAI  
│ - Returns "reply" JSON  
▼  
OpenAI API (gpt-4o-mini)  
 

---

## 🔐 Security Design

- OpenAI API Key is **never visible** to users or in HTML code.
- Key is stored only in:
Lambda → Configuration → Environment Variables → OPENAI_API_KEY

- CORS limits usage to only this domain: https://juliabaucher.github.io


---

## 🧩 Backend Implementation (Final Working Version)

### 1️⃣ Create the Lambda Function

📌 **Runtime must be Node.js 22.x**

📌 Function name example:JuliaBaucher_CV-backend


### 2️⃣ Add Environment Variable

| Key | Value |
|-----|-------|
| `OPENAI_API_KEY` | your secret key from https://platform.openai.com |

### 3️⃣ Paste this **final working code**

> ⚠️ Do not modify the `jsonResponse()` — it enforces CORS and is required for GitHub Pages.

```javascript
async function fetchWithTimeout(url, options = {}, timeoutMs = 8000) {
  const controller = new AbortController();
  const id = setTimeout(() => controller.abort(), timeoutMs);
  try {
    console.log("About to call fetch:", url);
    const res = await fetch(url, { ...options, signal: controller.signal });
    console.log("Fetch returned status:", res.status);
    return res;
  } catch (err) {
    console.error("Fetch failed/aborted:", err);
    throw err;
  } finally {
    clearTimeout(id);
  }
}

async function callOpenAI(userMessage, systemMessage) {
  const apiKey = process.env.OPENAI_API_KEY;
  console.log("callOpenAI start. userMessage length:", userMessage.length);

  const body = {
    model: "gpt-4o-mini",
    input: [
      { role: "system", content: systemMessage },
      { role: "user", content: userMessage }
    ]
  };

  const response = await fetchWithTimeout(
    "https://api.openai.com/v1/responses",
    {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${apiKey}`,
      },
      body: JSON.stringify(body),
    },
    8000
  );

  if (!response.ok) {
    const errText = await response.text();
    console.error("OpenAI error:", errText);
    throw new Error("OpenAI API call failed");
  }

  const data = await response.json();
  console.log("OpenAI raw response:", JSON.stringify(data));

  // ✅ NEW PARSING LOGIC
  let assistantText = "";

  if (Array.isArray(data.output)) {
    for (const messageChunk of data.output) {
      // Each messageChunk looks like { type: "message", content: [ ... ], role: "assistant", ... }
      if (Array.isArray(messageChunk.content)) {
        for (const contentPiece of messageChunk.content) {
          // Each contentPiece may look like { type: "output_text", text: "actual answer..." }
          if (
            contentPiece.type === "output_text" &&
            typeof contentPiece.text === "string"
          ) {
            assistantText += contentPiece.text;
          }
        }
      }
    }
  }

  // Fallback: some responses also include output_text at top level
  if (!assistantText && data.output_text) {
    assistantText = data.output_text;
  }

  assistantText = assistantText.trim();
  console.log("assistantText after parse:", assistantText);

  return assistantText;
}

export const handler = async (event) => {
  try {
    console.log("Lambda invoked with method:", event.requestContext?.http?.method);

    const method = event.requestContext?.http?.method || "GET";

    if (method === "OPTIONS") {
      console.log("Handling OPTIONS");
      return jsonResponse(200, { ok: true });
    }

    if (method === "POST") {
      console.log("Handling POST");

      const body = event.body ? JSON.parse(event.body) : {};
      console.log("Request body parsed:", JSON.stringify(body));

      const userMessage = body.message || "";
      if (!userMessage) {
        console.warn("No user message provided");
        return jsonResponse(400, { error: "No message provided" });
      }

      const systemMessage =
        "You are a helpful AI assistant on Julia Baucher’s CV website. Answer politely and concisely.";

      const answer = await callOpenAI(userMessage, systemMessage);

      console.log("Final answer length:", answer.length);

      return jsonResponse(200, { reply: answer });
    }

    console.warn("Method not allowed:", method);
    return jsonResponse(405, { error: "Method not allowed" });

  } catch (err) {
    console.error("Lambda error:", err);
    return jsonResponse(200, { reply: "" });
  }
};

function jsonResponse(statusCode, obj) {
  return {
    statusCode,
    headers: {
      "Content-Type": "application/json",
      "Access-Control-Allow-Origin": "https://juliabaucher.github.io",
      "Access-Control-Allow-Headers": "Content-Type",
      "Access-Control-Allow-Methods": "POST,OPTIONS",
    },
    body: JSON.stringify(obj),
  };
}
```
🌐 API Gateway Configuration
🔹 Select: HTTP API (not REST API)

📌 Route:

POST /chat

📌 Integration: Lambda function → JuliaBaucher_CV-backend

📌 Enable CORS with:

Origin: https://juliabaucher.github.io
Methods: POST, OPTIONS
Headers: Content-Type


📌 Deploy Stage name:prod

🎨 Frontend (Final Working Code Snippet)

In index.html, use:

```
const r = await fetch('https://XXXX.execute-api.eu-north-1.amazonaws.com/prod/chat', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ message: userText })
});
const data = await r.json();
addBubble('assistant', data.reply || '(No reply)');
```

🧪 Testing Guide (Non-Technical)
Test	Expected Result  
Open website	Chat icon appears  
Click chat	Greeting bubble appears  
Ask a question	Answer within ~3 seconds  
Check browser console	No CORS or 500 errors  
Check CloudWatch logs	Response text visible  
Ask again	Still works (key is secure)  
💾 Backup Guidelines  

 
To recreate everything if AWS is empty:


Step	What to Backup  
🔑 1	Your OpenAI API key (store safely)  
💾 2	Lambda ZIP or code file (index.mjs)  
🌍 3	API Gateway URL path /prod/chat  
🌐 4	CORS Origin: https://juliabaucher.github.io  
💡 5	Runtime version: Node.js 22.x  




