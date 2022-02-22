Spring boot GraphQL Basic Implementation

![](https://github.com/sercancelenk/spring-boot-graphql-demo/blob/main/graphql_demo.gif)



- Create Provider
```
@Component
public class GraphQLProvider {

    private GraphQL graphQL;

    @Bean
    public GraphQL graphQL() {
        return graphQL;
    }

    @PostConstruct
    public void init() throws IOException {
        URL url = Resources.getResource("schema.graphqls");
        String sdl = Resources.toString(url, Charsets.UTF_8);
        GraphQLSchema graphQLSchema = buildSchema(sdl);
        this.graphQL = GraphQL.newGraphQL(graphQLSchema).build();
    }

    @Autowired
    GraphQLDataFetchers graphQLDataFetchers;

    private GraphQLSchema buildSchema(String sdl) {
        TypeDefinitionRegistry typeRegistry = new SchemaParser().parse(sdl);
        RuntimeWiring runtimeWiring = buildWiring();
        SchemaGenerator schemaGenerator = new SchemaGenerator();
        return schemaGenerator.makeExecutableSchema(typeRegistry, runtimeWiring);
    }

    private RuntimeWiring buildWiring() {
        return RuntimeWiring.newRuntimeWiring()
                .type(newTypeWiring("Query")
                        .dataFetcher("bookById", graphQLDataFetchers.getBookByIdDataFetcher())
                        .dataFetcher("all", graphQLDataFetchers.getAllBooks())
                )
                .type(newTypeWiring("Book")
                        .dataFetcher("author", graphQLDataFetchers.getAuthorDataFetcher())
                        .dataFetcher("pageCount", graphQLDataFetchers.getPageCountDataFetcher())
                )
                .build();
    }
}
```

- Create Data Fetchers
```
@Component
public class GraphQLDataFetchers {

    private static List<Map<String, String>> books = Arrays.asList(
            ImmutableMap.of("id", "book-1",
                    "name", "Harry Potter and the Philosopher's Stone",
                    "pageCount", "223",
                    "authorId", "author-1"),
            ImmutableMap.of("id", "book-2",
                    "name", "Moby Dick",
                    "pageCount", "635",
                    "authorId", "author-2"),
            ImmutableMap.of("id", "book-3",
                    "name", "Interview with the vampire",
                    "pageCount", "371",
                    "authorId", "author-3")
    );

    private static List<Map<String, String>> authors = Arrays.asList(
            ImmutableMap.of("id", "author-1",
                    "firstName", "Joanne",
                    "lastName", "Rowling"),
            ImmutableMap.of("id", "author-2",
                    "firstName", "Herman",
                    "lastName", "Melville"),
            ImmutableMap.of("id", "author-3",
                    "firstName", "Anne",
                    "lastName", "Rice")
    );

    public DataFetcher getBookByIdDataFetcher() {
        return dataFetchingEnvironment -> {
            String bookId = dataFetchingEnvironment.getArgument("id");
            return books
                    .stream()
                    .filter(book -> book.get("id").equals(bookId))
                    .findFirst()
                    .orElse(null);
        };
    }

    public DataFetcher getAuthorDataFetcher() {
        return dataFetchingEnvironment -> {
            Map<String, String> book = dataFetchingEnvironment.getSource();
            String authorId = book.get("authorId");
            return authors
                    .stream()
                    .filter(author -> author.get("id").equals(authorId))
                    .findFirst()
                    .orElse(null);
        };
    }

    public DataFetcher getPageCountDataFetcher() {
        return dataFetchingEnvironment -> {
            Map<String, String> book = dataFetchingEnvironment.getSource();
            return book.get("totalPages");
        };
    }

    public DataFetcher getAllBooks() {
        return dataFetchingEnvironment -> books;
    }
}
```
- Add Dependencies
```
    implementation 'com.graphql-java:graphql-java:11.0' // NEW
    implementation 'com.graphql-java:graphql-java-spring-boot-starter-webmvc:1.0' // NEW
    implementation 'com.google.guava:guava:26.0-jre' // NEW
```

- Api Usage
```
Request Body
query {
  bookById(id: "book-1"){
    id
  }
  all{
    id
    name
    author{
      id
      firstName
      lastName
    }
  }
}

Response
{
  "data": {
    "bookById": {
      "id": "book-1"
    },
    "all": [
      {
        "id": "book-1",
        "name": "Harry Potter and the Philosopher's Stone",
        "author": {
          "id": "author-1",
          "firstName": "Joanne",
          "lastName": "Rowling"
        }
      },
      {
        "id": "book-2",
        "name": "Moby Dick",
        "author": {
          "id": "author-2",
          "firstName": "Herman",
          "lastName": "Melville"
        }
      },
      {
        "id": "book-3",
        "name": "Interview with the vampire",
        "author": {
          "id": "author-3",
          "firstName": "Anne",
          "lastName": "Rice"
        }
      }
    ]
  }
}
```
