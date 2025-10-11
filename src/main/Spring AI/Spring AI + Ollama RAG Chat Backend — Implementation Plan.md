# Spring AI + Ollama RAG Chat Backend — Implementation Plan

> Goal: Stand up a Spring Boot backend that uses Spring AI with Ollama for local LLM, RAG over your docs, and per-session chat history. Safe rollout with checkpoints.
> 

### Prerequisites

- Linux or macOS dev host with Docker optional
- JDK 21
- Maven 3.9+
- Git
- Ollama installed and running
- Postgres with pgvector (for production) or in-memory vector store for demo

### High-level architecture

- Spring Boot REST API
- Spring AI ChatClient backed by Ollama
- EmbeddingClient via Ollama for embeddings
- Vector store: start with in-memory, then switch to pgvector
- Chat memory: per-session conversation history

---

### Step 1 — Create project skeleton

- Scaffold a Spring Boot project
- Add dependencies in pom.xml
    - spring-boot-starter-web
    - spring-ai-ollama-spring-boot-starter
    - spring-ai-vectorstore-memory for demo
    - spring-ai-document-parsers for chunking
- Commit as baseline tag v0.1

### Step 2 — Configure models and application.yml

- Set server port 8080
- Configure [spring.ai](http://spring.ai).ollama.base-url: [http://localhost:11434](http://localhost:11434)
- Chat options: model llama3.1:8b, temperature 0.2
- Embedding model: all-minilm or nomic-embed-text
- Add commented Postgres config block for later
- Commit tag v0.2

### Step 3 — Pull models and verify Ollama

- ollama pull llama3.1:8b
- ollama pull all-minilm
- ollama list to verify
- Smoke test curl [http://localhost:11434](http://localhost:11434)
- Commit tag v0.3

### Step 4 — Wire Spring AI beans

- Create AiConfig with beans:
    - OllamaApi
    - ChatClient (OllamaChatClient)
    - EmbeddingClient (OllamaEmbeddingClient)
    - VectorStore (SimpleVectorStore for now)
    - ChatMemory + ChatMemoryAdvisor
- Boot app and confirm context loads
- Commit tag v0.4

### Step 5 — Implement RAG service

- Create RagService
    - ingest(String sourceName, String text)
    - retrieve(String query, int topK)
- Use TokenTextSplitter(512, 100) for chunking
- Unit test: ingest sample doc and assert retrieve returns chunks
- Commit tag v0.5

### Step 6 — Build REST endpoints

- POST /api/ingest: {sourceName, text}
- POST /api/chat: {sessionId, message}
- In /chat:
    - Retrieve topK=5 docs
    - Build system prompt with context
    - Call ChatClient with memoryAdvisor.withSessionId(sessionId)
    - Save user and assistant messages into memory
- Swagger or HTTPie examples in README
- Commit tag v0.6

### Step 7 — Local validation

- Start app and ingest demo text
- Ask query and a follow-up in same session to verify memory
- Validate RAG grounding: answers reference context, and say “not sure” when absent
- Commit tag v0.7

### Step 8 — Observability and guardrails

- Add request/response logging with redaction for secrets
- Add timeout settings for LLM calls
- Add max tokens and temperature controls via config
- Add prompt template constants and tests for them
- Commit tag v0.8

### Step 9 — Switch to pgvector (production)

- Install Postgres with pgvector extension
- Create db ai and user ai/ai
- Add spring-ai-pgvector-store starter
- Replace SimpleVectorStore with PgVectorStore bean
- Migrate existing in-memory content by re-ingesting sources
- Commit tag v0.9

### Step 10 — Packaging and deployment

- Build container image
- Health checks: /actuator/health
- Externalize config via env vars
- Rollout: dev → staging → prod with canary
- Commit tag v1.0

### Code examples

### 1) Maven dependencies (pom.xml)

```xml
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>spring-ai-ollama-rag</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <properties>
    <java.version>21</java.version>
    <spring-boot.version>3.3.3</spring-boot.version>
    <spring-ai.version>0.9.0</spring-ai.version>
  </properties>

  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-dependencies</artifactId>
        <version>${spring-boot.version}</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>

  <dependencies>
    <!-- Spring Boot Web -->
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring AI: Ollama chat + starter -->
    <dependency>
      <groupId>[org.springframework.ai](http://org.springframework.ai)</groupId>
      <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
      <version>${spring-ai.version}</version>
    </dependency>

    <!-- Vector store (demo: in-memory). For prod: pgvector starter below -->
    <dependency>
      <groupId>[org.springframework.ai](http://org.springframework.ai)</groupId>
      <artifactId>spring-ai-vectorstore-memory</artifactId>
      <version>${spring-ai.version}</version>
    </dependency>

    <!-- Optional parsers and splitters -->
    <dependency>
      <groupId>[org.springframework.ai](http://org.springframework.ai)</groupId>
      <artifactId>spring-ai-document-parsers</artifactId>
      <version>${spring-ai.version}</version>
    </dependency>

    <!-- For production vector store (comment in when switching) -->
    <!--
    <dependency>
      <groupId>[org.springframework.ai](http://org.springframework.ai)</groupId>
      <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
      <version>${spring-ai.version}</version>
    </dependency>
    <dependency>
      <groupId>org.postgresql</groupId>
      <artifactId>postgresql</artifactId>
    </dependency>
    -->
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
      </plugin>
    </plugins>
  </build>
</project>
```

### 2) application.yml

```yaml
server:
  port: 8080

spring:
  ai:
    ollama:
      base-url: [http://localhost:11434](http://localhost:11434)
      chat:
        options:
          model: llama3.1:8b
          temperature: 0.2
    embedding:
      ollama:
        options:
          model: all-minilm

# For pgvector (later):
# spring:
#   datasource:
#     url: jdbc:postgresql://[localhost:5432/ai](http://localhost:5432/ai)
#     username: ai
#     password: ai
#   ai:
#     vectorstore:
#       pgvector:
#         schema-name: public
#         table-name: rag_docs
```

### 3) Spring Boot main class

```java
package com.example;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class Application {
  public static void main(String[] args) {
    [SpringApplication.run](http://SpringApplication.run)(Application.class, args);
  }
}
```

### 4) AI configuration beans

```java
package [com.example.ai](http://com.example.ai).config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.beans.factory.annotation.Value;

import [org.springframework.ai.chat](http://org.springframework.ai.chat).ChatClient;
import [org.springframework.ai](http://org.springframework.ai).ollama.OllamaChatClient;
import [org.springframework.ai](http://org.springframework.ai).ollama.OllamaChatModel;
import [org.springframework.ai](http://org.springframework.ai).ollama.api.OllamaApi;

import [org.springframework.ai](http://org.springframework.ai).embedding.EmbeddingClient;
import [org.springframework.ai](http://org.springframework.ai).ollama.OllamaEmbeddingClient;

import [org.springframework.ai](http://org.springframework.ai).vectorstore.VectorStore;
import [org.springframework.ai](http://org.springframework.ai).vectorstore.SimpleVectorStore;

import [org.springframework.ai.chat](http://org.springframework.ai.chat).memory.ChatMemory;
import [org.springframework.ai.chat](http://org.springframework.ai.chat).memory.InMemoryChatMemoryStore;
import [org.springframework.ai.chat](http://org.springframework.ai.chat).memory.ChatMemoryAdvisor;

@Configuration
public class AiConfig {

  @Bean
  public OllamaApi ollamaApi(@Value("${[spring.ai](http://spring.ai).ollama.base-url}") String baseUrl) {
    return new OllamaApi(baseUrl);
  }

  @Bean
  public ChatClient chatClient(OllamaApi api) {
    return new OllamaChatClient(new OllamaChatModel(api));
  }

  @Bean
  public EmbeddingClient embeddingClient(OllamaApi api) {
    return new OllamaEmbeddingClient(api);
  }

  @Bean
  public VectorStore vectorStore(EmbeddingClient embeddingClient) {
    return new SimpleVectorStore(embeddingClient);
  }

  @Bean
  public ChatMemory chatMemory() {
    return new ChatMemory(new InMemoryChatMemoryStore());
  }

  @Bean
  public ChatMemoryAdvisor chatMemoryAdvisor(ChatMemory chatMemory) {
    return new ChatMemoryAdvisor(chatMemory);
  }
}
```

### 5) RAG service

```java
package [com.example.ai](http://com.example.ai).rag;

import java.util.List;
import org.springframework.stereotype.Service;
import [org.springframework.ai](http://org.springframework.ai).vectorstore.VectorStore;
import [org.springframework.ai](http://org.springframework.ai).document.Document;
import [org.springframework.ai](http://org.springframework.ai).document.Content;
import [org.springframework.ai](http://org.springframework.ai).document.splitter.TokenTextSplitter;
import [org.springframework.ai](http://org.springframework.ai).document.splitter.DocumentSplitter;

@Service
public class RagService {
  private final VectorStore vectorStore;
  private final DocumentSplitter splitter = new TokenTextSplitter(512, 100);

  public RagService(VectorStore vectorStore) {
    this.vectorStore = vectorStore;
  }

  public void ingest(String sourceName, String rawText) {
    var docs = splitter.apply(List.of(new Document(Content.from(rawText), sourceName)));
    vectorStore.add(docs);
  }

  public List<Document> retrieve(String query, int topK) {
    return vectorStore.similaritySearch(query, topK);
  }
}
```

### 6) REST controller with RAG + chat history

```java
package [com.example.ai](http://com.example.ai).web;

import java.util.List;
import [java.util.Map](http://java.util.Map);
import [java.util.stream](http://java.util.stream).Collectors;

import org.springframework.web.bind.annotation.*;
import org.springframework.http.MediaType;

import [org.springframework.ai.chat](http://org.springframework.ai.chat).ChatClient;
import [org.springframework.ai.chat](http://org.springframework.ai.chat).messages.SystemMessage;
import [org.springframework.ai.chat](http://org.springframework.ai.chat).messages.UserMessage;
import [org.springframework.ai.chat](http://org.springframework.ai.chat).prompt.Prompt;
import [org.springframework.ai.chat](http://org.springframework.ai.chat).prompt.SystemPromptTemplate;
import [org.springframework.ai.chat](http://org.springframework.ai.chat).memory.ChatMemory;
import [org.springframework.ai.chat](http://org.springframework.ai.chat).memory.ChatMemoryAdvisor;

import [com.example.ai](http://com.example.ai).rag.RagService;
import [org.springframework.ai](http://org.springframework.ai).document.Document;

@RestController
@RequestMapping("/api")
public class ChatController {
  private final ChatClient chat;
  private final RagService rag;
  private final ChatMemory memory;
  private final ChatMemoryAdvisor memoryAdvisor;

  public ChatController(ChatClient chat, RagService rag, ChatMemory memory, ChatMemoryAdvisor memoryAdvisor) {
    [this.chat](http://this.chat) = chat;
    this.rag = rag;
    this.memory = memory;
    this.memoryAdvisor = memoryAdvisor;
  }

  public record IngestReq(String sourceName, String text) {}
  public record ChatReq(String sessionId, String message) {}

  @PostMapping(value = "/ingest", consumes = MediaType.APPLICATION_JSON_VALUE)
  public Map<String, Object> ingest(@RequestBody IngestReq req) {
    rag.ingest(req.sourceName(), req.text());
    return Map.of("ok", true);
  }

  @PostMapping(value = "/chat", consumes = MediaType.APPLICATION_JSON_VALUE)
  public Map<String, Object> chat(@RequestBody ChatReq req) {
    List<Document> docs = rag.retrieve(req.message(), 5);
    String context = [docs.stream](http://docs.stream)()
      .map(d -> "- " + d.getContent().text())
      .collect(Collectors.joining("\n"));

    String sysTemplate = """
      You are a helpful assistant. Use the provided context to answer the user query.
      If the answer is not in the context, say you are not sure.
      Context:
      {context}
      """;

    String systemMsg = new SystemPromptTemplate(sysTemplate)
      .createMessage(Map.of("context", context))
      .getContent();

    var messages = List.of(
      new SystemMessage(systemMsg),
      new UserMessage(req.message())
    );

    var response = [chat.call](http://chat.call)(new Prompt(messages, memoryAdvisor.withSessionId(req.sessionId())));
    var answer = response.getResult().getOutput().getContent();

    // Persist history
    memory.addUserMessage(req.sessionId(), req.message());
    memory.addAssistantMessage(req.sessionId(), answer);

    return Map.of(
      "sessionId", req.sessionId(),
      "answer", answer,
      "contextDocCount", docs.size()
    );
  }
}
```

### 7) cURL smoke tests

```bash
# Ensure Ollama is running and models are pulled
ollama pull llama3.1:8b
ollama pull all-minilm

# Start the app
mvn spring-boot:run

# Ingest sample content
curl -X POST [http://localhost:8080/api/ingest](http://localhost:8080/api/ingest) \
  -H "Content-Type: application/json" \
  -d '{"sourceName":"notes-1","text":"Spring AI can connect to Ollama and do RAG via a vector store."}'

# Ask a question
curl -X POST [http://localhost:8080/api/chat](http://localhost:8080/api/chat) \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"sess-1","message":"How to use Ollama with Spring AI for RAG?"}'

# Follow-up (history applied)
curl -X POST [http://localhost:8080/api/chat](http://localhost:8080/api/chat) \
  -H "Content-Type: application/json" \
  -d '{"sessionId":"sess-1","message":"Summarize the steps again."}'
```

### 8) Optional: switch to pgvector beans

```java
// Replace vectorStore bean in AiConfig when moving to Postgres
// @Bean
// public VectorStore vectorStore(DataSource dataSource, EmbeddingClient embeddingClient) {
//   return new PgVectorStore(dataSource, embeddingClient, "rag_docs");
// }
```

---

### Validation checklist

- App starts cleanly, beans initialized
- /api/ingest returns ok for sample content
- /api/chat returns plausible, grounded answers
- Follow-up questions in same session leverage history
- Empty or irrelevant context produces "not sure" answer
- Load test: 20 rps sustained without timeouts on dev hardware

### Production hardening

- Switch to pgvector
- Add caching for retrieval results
- Add rate limiting and auth
- Structured logging and tracing
- Backups for Postgres
- Monitoring dashboards for latency, error rate, token usage

### Troubleshooting

- Ollama connection refused: confirm service at [http://localhost:11434](http://localhost:11434)
- Model not found: run `ollama pull <model>`
- Slow responses: reduce model size, increase CPU threads, lower topK
- Irrelevant answers: tune chunking size/overlap, improve system prompt

### Timeline (suggested)

- Day 1: Steps 1–4
- Day 2: Steps 5–7
- Day 3: Steps 8–9
- Day 4: Step 10 and production validation

### Useful commands

```bash
# Pull models
ollama pull llama3.1:8b
ollama pull all-minilm

# Start Spring Boot
env SPRING_PROFILES_ACTIVE=dev mvn spring-boot:run

# Ingest sample
curl -sS -X POST [http://localhost:8080/api/ingest](http://localhost:8080/api/ingest) \
  -H 'Content-Type: application/json' \
  -d '{"sourceName":"notes-1","text":"Spring AI + Ollama + RAG demo text."}'

# Ask question
curl -sS -X POST [http://localhost:8080/api/chat](http://localhost:8080/api/chat) \
  -H 'Content-Type: application/json' \
  -d '{"sessionId":"sess-1","message":"How to use RAG here?"}'

# Follow-up in same session (history applied)
curl -sS -X POST [http://localhost:8080/api/chat](http://localhost:8080/api/chat) \
  -H 'Content-Type: application/json' \
  -d '{"sessionId":"sess-1","message":"Summarize the steps again."}'
```