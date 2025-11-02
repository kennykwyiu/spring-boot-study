# Spring AI Embedding Test — Step-by-Step Guide

### Overview

This page walks you through creating and running a minimal Spring AI embedding test that computes cosine similarity and Euclidean distance between a query and candidate texts, prints rankings, and helps you validate RAG retrieval behavior.

---

### 1) Project setup

- Create a Spring Boot project (3.3+) with Java 17 or later.
- Add Spring AI Ollama dependency.

```xml
<!-- pom.xml -->
<project>
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.example</groupId>
  <artifactId>embedding-test</artifactId>
  <version>0.0.1-SNAPSHOT</version>
  <properties>
    <java.version>17</java.version>
    <spring-boot.version>3.3.4</spring-boot.version>
  </properties>
  <dependencies>
    <dependency>
      <groupId>[org.springframework.ai](http://org.springframework.ai)</groupId>
      <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
      <version>0.9.0</version>
    </dependency>
    <dependency>
      <groupId>org.springframework.boot</groupId>
      <artifactId>spring-boot-starter-test</artifactId>
      <scope>test</scope>
    </dependency>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>5.10.3</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
  <build>
    <plugins>
      <plugin>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-maven-plugin</artifactId>
        <version>${spring-boot.version}</version>
      </plugin>
    </plugins>
  </build>
</project>
```

- Ensure Ollama is running locally and the models are pulled:
    - Base URL: [http://localhost:11434](http://localhost:11434)
    - Chat model: gpt-oss:20b (or your choice)
    - Embedding model: nomic-embed-text (or bge-m3, multilingual-e5, etc.)

---

### 2) Configuration (application.yml)

```yaml
spring:
  application:
    name: kenny-ai
  ai:
    ollama:
      base-url: [http://localhost:11434](http://localhost:11434)
      chat:
        model: gpt-oss:20b
      embedding:
        options:
          model: nomic-embed-text
```

Notes:

- If you switch to another embedder (e.g., bge-m3), update model accordingly.
- Keep one embedding model consistent across indexing and querying.

---

### 3) Vector utilities (cosine + Euclidean)

Use cosine similarity for ranking. Euclidean distance is equivalent on L2-normalized vectors; we expose both for inspection.

```java
package [com.example.ai](http://com.example.ai).util;

public class VectorDistanceUtils {
    private VectorDistanceUtils() {}
    private static final double EPSILON = 1e-12;

    public static double euclideanDistance(float[] a, float[] b) {
        validate(a, b);
        double sum = 0.0;
        for (int i = 0; i < a.length; i++) {
            double d = a[i] - b[i];
            sum += d * d;
        }
        return Math.sqrt(sum);
    }

    /** Cosine similarity in [-1, 1]. */
    public static double cosineSimilarity(float[] a, float[] b) {
        validate(a, b);
        double dot = 0.0, na = 0.0, nb = 0.0;
        for (int i = 0; i < a.length; i++) {
            dot += a[i] * b[i];
            na  += a[i] * a[i];
            nb  += b[i] * b[i];
        }
        na = Math.sqrt(na);
        nb = Math.sqrt(nb);
        if (na < EPSILON || nb < EPSILON) {
            throw new IllegalArgumentException("Vectors cannot be zero vectors");
        }
        double sim = dot / (na * nb);
        return Math.max(-1.0, Math.min(1.0, sim));
    }

    /** Cosine distance in [0, 2] = 1 - similarity. */
    public static double cosineDistance(float[] a, float[] b) {
        return 1.0 - cosineSimilarity(a, b);
    }

    private static void validate(float[] a, float[] b) {
        if (a == null || b == null) throw new IllegalArgumentException("Vectors cannot be null");
        if (a.length != b.length) throw new IllegalArgumentException("Vectors must have same dimension");
        if (a.length == 0) throw new IllegalArgumentException("Vectors cannot be empty");
    }
}
```

---

### 4) Test datasets

Provide a few datasets to compare cross-lingual behavior and domain relevance. You can toggle between them.

```java
package [com.example.ai](http://com.example.ai);

public final class TestDatasets {
    private TestDatasets() {}

    public static String query_en_conflicts() {
        return "global conflicts and ceasefire negotiations";
    }

    public static String[] docs_conflicts_mixed() {
        return new String[]{
            "哈马斯称加沙下阶段停火谈判仍在进行 以方尚未做出承诺",
            "乌克兰东部前线再起激烈交火 多地发布空袭警报",
            "土耳其、芬兰、瑞典与北约代表将继续就瑞典‘入约’问题进行谈判",
            "也门胡塞武装与政府军在港口城市再度爆发冲突",
            "联合国安理会呼吁有关各方保持克制 通过对话缓解地区紧张",
            "国家游泳中心（水立方）：恢复游泳、嬉水乐园等水上项目运营",
            "我国首次在空间站开展舱外辐射生物学暴露实验",
            "日本航空基地水井中检测出有机氟化物超标"
        };
    }

    public static String query_zh_conflicts() {
        return "国际 冲突 战争 停火 谈判";
    }
}
```

---

### 5) JUnit test (embedding + scoring + ranking)

This test embeds the query and docs, prints cosine similarity and Euclidean distance, and shows top-k.

```java
package [com.example.ai](http://com.example.ai);

import [com.example.ai](http://com.example.ai).util.VectorDistanceUtils;
import org.junit.jupiter.api.Test;
import [org.springframework.ai](http://org.springframework.ai).ollama.OllamaEmbeddingModel;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import java.util.*;
import [java.util.stream](http://java.util.stream).*;

@SpringBootTest
class EmbeddingRankingTests {

    @Autowired
    private OllamaEmbeddingModel embeddingModel;

    record ScoredDoc(String text, double cosSim, double l2) {}

    @Test
    void rankByCosineSimilarity() {
        // 1) Choose dataset
        String query = TestDatasets.query_en_conflicts();
        List<String> docs = Arrays.asList([TestDatasets.docs](http://TestDatasets.docs)_conflicts_mixed());

        // Optional: add prefixes that some dual-encoder models expect
        String queryInput = "search_query: " + query;
        List<String> docInputs = [docs.stream](http://docs.stream)().map(t -> "search_document: " + t).toList();

        // 2) Embed
        float[] qv = embeddingModel.embed(queryInput);
        List<float[]> dvs = embeddingModel.embed(docInputs);

        // 3) Score
        List<ScoredDoc> scored = IntStream.range(0, docs.size()).mapToObj(i -> {
            double cos = VectorDistanceUtils.cosineSimilarity(qv, dvs.get(i));
            double l2  = VectorDistanceUtils.euclideanDistance(qv, dvs.get(i));
            return new ScoredDoc(docs.get(i), cos, l2);
        }).collect(Collectors.toList());

        // 4) Sort by cosine similarity desc (higher is better)
        List<ScoredDoc> top = [scored.stream](http://scored.stream)()
            .sorted(Comparator.comparingDouble(ScoredDoc::cosSim).reversed())
            .limit(5)
            .toList();

        // 5) Print
        System.out.println("Query: " + query);
        System.out.println("--- Cosine similarity (desc) / L2 distance ---");
        top.forEach(s -> {
            System.out.printf([Locale.US](http://Locale.US), "cos=%.4f  L2=%.4f  | %s%n", s.cosSim(), s.l2(), s.text());
        });

        // 6) Sanity checks
        System.out.println("Self-check (query vs. query):");
        System.out.printf([Locale.US](http://Locale.US), "cos=%.4f  L2=%.4f%n",
                VectorDistanceUtils.cosineSimilarity(qv, qv),
                VectorDistanceUtils.euclideanDistance(qv, qv));
    }
}
```

---

### 6) Interpreting results

- Expect self vs self: cosine ≈ 1.0, L2 ≈ 0.0.
- Cross-lingual queries often show lower cosine values; try the Chinese query variant to compare.
- If embeddings are unit-normalized, L2^2 ≈ 2(1 − cosine). A cosine of 0.40 implies L2 ≈ sqrt(1.2) ≈ 1.095.

---

### 7) Improving retrieval quality

- Add dual-encoder prefixes: "search_query:" for queries, "search_document:" for docs.
- Unify language: translate query to Chinese or docs to English before embedding.
- Try stronger multilingual models on CPU: bge-m3, multilingual-e5. On APIs: text-embedding-3-large, Cohere multilingual.
- Use cosine similarity consistently for ANN indexes and scoring.

---

### 8) Next steps (RAG integration)

- After ranking, take top-k documents and pass them as context to the LLM prompt.
- Optionally hybrid-retrieve: combine BM25 keyword scores with embedding cosine similarity.
- Log scores and chosen docs for evaluation and future fine-tuning.