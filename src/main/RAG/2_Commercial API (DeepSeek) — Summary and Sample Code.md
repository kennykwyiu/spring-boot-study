# Commercial API (DeepSeek) — Summary and Sample Code

### Commercial API (DeepSeek) — Summary

Run large models without a GPU by calling a provider’s OpenAI‑compatible API.

- Console: [https://platform.deepseek.com/](https://platform.deepseek.com/)
- Base URL: [https://api.deepseek.com](https://api.deepseek.com)
- Steps: register, create an application and API key, then call the chat completions endpoint.

---

### curl

```bash
curl [https://api.deepseek.com/chat/completions](https://api.deepseek.com/chat/completions) \
  -H "Authorization: Bearer $DEEPSEEK_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "deepseek-chat",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Hello"}
    ],
    "stream": false
  }'
```

### Python (openai library)

```python
from openai import OpenAI

client = OpenAI(
    base_url="[https://api.deepseek.com](https://api.deepseek.com)",
    api_key="YOUR_API_KEY",
)

resp = [client.chat](http://client.chat).completions.create(
    model="deepseek-chat",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Hello"}
    ],
    temperature=0.7,
)
print(resp.choices[0].message.content)
```

### Node.js

```jsx
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "[https://api.deepseek.com](https://api.deepseek.com)",
  apiKey: process.env.DEEPSEEK_API_KEY,
});

const res = await [client.chat](http://client.chat).completions.create({
  model: "deepseek-chat",
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "Hello" },
  ],
});
console.log(res.choices[0].message.content);
```

### Java

```java
import [java.net](http://java.net).http.*;
import [java.net](http://java.net).URI;

public class DeepseekChat {
  public static void main(String[] args) throws Exception {
    String body = """
    {
      "model": "deepseek-chat",
      "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Hello"}
      ],
      "stream": false
    }
    """;

    HttpClient http = HttpClient.newHttpClient();
    HttpRequest req = HttpRequest.newBuilder()
        .uri(URI.create("[https://api.deepseek.com/chat/completions](https://api.deepseek.com/chat/completions)"))
        .header("Authorization", "Bearer " + System.getenv("DEEPSEEK_API_KEY"))
        .header("Content-Type", "application/json")
        .POST(HttpRequest.BodyPublishers.ofString(body))
        .build();

    HttpResponse<String> res = http.send(req, HttpResponse.BodyHandlers.ofString());
    System.out.println(res.body());
  }
}
```