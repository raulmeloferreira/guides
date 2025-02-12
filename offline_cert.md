# Guia de Autenticação com RHSSO usando Audiences e Certificados no Microservice Spring

## Introdução
Este guia detalha o processo de configuração da autenticação de um microservice Spring com o **RHSSO (Red Hat Single Sign-On)** utilizando **audiences** e **certificados**.

## 1. Configurando o Audience no RHSSO
Para que o RHSSO gere tokens com a audiência correta, siga os passos abaixo:

### 1.1. Criando um Client com Audience no RHSSO
1. Acesse o **Keycloak Admin Console** em:
   ```
   https://<RHSSO_HOST>/auth/admin/<REALM>/console
   ```
2. Vá para **Clients** e clique em **Create**.
3. Configure os seguintes campos:
   - **Client ID**: `my-service`
   - **Client Protocol**: `openid-connect`
   - **Access Type**: `confidential`
4. Clique em **Save**.

### 1.2. Configurando Audience
1. No client criado, vá para a aba **Settings** e ative as opções:
   - **Service Accounts Enabled**: `ON`
   - **Authorization Enabled**: `ON`
2. Vá para a aba **Advanced Settings** e configure:
   - **Valid Redirect URIs**: `http://localhost:8080/*`
   - **Web Origins**: `+`
3. Acesse a aba **Mappers** e clique em **Create**:
   - **Name**: `audience`
   - **Mapper Type**: `Audience`
   - **Included Client Audience**: `my-service-audience`
   - **Add to ID Token**: `ON`
   - **Add to Access Token**: `ON`
4. Clique em **Save**.

## 2. Obtendo o Certificado no RHSSO
O RHSSO utiliza certificados para assinar os tokens JWT. Para validar a autenticidade dos tokens, é necessário obter o certificado público do servidor RHSSO.

### 2.1. Obtendo o Certificado Manualmente
1. Solicite o certificado ao administrador do RHSSO ou exporte-o manualmente se tiver acesso.
2. O certificado geralmente está no formato **PEM** ou **CRT**. Se necessário, converta-o para PEM com o seguinte comando:
   ```sh
   openssl x509 -inform DER -in certificado.crt -out certificado.pem
   ```
3. Salve o certificado em `src/main/resources/certs/rhsso-public.pem` dentro do seu projeto Spring Boot.

## 3. Configurando o Certificado no Spring Boot

### 3.1. Adicionando Dependências no `pom.xml`
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

### 3.2. Configuração do `application.yml`

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

## 4. Testando a Autenticação

### 4.1. Gerando um Token JWT de Teste

```sh
curl -X POST "https://<RHSSO_HOST>/auth/realms/<REALM>/protocol/openid-connect/token" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "grant_type=client_credentials" \
     -d "client_id=my-service" \
     -d "client_secret=<CLIENT_SECRET>"
```

### 4.2. Testando a API Protegida

```sh
curl -X GET "http://localhost:8080/api/protected" \
     -H "Authorization: Bearer <ACCESS_TOKEN>"
```

Se tudo estiver correto, a API responderá com sucesso.

## Conclusão
Agora seu microservice Spring está configurado para autenticação e autorização com RHSSO utilizando **audiences** e **certificados**, sem depender do acesso direto ao RHSSO na K8s.

