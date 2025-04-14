package com.amazonaws.appflow.custom.connector.http;

import com.amazonaws.appflow.custom.connector.model.ConnectorContext;
import com.amazonaws.appflow.custom.connector.model.credentials.AuthCredentials;
import com.amazonaws.appflow.custom.connector.model.credentials.BasicAuthCredentials;
import com.amazonaws.appflow.custom.connector.model.metadata.EntityDefinition;
import com.amazonaws.appflow.custom.connector.model.metadata.EntityDefinitionProvider;
import com.amazonaws.appflow.custom.connector.model.metadata.FieldDefinition;
import com.amazonaws.appflow.custom.connector.model.query.QueryFilter;
import com.amazonaws.appflow.custom.connector.model.retreive.RetrieveDataRequest;
import com.amazonaws.appflow.custom.connector.model.retreive.RetrieveDataResponse;
import com.amazonaws.appflow.custom.connector.model.write.WriteDataRequest;
import com.amazonaws.appflow.custom.connector.model.write.WriteDataResponse;
import com.amazonaws.appflow.custom.connector.queryfilter.QueryFilterBuilder;
import com.amazonaws.appflow.custom.connector.handlers.RecordHandler;
import com.amazonaws.appflow.custom.connector.model.ConnectorRequestContext;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import okhttp3.MediaType;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;

import java.io.IOException;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

@Slf4j
public class HttpConnector implements RecordHandler {
    private static final MediaType JSON = MediaType.get("application/json; charset=utf-8");
    private final OkHttpClient client;
    private final ObjectMapper objectMapper;

    public HttpConnector() {
        this.client = new OkHttpClient();
        this.objectMapper = new ObjectMapper();
    }

    @Override
    public RetrieveDataResponse retrieveData(ConnectorContext context, RetrieveDataRequest request) throws Exception {
        String url = context.getConnectorRuntimeSettings().get("apiUrl");
        String requestBody = context.getConnectorRuntimeSettings().get("requestBody");

        Request httpRequest = new Request.Builder()
                .url(url)
                .post(RequestBody.create(requestBody, JSON))
                .build();

        try (Response response = client.newCall(httpRequest).execute()) {
            if (!response.isSuccessful()) {
                throw new IOException("Unexpected response code: " + response);
            }

            String responseBody = response.body().string();
            JsonNode jsonResponse = objectMapper.readTree(responseBody);

            List<Map<String, Object>> records = new ArrayList<>();
            // Convert JSON response to records - this is a simple example
            // You'll need to adjust this based on your API response structure
            if (jsonResponse.isArray()) {
                jsonResponse.forEach(node -> {
                    Map<String, Object> record = new HashMap<>();
                    node.fields().forEachRemaining(entry -> 
                        record.put(entry.getKey(), entry.getValue().asText()));
                    records.add(record);
                });
            }

            return RetrieveDataResponse.builder()
                    .records(records)
                    .isSuccess(true)
                    .build();
        }
    }

    @Override
    public WriteDataResponse writeData(ConnectorContext context, WriteDataRequest request) {
        // Implement if write operations are needed
        throw new UnsupportedOperationException("Write operations are not supported");
    }
} 



////


package com.amazonaws.appflow.custom.connector.http;

import com.amazonaws.appflow.custom.connector.model.ConnectorContext;
import com.amazonaws.appflow.custom.connector.model.metadata.EntityDefinition;
import com.amazonaws.appflow.custom.connector.model.metadata.FieldDefinition;
import com.amazonaws.appflow.custom.connector.model.settings.ConnectorRuntimeSetting;
import com.amazonaws.appflow.custom.connector.model.settings.ConnectorRuntimeSettingDataType;
import com.amazonaws.appflow.custom.connector.handlers.ConfigurationHandler;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;

public class HttpConnectorConfiguration implements ConfigurationHandler {

    @Override
    public List<ConnectorRuntimeSetting> getConnectorRuntimeSettings(ConnectorContext context) {
        return Arrays.asList(
            ConnectorRuntimeSetting.builder()
                .key("apiUrl")
                .required(true)
                .label("API URL")
                .description("The URL endpoint for the HTTP POST request")
                .dataType(ConnectorRuntimeSettingDataType.String)
                .build(),
            ConnectorRuntimeSetting.builder()
                .key("requestBody")
                .required(true)
                .label("Request Body")
                .description("The JSON body to be sent in the POST request")
                .dataType(ConnectorRuntimeSettingDataType.String)
                .build()
        );
    }

    @Override
    public List<EntityDefinition> getEntities(ConnectorContext context) {
        EntityDefinition entityDefinition = EntityDefinition.builder()
                .entityIdentifier("HttpResponse")
                .label("HTTP Response")
                .hasNestedEntities(false)
                .description("Represents the response from the HTTP POST request")
                .fields(getFields())
                .build();

        return Arrays.asList(entityDefinition);
    }

    private List<FieldDefinition> getFields() {
        // Define the fields based on your expected response structure
        // This is a sample - modify according to your needs
        List<FieldDefinition> fields = new ArrayList<>();
        
        fields.add(FieldDefinition.builder()
                .fieldName("id")
                .dataType("String")
                .label("ID")
                .description("Unique identifier")
                .isPrimaryKey(true)
                .build());

        fields.add(FieldDefinition.builder()
                .fieldName("data")
                .dataType("String")
                .label("Data")
                .description("Response data")
                .build());

        return fields;
    }
} 

//////
package com.amazonaws.appflow.custom.connector.http;

import com.amazonaws.appflow.custom.connector.handlers.ConfigurationHandler;
import com.amazonaws.appflow.custom.connector.handlers.RecordHandler;
import com.amazonaws.appflow.custom.connector.model.ConnectorContext;
import com.amazonaws.appflow.custom.connector.model.ConnectorType;
import com.amazonaws.appflow.custom.connector.model.credentials.AuthenticationType;
import com.amazonaws.appflow.custom.connector.model.settings.ValidateCredentialsResponse;
import com.amazonaws.appflow.custom.connector.providers.ConnectorProvider;

public class HttpConnectorProvider implements ConnectorProvider {
    private final HttpConnector connector;
    private final HttpConnectorConfiguration configuration;

    public HttpConnectorProvider() {
        this.connector = new HttpConnector();
        this.configuration = new HttpConnectorConfiguration();
    }

    @Override
    public String getConnectorName() {
        return "HTTP-Custom-Connector";
    }

    @Override
    public ConnectorType getConnectorType() {
        return ConnectorType.CustomConnector;
    }

    @Override
    public AuthenticationType getAuthenticationType() {
        return AuthenticationType.None;
    }

    @Override
    public ValidateCredentialsResponse validateCredentials(ConnectorContext context) {
        // No credentials validation needed for this simple example
        return ValidateCredentialsResponse.builder().isSuccess(true).build();
    }

    @Override
    public RecordHandler getRecordHandler() {
        return connector;
    }

    @Override
    public ConfigurationHandler getConfigurationHandler() {
        return configuration;
    }
} 


///pom

<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <groupId>com.amazonaws.appflow</groupId>
        <artifactId>custom-connector-sdk-parent</artifactId>
        <version>1.0.0</version>
        <relativePath>../pom.xml</relativePath>
    </parent>

    <artifactId>custom-connector-http</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging>

    <dependencies>
        <dependency>
            <groupId>com.amazonaws.appflow</groupId>
            <artifactId>custom-connector-sdk</artifactId>
            <version>1.0.0</version>
        </dependency>
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>okhttp</artifactId>
            <version>4.9.3</version>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.13.4.2</version>
        </dependency>
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <version>1.18.24</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.4</version>
                <configuration>
                    <createDependencyReducedPom>false</createDependencyReducedPom>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project> 
