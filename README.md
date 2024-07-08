To securely access your Elasticsearch endpoint from a Spring Boot project, you'll need to add authentication credentials. Here's how you can do it:

1. **Configure Elasticsearch credentials:**

   Add the Elasticsearch username and password to your `application.properties` or `application.yml` file.

   **application.properties:**
   ```properties
   spring.elasticsearch.uris=http://<external-ip>:9200
   spring.elasticsearch.username=<your-username>
   spring.elasticsearch.password=<your-password>
   ```

   Replace `<external-ip>`, `<your-username>`, and `<your-password>` with the actual values for your Elasticsearch instance.

2. **Modify the Elasticsearch configuration class:**

   Update your configuration class to use the credentials for the `RestHighLevelClient`.

   ```java
   import org.apache.http.auth.AuthScope;
   import org.apache.http.auth.UsernamePasswordCredentials;
   import org.apache.http.impl.client.BasicCredentialsProvider;
   import org.apache.http.impl.nio.client.HttpAsyncClientBuilder;
   import org.elasticsearch.client.RestClient;
   import org.elasticsearch.client.RestClientBuilder;
   import org.elasticsearch.client.RestHighLevelClient;
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
       public RestHighLevelClient client() {
           final BasicCredentialsProvider credentialsProvider = new BasicCredentialsProvider();
           credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(username, password));

           RestClientBuilder builder = RestClient.builder(HttpHost.create(elasticsearchUri))
                   .setHttpClientConfigCallback(new RestClientBuilder.HttpClientConfigCallback() {
                       @Override
                       public HttpAsyncClientBuilder customizeHttpClient(HttpAsyncClientBuilder httpClientBuilder) {
                           return httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider);
                       }
                   });

           return new RestHighLevelClient(builder);
       }
   }
   ```

3. **Using the RestHighLevelClient in your service:**

   The service and controller classes remain the same as before. They will use the configured `RestHighLevelClient` with authentication to interact with Elasticsearch.

4. **Complete Example:**

   **application.properties:**
   ```properties
   spring.elasticsearch.uris=http://<external-ip>:9200
   spring.elasticsearch.username=<your-username>
   spring.elasticsearch.password=<your-password>
   ```

   **ElasticsearchConfig.java:**
   ```java
   import org.apache.http.auth.AuthScope;
   import org.apache.http.auth.UsernamePasswordCredentials;
   import org.apache.http.impl.client.BasicCredentialsProvider;
   import org.apache.http.impl.nio.client.HttpAsyncClientBuilder;
   import org.apache.http.HttpHost;
   import org.elasticsearch.client.RestClient;
   import org.elasticsearch.client.RestClientBuilder;
   import org.elasticsearch.client.RestHighLevelClient;
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
       public RestHighLevelClient client() {
           final BasicCredentialsProvider credentialsProvider = new BasicCredentialsProvider();
           credentialsProvider.setCredentials(AuthScope.ANY, new UsernamePasswordCredentials(username, password));

           RestClientBuilder builder = RestClient.builder(HttpHost.create(elasticsearchUri))
                   .setHttpClientConfigCallback(new RestClientBuilder.HttpClientConfigCallback() {
                       @Override
                       public HttpAsyncClientBuilder customizeHttpClient(HttpAsyncClientBuilder httpClientBuilder) {
                           return httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider);
                       }
                   });

           return new RestHighLevelClient(builder);
       }
   }
   ```

   **ElasticsearchService.java:**
   ```java
   import org.elasticsearch.action.get.GetRequest;
   import org.elasticsearch.action.get.GetResponse;
   import org.elasticsearch.action.index.IndexRequest;
   import org.elasticsearch.action.index.IndexResponse;
   import org.elasticsearch.client.RequestOptions;
   import org.elasticsearch.client.RestHighLevelClient;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.stereotype.Service;

   import java.io.IOException;
   import java.util.Map;

   @Service
   public class ElasticsearchService {

       @Autowired
       private RestHighLevelClient client;

       public Map<String, Object> getDocumentById(String index, String id) throws IOException {
           GetRequest getRequest = new GetRequest(index, id);
           GetResponse getResponse = client.get(getRequest, RequestOptions.DEFAULT);
           return getResponse.getSource();
       }

       public String indexDocument(String index, String id, Map<String, Object> document) throws IOException {
           IndexRequest indexRequest = new IndexRequest(index).id(id).source(document);
           IndexResponse indexResponse = client.index(indexRequest, RequestOptions.DEFAULT);
           return indexResponse.getId();
       }
   }
   ```

   **ElasticsearchController.java:**
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
