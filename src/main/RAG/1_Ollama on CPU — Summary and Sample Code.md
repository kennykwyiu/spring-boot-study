# Ollama on CPU — Summary and Sample Code

### Ollama on CPU — Summary

Use large models on CPU with Ollama.

- Install
    - curl -fsSL [https://ollama.com/install.sh](https://ollama.com/install.sh) | sh
- Run a small CPU‑friendly model
    - ollama run qwen2:1.5b
- Optional pull
    - ollama pull qwen2:1.5b
- Local OpenAI‑compatible endpoint
    - [http://localhost:11434/v1](http://localhost:11434/v1)

---

### Python (OpenAI‑compatible client)

```python
from openai import OpenAI

client = OpenAI(
    base_url="[http://localhost:11434/v1](http://localhost:11434/v1)",
    api_key="ollama",  # any non-empty string
)

resp = [client.chat](http://client.chat).completions.create(
    model="qwen2:1.5b",
    messages=[{"role": "user", "content": "Say this is a test"}],
    temperature=0.7,
)
print(resp.choices[0].message.content)
```

### Python (plain REST)

```python
import requests, json

url = "[http://localhost:11434/v1/chat/completions](http://localhost:11434/v1/chat/completions)"
headers = {"Authorization": "Bearer ollama", "Content-Type": "application/json"}
data = {
    "model": "qwen2:1.5b",
    "messages": [{"role": "user", "content": "Say this is a test"}],
    "stream": False
}
print([requests.post](http://requests.post)(url, headers=headers, data=json.dumps(data)).json()["choices"][0]["message"]["content"])
```

### Node.js

```jsx
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "[http://localhost:11434/v1](http://localhost:11434/v1)",
  apiKey: "ollama",
});

const res = await [client.chat](http://client.chat).completions.create({
  model: "qwen2:1.5b",
  messages: [{ role: "user", content: "Say this is a test" }],
});
console.log(res.choices[0].message.content);
```

### Java

```java
import [java.net](http://java.net).http.*;
import [java.net](http://java.net).URI;

public class OllamaChat {
  public static void main(String[] args) throws Exception {
    String body = """
    {
      "model": "qwen2:1.5b",
      "messages": [{"role":"user","content":"Say this is a test"}],
      "stream": false
    }
    """;

    HttpClient http = HttpClient.newHttpClient();
    HttpRequest req = HttpRequest.newBuilder()
        .uri(URI.create("[http://localhost:11434/v1/chat/completions](http://localhost:11434/v1/chat/completions)"))
        .header("Authorization", "Bearer ollama")
        .header("Content-Type", "application/json")
        .POST(HttpRequest.BodyPublishers.ofString(body))
        .build();

    HttpResponse<String> res = http.send(req, HttpResponse.BodyHandlers.ofString());
    System.out.println(res.body());
  }
}
```

---

### CPU Tips

- Prefer small or quantized models. Keep prompts and context concise.
- Match tokenizer and chat template. Set threads to physical cores.