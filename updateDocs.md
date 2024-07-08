
Here’s how you can update your Spring Boot application to use the new Elasticsearch Java Client:

1. **Add Elasticsearch dependencies:**

   Update your `pom.xml` to include the new Elasticsearch client dependency.

   ```xml
   <dependency>
       <groupId>co.elastic.clients</groupId>
       <artifactId>elasticsearch-java</artifactId>
       <version>8.5.0</version>
   </dependency>
   ```

2. **Configure Elasticsearch client:**

   Create a configuration class to set up the `ElasticsearchClient`.

   ```java
   import co.elastic.clients.elasticsearch.ElasticsearchClient;
   import co.elastic.clients.elasticsearch.ElasticsearchTransport;
   import co.elastic.clients.transport.rest_client.RestClientTransport;
   import co.elastic.clients.transport.TransportOptions;
   import co.elastic.clients.transport.Transport;
   import co.elastic.clients.transport.rest_client.RestClient;
   import co.elastic.clients.json.jackson.JacksonJsonpMapper;
   import org.apache.http.auth.AuthScope;
   import org.apache.http.auth.UsernamePasswordCredentials;
   import org.apache.http.impl.client.BasicCredentialsProvider;
   import org.apache.http.impl.nio.client.HttpAsyncClientBuilder;
   import org.elasticsearch.client.RestClientBuilder;
   import org.springframework.beans.factory.annotation.Value;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;

   @Configuration
   public class ElasticsearchConfig {

       @Value("${spring.elasticsearch.uris}")
       private String elasticsearchUri;

       @Value("${spring.elasticsearch.username}")
       private String username;

       @Value("${spring.elasticsearch.password}")
       private String password;

       @Bean
       public ElasticsearchClient elasticsearchClient() {
           final BasicCredentialsProvider credentialsProvider = new BasicCredentialsProvider();
           credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(username, password));

           RestClientBuilder builder = RestClient.builder(HttpHost.create(elasticsearchUri))
                   .setHttpClientConfigCallback(new RestClientBuilder.HttpClientConfigCallback() {
                       @Override
                       public HttpAsyncClientBuilder customizeHttpClient(HttpAsyncClientBuilder httpClientBuilder) {
                           return httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider);
                       }
                   });

           RestClient restClient = builder.build();
           ElasticsearchTransport transport = new RestClientTransport(restClient, new JacksonJsonpMapper());
           return new ElasticsearchClient(transport);
       }
   }
   ```

3. **Create a service class to interact with Elasticsearch:**

   Create a service class that uses the `ElasticsearchClient` to interact with Elasticsearch.

   ```java
   import co.elastic.clients.elasticsearch.ElasticsearchClient;
   import co.elastic.clients.elasticsearch.core.GetRequest;
   import co.elastic.clients.elasticsearch.core.GetResponse;
   import co.elastic.clients.elasticsearch.core.IndexRequest;
   import co.elastic.clients.elasticsearch.core.IndexResponse;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.stereotype.Service;

   import java.io.IOException;
   import java.util.Map;

   @Service
   public class ElasticsearchService {

       @Autowired
       private ElasticsearchClient client;

       public Map<String, Object> getDocumentById(String index, String id) throws IOException {
           GetRequest getRequest = GetRequest.of(g -> g.index(index).id(id));
           GetResponse<Map> getResponse = client.get(getRequest, Map.class);
           return getResponse.source();
       }

       public String indexDocument(String index, String id, Map<String, Object> document) throws IOException {
           IndexRequest<Map<String, Object>> indexRequest = IndexRequest.of(i -> i.index(index).id(id).document(document));
           IndexResponse indexResponse = client.index(indexRequest);
           return indexResponse.id();
       }
   }
   ```

4. **Create a controller to expose REST endpoints:**

   Create a controller class to expose REST endpoints that interact with Elasticsearch.

   ```java
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.web.bind.annotation.*;

   import java.io.IOException;
   import java.util.Map;

   @RestController
   @RequestMapping("/api/elasticsearch")
   public class ElasticsearchController {

       @Autowired
       private ElasticsearchService elasticsearchService;

       @GetMapping("/document/{index}/{id}")
       public Map<String, Object> getDocumentById(@PathVariable String index, @PathVariable String id) throws IOException {
           return elasticsearchService.getDocumentById(index, id);
       }

       @PostMapping("/document/{index}/{id}")
       public String indexDocument(@PathVariable String index, @PathVariable String id, @RequestBody Map<String, Object> document) throws IOException {
           return elasticsearchService.indexDocument(index, id, document);
       }
   }
   ```

5. **Test the endpoints:**

   - To get a document by ID:
     ```
     GET http://localhost:8080/api/elasticsearch/document/{index}/{id}
     ```

   - To index a document:
     ```
     POST http://localhost:8080/api/elasticsearch/document/{index}/{id}
     {
       "field1": "value1",
       "field2": "value2"
     }
     ```

