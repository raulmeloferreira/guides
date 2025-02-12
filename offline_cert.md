# Guia de Autenticação com RHSSO usando Audiences e Certificados no Microservice Spring

## Introdução
Este guia detalha o processo de configuração da autenticação de um microservice Spring com o **RHSSO (Red Hat Single Sign-On)** utilizando **audiences** e **certificados**.

## 1. Obtendo o Certificado no RHSSO
O RHSSO utiliza certificados para assinar os tokens JWT. Para validar a autenticidade dos tokens, é necessário obter o certificado público do servidor RHSSO.

### 1.1. Acessando a Chave Pública do RHSSO
1. Acesse a URL do RHSSO:
   ```sh
   https://<RHSSO_HOST>/auth/realms/<REALM>/protocol/openid-connect/certs
   ```
2. O retorno será um JSON contendo a chave pública (JWKS), por exemplo:
   ```json
   {
       "keys": [
           {
               "kty": "RSA",
               "kid": "xyz123",
               "use": "sig",
               "alg": "RS256",
               "n": "<BASE64_PUBLIC_KEY>",
               "e": "AQAB"
           }
       ]
   }
   ```
3. Copie o valor da chave "n" e converta para formato **PEM** (OpenSSL pode ser usado para isso).
4. Salve o certificado no diretório `src/main/resources/certs` do projeto.

## 2. Configurando o Certificado no Spring Boot

### 2.1. Adicionando Dependências no `pom.xml`
Certifique-se de ter as dependências do Spring Security e Keycloak Adapter:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.keycloak</groupId>
    <artifactId>keycloak-spring-boot-starter</artifactId>
    <version>21.0.1</version>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

### 2.2. Configuração do `application.yml`

```yaml
server:
  port: 8080

spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://<RHSSO_HOST>/auth/realms/<REALM>
          jwk-set-uri: https://<RHSSO_HOST>/auth/realms/<REALM>/protocol/openid-connect/certs
```

### 2.3. Configurando Audience Validation no Spring Security
Crie uma classe de configuração para validar audiences no JWT.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.oauth2.server.resource.authentication.JwtGrantedAuthoritiesConverter;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwtConfigurer -> jwtConfigurer
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            );

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");

        JwtAuthenticationConverter authenticationConverter = new JwtAuthenticationConverter();
        authenticationConverter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        return authenticationConverter;
    }
}
```

### 2.4. Validando o Audience no Token
Crie um filtro para garantir que o JWT tenha o **audience** correto.

```java
@Component
public class AudienceValidator implements OAuth2TokenValidator<Jwt> {

    private final List<String> allowedAudiences = List.of("my-service-audience");

    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        List<String> audiences = jwt.getAudience();
        if (audiences == null || audiences.stream().noneMatch(allowedAudiences::contains)) {
            return OAuth2TokenValidatorResult.failure(new OAuth2Error("invalid_token", "Invalid audience", null));
        }
        return OAuth2TokenValidatorResult.success();
    }
}
```

## 3. Testando a Autenticação

### 3.1. Gerando um Token JWT de Teste

```sh
curl -X POST "https://<RHSSO_HOST>/auth/realms/<REALM>/protocol/openid-connect/token" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "grant_type=client_credentials" \
     -d "client_id=<CLIENT_ID>" \
     -d "client_secret=<CLIENT_SECRET>"
```

### 3.2. Testando a API Protegida

```sh
curl -X GET "http://localhost:8080/api/protected" \
     -H "Authorization: Bearer <ACCESS_TOKEN>"
```

Se tudo estiver correto, a API responderá com sucesso.

## Conclusão
Agora seu microservice Spring está configurado para autenticação e autorização com RHSSO utilizando **audiences** e **certificados**. Isso garante que apenas tokens válidos e destinados ao seu serviço sejam aceitos.

