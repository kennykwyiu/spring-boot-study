# Java: HTTP call + parse XML into DTO (JAXB and Jackson)

> End-to-end example: make an HTTP call that returns XML, then parse into a Java DTO using either JAXB or Jackson-XML.
> 

### Example XML response

```xml
<?xml version="1.0" encoding="UTF-8"?>
<order id="123" status="SHIPPED">
	<customer>
		<name>Kenny</name>
		<email>[kenny@sample.com](mailto:kenny@sample.com)</email>
	</customer>
	<items>
		<item>
			<sku>ABC-001</sku>
			<qty>2</qty>
			<price>9.99</price>
		</item>
		<item>
			<sku>XYZ-777</sku>
			<qty>1</qty>
			<price>25.00</price>
		</item>
	</items>
	<createdAt>2025-10-09T13:21:45Z</createdAt>
</order>
```

---

### DTOs (shared shape)

```java
package com.example.dto;

import java.time.OffsetDateTime;
import java.util.List;

public class Order {
	public String id;
	public String status;
	public Customer customer;
	public List<Item> items;
	public OffsetDateTime createdAt;

	public static class Customer {
		public String name;
		public String email;
	}

	public static class Item {
		public String sku;
		public int qty;
		public double price;
	}
}
```

---

### Option A: JAXB (Jakarta XML Binding) + Java 11 HttpClient

Dependencies (Maven)

```xml
<dependencies>
	<dependency>
		<groupId>jakarta.xml.bind</groupId>
		<artifactId>jakarta.xml.bind-api</artifactId>
		<version>4.0.2</version>
	</dependency>
	<dependency>
		<groupId>org.glassfish.jaxb</groupId>
		<artifactId>jaxb-runtime</artifactId>
		<version>4.0.5</version>
	</dependency>
</dependencies>
```

Annotated DTO (no XML namespace)

```java
package com.example.dto;

import jakarta.xml.bind.annotation.*;
import jakarta.xml.bind.annotation.adapters.XmlJavaTypeAdapter;
import java.time.OffsetDateTime;
import java.util.List;

@XmlRootElement(name = "order")
@XmlAccessorType(XmlAccessType.FIELD)
public class OrderJaxb {
	@XmlAttribute public String id;
	@XmlAttribute public String status;

	@XmlElement public Customer customer;

	@XmlElementWrapper(name = "items")
	@XmlElement(name = "item")
	public List<Item> items;

	@XmlElement
	@XmlJavaTypeAdapter(OffsetDateTimeAdapter.class)
	public OffsetDateTime createdAt;

	@XmlAccessorType(XmlAccessType.FIELD)
	public static class Customer {
		@XmlElement public String name;
		@XmlElement public String email;
	}

	@XmlAccessorType(XmlAccessType.FIELD)
	public static class Item {
		@XmlElement public String sku;
		@XmlElement public int qty;
		@XmlElement public double price;
	}
}
```

XmlAdapter for OffsetDateTime

```java
package com.example.jaxb;

import jakarta.xml.bind.annotation.adapters.XmlAdapter;
import java.time.OffsetDateTime;

public class OffsetDateTimeAdapter extends XmlAdapter<String, OffsetDateTime> {
	@Override public OffsetDateTime unmarshal(String v) { return OffsetDateTime.parse(v); }
	@Override public String marshal(OffsetDateTime v) { return v != null ? v.toString() : null; }
}
```

HTTP call and unmarshal

```java
package com.example;

import com.example.dto.OrderJaxb;
import jakarta.xml.bind.JAXBContext;
import jakarta.xml.bind.Unmarshaller;

import [java.net](http://java.net).URI;
import [java.net](http://java.net).http.HttpClient;
import [java.net](http://java.net).http.HttpRequest;
import [java.net](http://java.net).http.HttpResponse;

public class OrderClientJaxb {
	public OrderJaxb fetchOrder(String url) throws Exception {
		HttpClient client = HttpClient.newHttpClient();
		HttpRequest req = HttpRequest.newBuilder()
			.uri(URI.create(url))
			.header("Accept", "application/xml")
			.GET()
			.build();
		HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
		if (resp.statusCode() / 100 != 2) throw new RuntimeException("HTTP " + resp.statusCode() + ": " + resp.body());

		JAXBContext ctx = JAXBContext.newInstance(OrderJaxb.class);
		Unmarshaller um = ctx.createUnmarshaller();
		return (OrderJaxb) um.unmarshal(new [java.io](http://java.io).StringReader(resp.body()));
	}
}
```

---

### Option B: Jackson XML (simple and flexible)

Dependencies (Maven)

```xml
<dependencies>
	<dependency>
		<groupId>com.fasterxml.jackson.dataformat</groupId>
		<artifactId>jackson-dataformat-xml</artifactId>
		<version>2.17.2</version>
	</dependency>
	<dependency>
		<groupId>com.fasterxml.jackson.core</groupId>
		<artifactId>jackson-databind</artifactId>
		<version>2.17.2</version>
	</dependency>
	<dependency>
		<groupId>com.fasterxml.jackson.datatype</groupId>
		<artifactId>jackson-datatype-jsr310</artifactId>
		<version>2.17.2</version>
	</dependency>
</dependencies>
```

Annotated DTO (no XML namespace)

```java
package com.example.dto;

import com.fasterxml.jackson.dataformat.xml.annotation.*;
import com.fasterxml.jackson.annotation.JsonFormat;
import java.time.OffsetDateTime;
import java.util.List;

@JacksonXmlRootElement(localName = "order")
public class OrderXml {
	@JacksonXmlProperty(isAttribute = true) public String id;
	@JacksonXmlProperty(isAttribute = true) public String status;

	@JacksonXmlProperty(localName = "customer")
	public Customer customer;

	@JacksonXmlElementWrapper(localName = "items")
	@JacksonXmlProperty(localName = "item")
	public java.util.List<Item> items;

	@JsonFormat(pattern = "yyyy-MM-dd'T'HH:mm:ssX")
	@JacksonXmlProperty(localName = "createdAt")
	public OffsetDateTime createdAt;

	public static class Customer {
		@JacksonXmlProperty(localName = "name")
		public String name;
		@JacksonXmlProperty(localName = "email")
		public String email;
	}

	public static class Item {
		@JacksonXmlProperty(localName = "sku")
		public String sku;
		@JacksonXmlProperty(localName = "qty")
		public int qty;
		@JacksonXmlProperty(localName = "price")
		public double price;
	}
}
```

HTTP call and parse with XmlMapper

```java
package com.example;

import com.example.dto.OrderXml;
import com.fasterxml.jackson.dataformat.xml.XmlMapper;
import com.fasterxml.jackson.databind.DeserializationFeature;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;

import [java.net](http://java.net).URI;
import [java.net](http://java.net).http.HttpClient;
import [java.net](http://java.net).http.HttpRequest;
import [java.net](http://java.net).http.HttpResponse;

public class OrderClientJackson {
	private final XmlMapper xml = XmlMapper.builder()
		.addModule(new JavaTimeModule())
		.configure([DeserializationFeature.FAIL](http://DeserializationFeature.FAIL)_ON_UNKNOWN_PROPERTIES, false)
		.build();

	public OrderXml fetchOrder(String url) throws Exception {
		HttpClient client = HttpClient.newHttpClient();
		HttpRequest req = HttpRequest.newBuilder()
			.uri(URI.create(url))
			.header("Accept", "application/xml")
			.GET()
			.build();
		HttpResponse<String> resp = client.send(req, HttpResponse.BodyHandlers.ofString());
		if (resp.statusCode() / 100 != 2) throw new RuntimeException("HTTP " + resp.statusCode() + ": " + resp.body());
		return xml.readValue(resp.body(), OrderXml.class);
	}
}
```

---

### Notes

- If your real API uses a default xmlns, add the same namespace to annotations as shown previously.
- Dates: Jackson JavaTimeModule handles ISO-8601. JAXB needs an XmlAdapter or map as String.
- Unknown fields: For Jackson, set FAIL_ON_UNKNOWN_PROPERTIES=false to tolerate extra tags.
- Unit test a saved sample XML to verify mappings.