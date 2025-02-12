# Guia de Autenticação com RHSSO usando Audiences e Certificados no Microservice Spring

## Introdução
Este guia detalha o processo de configuração da autenticação de um microservice Spring com o **RHSSO (Red Hat Single Sign-On)** utilizando **audiences** e **certificados**.

## 1. Obtendo o Certificado no RHSSO
O RHSSO utiliza certificados para assinar os tokens JWT. Para validar a autenticidade dos tokens, é necessário obter o certificado público do servidor RHSSO.

### 1.1. Obtendo o Certificado Manualmente
1. Solicite o certificado ao administrador do RHSSO ou exporte-o manualmente se tiver acesso.
2. O certificado geralmente está no formato **PEM** ou **CRT**. Se necessário, converta-o para PEM com o seguinte comando:
   ```sh
   openssl x509 -inform DER -in certificado.crt -out certificado.pem
   ```
3. Salve o certificado em `src/main/resources/certs/rhsso-public.pem` dentro do seu projeto Spring Boot.

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
          jwk-set-uri: file:src/main/resources/certs/rhsso-public.pem
```

### 2.3. Configurando a Leitura do Certificado Manualmente
Crie um `JwtDecoder` personalizado para carregar o certificado manualmente.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ClassPathResource;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.oauth2.jwt.NimbusJwtDecoder;
import org.springframework.security.oauth2.jwt.JwtValidators;
import java.security.interfaces.RSAPublicKey;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.io.InputStream;

@Configuration
public class JwtConfig {

    @Bean
    public JwtDecoder jwtDecoder() throws Exception {
        InputStream certStream = new ClassPathResource("certs/rhsso-public.pem").getInputStream();
        CertificateFactory certFactory = CertificateFactory.getInstance("X.509");
        X509Certificate certificate = (X509Certificate) certFactory.generateCertificate(certStream);
        RSAPublicKey publicKey = (RSAPublicKey) certificate.getPublicKey();

        return NimbusJwtDecoder.withPublicKey(publicKey).build();
    }
}
```

### 2.4. Configurando Audience Validation no Spring Security
Crie uma classe de configuração para validar audiences no JWT.

```java
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.jwt.OAuth2Error;
import org.springframework.security.oauth2.jwt.OAuth2TokenValidator;
import org.springframework.security.oauth2.jwt.OAuth2TokenValidatorResult;
import org.springframework.stereotype.Component;
import java.util.List;

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
Agora seu microservice Spring está configurado para autenticação e autorização com RHSSO utilizando **audiences** e **certificados**, sem depender do acesso direto ao RHSSO na K8s.

