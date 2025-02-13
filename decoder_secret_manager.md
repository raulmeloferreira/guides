# 🔐 Autenticação JWT com Spring Security e Secret Manager

Este guia ensina como configurar o **Spring Security** para buscar a chave pública do JWT em um **Secret Manager** (AWS Secrets Manager, HashiCorp Vault, Azure Key Vault, etc.), em vez de depender diretamente de um OpenID Provider (como Keycloak, RH-SSO).

---

## 📌 Visão Geral

A estratégia adotada será:
1. **Recuperar a chave pública do JWT** armazenada no Secret Manager.
2. **Criar um `JwtDecoder` personalizado** para validar tokens usando essa chave.
3. **Configurar o Spring Security** para usar o `JwtDecoder` com validação JWT.

---

## 🛠 Configuração no Spring Boot

### 1️⃣ Criar um `JwtDecoder` Personalizado

Crie um **serviço** que buscará a chave pública no Secret Manager.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.security.oauth2.jwt.*;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueResponse;

import java.security.interfaces.RSAPublicKey;
import java.util.Base64;
import java.security.spec.X509EncodedKeySpec;
import java.security.KeyFactory;

@Service
public class CustomJwtDecoderService {
    
    private final SecretsManagerClient secretsManagerClient;

    public CustomJwtDecoderService(SecretsManagerClient secretsManagerClient) {
        this.secretsManagerClient = secretsManagerClient;
    }

    private String getPublicKeyFromSecretsManager(String secretName) {
        GetSecretValueRequest request = GetSecretValueRequest.builder()
                .secretId(secretName)
                .build();
        GetSecretValueResponse response = secretsManagerClient.getSecretValue(request);
        return response.secretString(); // Assume que a chave pública está armazenada como uma string JSON ou PEM
    }

    private RSAPublicKey convertStringToPublicKey(String publicKeyPem) throws Exception {
        publicKeyPem = publicKeyPem.replace("-----BEGIN PUBLIC KEY-----", "")
                                   .replace("-----END PUBLIC KEY-----", "")
                                   .replaceAll("\s", ""); // Remove caracteres inválidos
        
        byte[] decoded = Base64.getDecoder().decode(publicKeyPem);
        X509EncodedKeySpec spec = new X509EncodedKeySpec(decoded);
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        return (RSAPublicKey) keyFactory.generatePublic(spec);
    }

    @Bean
    public JwtDecoder jwtDecoder() throws Exception {
        String secretName = "jwt-public-key"; // Nome do secret no Secrets Manager
        String publicKeyPem = getPublicKeyFromSecretsManager(secretName);
        RSAPublicKey publicKey = convertStringToPublicKey(publicKeyPem);

        return NimbusJwtDecoder.withPublicKey(publicKey).build();
    }
}
```

📌 **Explicação**:
- O **`getPublicKeyFromSecretsManager()`** recupera a chave pública do Secret Manager.
- O **`convertStringToPublicKey()`** converte a chave PEM para um objeto `RSAPublicKey`.
- O método **`jwtDecoder()`** configura um `JwtDecoder` com a chave pública.

---

### 2️⃣ Configurar o `SecurityFilterChain`

Agora, configuramos o Spring Security para usar esse `JwtDecoder`:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.oauth2.jwt.JwtDecoder;

@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http, JwtDecoder jwtDecoder) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            .requestMatchers("/api/admin/**").hasAuthority("ROLE_ADMIN")
            .requestMatchers("/api/user/**").hasAuthority("ROLE_USER")
            .anyRequest().authenticated()
        )
        .oauth2ResourceServer(oauth2 -> oauth2.jwt(jwt -> jwt.decoder(jwtDecoder)));

    return http.build();
}
```

🔹 **Explicação**:
- **Protege os endpoints** com roles específicas (`ROLE_ADMIN`, `ROLE_USER`).
- **Usa o `jwtDecoder`** que buscamos do Secret Manager.

---

### 3️⃣ Dependências Necessárias

Adicione as seguintes dependências no **`pom.xml`**:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>

<dependency>
    <groupId>software.amazon.awssdk</groupId>
    <artifactId>secretsmanager</artifactId>
</dependency>
```

Caso esteja usando outro Secret Manager (**HashiCorp Vault, Azure Key Vault**), adapte a função `getPublicKeyFromSecretsManager()` para buscar o segredo correto.

---

## 🎯 Conclusão

- Em vez de buscar a chave pública de um OpenID Provider, agora o **Spring Security usa a chave pública armazenada em um Secret Manager**.
- O `JwtDecoder` personalizado baixa a chave pública e **valida os tokens JWT dinamicamente**.
- Esse método **melhora a segurança**, pois evita dependência direta de serviços externos.

💡 **Agora você tem uma implementação segura e centralizada para autenticação JWT no Spring Boot!** 🚀
