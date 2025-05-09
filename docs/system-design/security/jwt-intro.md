---
title: Detailed Explanation of JWT Basic Concepts
category: System Design
tag:
  - Security
---

<!-- @include: @article-header.snippet.md -->

## What is JWT?

JWT (JSON Web Token) is currently the most popular solution for cross-domain authentication and is a token-based authentication and authorization mechanism. As can be seen from the full name of JWT, JWT itself is also a token, a token in a standardized JSON structure.

JWT itself contains all the information needed for authentication, so our server does not need to store session information. This obviously increases the system's availability and scalability, greatly reducing the pressure on the server.

It can be seen that **JWT aligns with the "Stateless" principle when designing RESTful APIs**.

Additionally, using JWT for authentication can effectively avoid CSRF attacks, as JWT is generally stored in localStorage, and the authentication process does not involve cookies.

I have detailed the advantages and disadvantages of using JWT for authentication in my article [Analysis of JWT Advantages and Disadvantages](./advantages-and-disadvantages-of-jwt.md).

Below is a more formal definition of JWT from [RFC 7519](https://tools.ietf.org/html/rfc7519).

> JSON Web Token (JWT) is a compact, URL-safe means of representing claims to be transferred between two parties. The claims in a JWT are encoded as a JSON object that is used as the payload of a JSON Web Signature (JWS) structure or as the plaintext of a JSON Web Encryption (JWE) structure, enabling the claims to be digitally signed or integrity protected with a Message Authentication Code (MAC) and/or encrypted. ——[JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519)

## What Does JWT Consist Of?

![JWT Composition](https://oss.javaguide.cn/javaguide/system-design/jwt/jwt-composition.png)

JWT is essentially a set of strings separated by (`.`) into three Base64-encoded parts:

- **Header**: Describes the metadata of JWT, defining the algorithm used to generate the signature and the type of the `Token`. The Header, after being Base64Url encoded, becomes the first part of the JWT.
- **Payload**: Used to store the actual data that needs to be transmitted, containing claims (Claims), such as `sub` (subject) and `jti` (JWT ID). The Payload, after being Base64Url encoded, becomes the second part of the JWT.
- **Signature**: Generated by the server using the Payload, Header, and a secret key, according to the signature algorithm specified in the Header (default is HMAC SHA256). The generated signature becomes the third part of the JWT.

JWT typically looks like this: `xxxxx.yyyyy.zzzzz`.

Example:

```plain
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

You can decode the JWT on the website [jwt.io](https://jwt.io/), and you will get the Header, Payload, and Signature parts.

Both Header and Payload are in JSON format, and the Signature is obtained by performing a specific calculation and encryption algorithm on the Payload, Header, and Secret.

![](https://oss.javaguide.cn/javaguide/system-design/jwt/jwt.io.png)

### Header

The Header typically consists of two parts:

- `typ` (Type): The type of token, which is JWT.
- `alg` (Algorithm): The signature algorithm, such as HS256.

Example:

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

The JSON-formatted Header is encoded into Base64, making it the first part of the JWT.

### Payload

The Payload is also in JSON format and contains claims related to the JWT.

Claims are divided into three types:

- **Registered Claims**: Some predefined claims that are recommended but not mandatory.
- **Public Claims**: Claims that the JWT issuer can define, but to avoid conflicts, they should be defined in the [IANA JSON Web Token Registry](https://www.iana.org/assignments/jwt/jwt.xhtml).
- **Private Claims**: Claims that the JWT issuer customizes for specific project needs, more relevant to actual project scenarios.

Here are some common registered claims:

- `iss` (issuer): The issuer of the JWT.
- `iat` (issued at time): The time the JWT was issued.
- `sub` (subject): The subject of the JWT.
- `aud` (audience): The intended audience of the JWT.
- `exp` (expiration time): The expiration time of the JWT.
- `nbf` (not before time): The time before which the JWT must not be accepted for processing.
- `jti` (JWT ID): A unique identifier for the JWT.

Example:

```json
{
  "uid": "ff1212f5-d8d1-4496-bf41-d2dda73de19a",
  "sub": "1234567890",
  "name": "John Doe",
  "exp": 15323232,
  "iat": 1516239022,
  "scope": ["admin", "user"]
}
```

The Payload part is not encrypted by default, **do not store sensitive information in the Payload!!!**

The JSON-formatted Payload is encoded into Base64, becoming the second part of the JWT.

### Signature

The Signature part is a signature of the first two parts, serving to prevent the JWT (mainly the payload) from being tampered with.

The generation of this signature requires:

- Header + Payload.
- A secret stored on the server (must not be leaked).
- The signature algorithm.

The calculation formula for the signature is as follows:

```plain
HMACSHA256(
  base64UrlEncode(header) + "." +
  base64UrlEncode(payload),
  secret)
```

After calculating the signature, concatenate the Header, Payload, and Signature into a string, with each part separated by a dot (`.`); this string is the JWT.

## How to Authenticate Based on JWT?

In applications that authenticate based on JWT, the server creates a JWT using the Payload, Header, and Secret, and sends the JWT to the client. After receiving the JWT, the client will store it in a cookie or localStorage, and all future requests from the client will carry this token.

![ JWT Authentication Process](https://oss.javaguide.cn/github/javaguide/system-design/jwt/jwt-authentication%20process.png)

The simplified steps are as follows:

1. The user sends their username, password, and verification code to log into the system.
1. If the username, password, and verification code are verified correctly, the server returns the signed token, which is the JWT.
1. The user includes this JWT in the Header of every future request to the backend.
1. The server checks the JWT and retrieves the user's relevant information from it.

Two suggestions:

1. It is recommended to store the JWT in localStorage, as storing it in cookies poses a CSRF risk.
1. A common practice for sending the JWT to the server is to place it in the HTTP Header's `Authorization` field (`Authorization: Bearer Token`).

**[spring-security-jwt-guide](https://github.com/Snailclimb/spring-security-jwt-guide)** is a simple case study based on JWT for authentication, which you may find interesting.

## How to Prevent JWT Tampering?

With the signature in place, even if the JWT is leaked or intercepted, hackers cannot tamper with the Signature, Header, and Payload simultaneously.

Why is that? Because when the server receives the JWT, it parses the Header, Payload, and Signature contained within. The server then regenerates a Signature based on the Header, Payload, and secret. If the newly generated Signature matches the one in the JWT, it indicates that the Header and Payload have not been modified.

However, if the server's secret is also leaked, hackers could tamper with the Signature, Header, and Payload as well. They could directly modify the Header and Payload, then regenerate a Signature.

**Keep the secret safe and do not leak it. The security of JWT relies primarily on the signature, and the security of the signature is based on the secret.**

## How to Enhance JWT Security?

1. Use encryption algorithms with a high security factor.
1. Use mature open-source libraries; there is no need to reinvent the wheel.
1. Store JWT in localStorage rather than in cookies to avoid CSRF risks.
1. Never store sensitive information in the Payload.
1. Keep the secret safe and never leak it. The security of JWT relies on the signature, and the security of the signature relies on the secret.
1. The Payload should include `exp` (the expiration time of the JWT); having a permanently valid JWT is unreasonable. Moreover, the expiration time of the JWT should not be too long.
1. ……

<!-- @include: @article-footer.snippet.md -->
