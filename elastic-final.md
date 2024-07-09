## Configuration `pom.xml`

Assurez-vous d'avoir les dépendances nécessaires :

```xml
<dependencies>
    <!-- Spring Boot Dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- ElasticSearch Dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
    </dependency>

    <!-- Apache Tika for file content extraction -->
    <dependency>
        <groupId>org.apache.tika</groupId>
        <artifactId>tika-core</artifactId>
        <version>2.4.1</version>
    </dependency>
    <dependency>
        <groupId>org.apache.tika</groupId>
        <artifactId>tika-parsers-standard-package</artifactId>
        <version>2.4.1</version>
    </dependency>

    <!-- Testing Dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

### Modèle de Document

La classe de modèle reste la même :

```java
import org.springframework.data.annotation.Id;
import org.springframework.data.elasticsearch.annotations.Document;

@Document(indexName = "documents")
public class DocumentModel {

    @Id
    private String id;
    private String title;
    private String content;

    // Getters and Setters
}
```

### Repository ElasticSearch

Le repository reste le même :

```java
import org.springframework.data.elasticsearch.repository.ElasticsearchRepository;
import java.util.List;

public interface DocumentRepository extends ElasticsearchRepository<DocumentModel, String> {
    List<DocumentModel> findByContentContaining(String content);
}
```

### Service

Ajoutez la logique pour extraire le contenu du fichier à l'aide de Tika :

```java
import org.apache.tika.Tika;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.util.List;
import java.util.UUID;

@Service
public class DocumentService {

    @Autowired
    private DocumentRepository documentRepository;

    private final Tika tika = new Tika();

    public DocumentModel save(MultipartFile file) throws Exception {
        String content = tika.parseToString(file.getInputStream());
        DocumentModel document = new DocumentModel();
        document.setId(UUID.randomUUID().toString());
        document.setTitle(file.getOriginalFilename());
        document.setContent(content);
        return documentRepository.save(document);
    }

    public List<DocumentModel> search(String query) {
        return documentRepository.findByContentContaining(query);
    }
}
```

### Controller

Mettez à jour le contrôleur pour gérer le téléchargement de fichiers :

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.util.List;

@RestController
@RequestMapping("/documents")
public class DocumentController {

    @Autowired
    private DocumentService documentService;

    @PostMapping
    public DocumentModel indexDocument(@RequestParam("file") MultipartFile file) throws Exception {
        return documentService.save(file);
    }

    @GetMapping("/search")
    public List<DocumentModel> searchDocuments(@RequestParam String query) {
        return documentService.search(query);
    }
}
```

### Tests Unitaires

Modifiez les tests unitaires pour vérifier le téléchargement et l'indexation de fichiers :

```java
import static org.mockito.Mockito.*;
import static org.assertj.core.api.Assertions.*;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.http.MediaType;
import org.springframework.mock.web.MockMultipartFile;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders;

import java.util.Collections;

@WebMvcTest(DocumentController.class)
public class DocumentControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private DocumentService documentService;

    @Test
    void shouldIndexDocument() throws Exception {
        MockMultipartFile file = new MockMultipartFile("file", "test.txt", MediaType.TEXT_PLAIN_VALUE, "Test content".getBytes());

        DocumentModel document = new DocumentModel();
        document.setId("1");
        document.setTitle("test.txt");
        document.setContent("Test content");

        when(documentService.save(any(MultipartFile.class))).thenReturn(document);

        mockMvc.perform(MockMvcRequestBuilders.multipart("/documents")
                .file(file))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.id").value("1"))
                .andExpect(jsonPath("$.title").value("test.txt"))
                .andExpect(jsonPath("$.content").value("Test content"));
    }

    @Test
    void shouldSearchDocuments() throws Exception {
        DocumentModel document = new DocumentModel();
        document.setId("1");
        document.setTitle("test.txt");
        document.setContent("Test content");

        when(documentService.search("Test")).thenReturn(Collections.singletonList(document));

        mockMvc.perform(MockMvcRequestBuilders.get("/documents/search")
                .param("query", "Test"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$[0].id").value("1"))
                .andExpect(jsonPath("$[0].title").value("test.txt"))
                .andExpect(jsonPath("$[0].content").value("Test content"));
    }
}
```

### Résumé

1. **Configuration** : Dépendances pour Spring Boot, ElasticSearch et Apache Tika.
2. **Modèle** : Définition de la structure des documents.
3. **Repository** : Interface pour interagir avec ElasticSearch.
4. **Service** : Gestion de la logique métier, extraction de contenu de fichiers avec Tika.
5. **Controller** : Gestion des endpoints pour l'upload et la recherche.
6. **Tests Unitaires** : Vérification du bon fonctionnement des fonctionnalités.
