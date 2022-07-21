# HTTP Message Signatures

HTTP Message Signatures provide a mechanism for providing end-to-end integrity
and authenticity for components of an HTTP message.

This library provides high-level Java interface for creating and verifying the
signatures as defined in
[HTTP Message Signatures specification](https://www.ietf.org/archive/id/draft-ietf-httpbis-message-signatures-10.html)
(draft 10). As by-products, it implements
[Digest Fields](https://www.ietf.org/archive/id/draft-ietf-httpbis-digest-headers-10.html)
(draft 10) and
[Structured Field Values for HTTP](https://www.rfc-editor.org/rfc/rfc8941).

It requires Java 11 or newer and does not have compile dependencies.

Javadoc is available at https://visma-autopay.github.io/http-signatures/

## Introduction

Consider the following HTTP request
```http request
POST https://example.com/foo
Content-Type: application/json
Content-Digest: sha-256=:Zsg9Nyzj13UPzkyaQlnA7wbgTfBaZmH02OVyiRjpydE=:
Signature-Input: my-signature=("@method" "@path" "@authority" "content-type" "content-digest");created=1658319872;nonce="bcf52bbd67af4d4b95e806d2c2c63481";keyid="test-key-ed25519"
Signature: my-signature=:6R8T8jBjqZfYtshgTaYVahGmXIRmr9C3zaLIEYLLtQKrMiR/W4LCYqHX1eUaEPXBVU12VL+nk3knejHqGnqiDQ==:

{"id": 5, "name": "Item"}
```

This library allows to compute and verify `Content-Digest`, `Signature-Input`
and `Sigature` headers.

- `Content-Digest` - A digest (hash value) of request body, defined by _Digest
                     Fields_
- `Signature-Input` - Defines the signature: which parts of the request are 
                      included, what key is used
- `Signature` - Concatenated request parts defined in the input are signed by
                using the referenced key. Both input and signature are defined
                by _HTTP Message Signatures_. 
- Syntax - Those headers are formatted by using syntax of
           _Structured Field Values for HTTP_


## Signatures

### Simple example

#### Creating signature

Before creating signature you need to decide on, among others,
1. What key and signature algorithm to use, like RSA or EdDSA 
2. Which HTTP headers to include
3. Which other parts of HTTP message to include, like method or query param
4. Which signature parameters to include, like creation time or nonce

You can start building the signature by specifying which components (headers,
other parts of the request or response) to include.
```java
var signatureComponents = SignatureComponents.builder()
        .method().path().authority()
        .headers("Content-Type", "Content-Digest")
        .build();
```

Then, signature parameters should be defined
```java
var signatureParameters = SignatureParameters.builder()
        .createdNow()
        .randomNonce()
        .keyId("test-key-ed25519")
        .algorithm(SignatureAlgorithm.ED_25519)
        .build();
```

After that, the actual values from signed request or response should be provided
```java
var requestContext = SignatureContext.builder()
        .method("POST")
        .targetUri("https://example.com/foo")
        .headers(httpHeaders)
        .build();
```

Having those, signature specification can be built
```java
var signatureSpec = SignatureSpec.builder()
        .signatureLabel("my-signature")
        .privateKey(privateKey)
        .context(requestContext)
        .parameters(signatureParameters)
        .components(signatureComponents)
        .build();
```

And signature computed and applied
```java
try {
    var signature = signatureSpec.sign();
    httpHeaders.add(SignatureHeaders.SIGNATURE_INPUT, signature.getSignatureInput());
    httpHeaders.add(SignatureHeaders.SIGNATURE, signature.getSignature());
} catch (SignatureException e) {
    log.error("Problem when creating signature. spec={}", signatureSpec, e);
}
```

#### Verifying signature

Similar steps are needed when verifying signature.

First, you can define required components and parameters. It's an optional step,
needed only if some of them are actually required.

```java
var requiredComponents = SignatureComponents.builder()
        .method().path().authority()
        .headers("Content-Digest")
        .build();

var requiredParameters = List.of(SignatureParameterType.KEY_ID, SignatureParameterType.CREATED, SignatureParameterType.NONCE);
var maximumAgeSeconds = 60;
```

Then, values from verified request or response are needed
```java
var signatureContext = SignatureContext.builder()
        .method(containerRequestContext.getMethod())
        .targetUri(containerRequestContext.getUriInfo().getRequestUri())
        .headers(containerRequestContext.getHeaders())
        .build();
```

You need a method which for given key ID returns corresponding public key
```java
private PublicKeyInfo getPublicKey(String keyId) {
    PublicKey publicKey = publicKeyRepository.getPublicKey(keyId);

    return PublicKeyInfo.builder()
            .algorithm(SignatureAlgorithm.ED_25519)
            .publicKey(publicKey)
            .build();
}
```

Now, verification specification can be built
```java
var verificationSpec = VerificationSpec.builder()
        .signatureLabel("my-signature")
        .requiredComponents(requiredComponents)
        .requiredParameters(requiredParameters)
        .maximumAge(maximumAgeSeconds)
        .context(signatureContext)
        .publicKeyGetter(this::getPublicKey)
        .build();
```

And signature can be verified
```java
try {
    verificationSpec.verify();
} catch (SignatureException e) {
    log.warn("Invalid signature. spec={}", verificationSpec, e);
}
```

### Full example

#### Creating signature

##### Signature components
For details, see [SignatureComponents.Builder](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/signature/SignatureComponents.Builder.html)
```java
var signatureComponents = SignatureComponents.builder()
        // Components derived from request
        .method().path().authority().query().queryParam("id")
        // Components derived from response
        .status()
        // When signing response, components from related request can be added
        .relatedRequestMethod().relatedRequestPath()

        // Header names can be provided one-by-one
        .header("Content-Type")
        .header("Content-Length")
        // Or as vararg
        .headers("Content-Type", "Content-Digest")
        // Or as a collection
        .headers(List.of("Content-Type", "Content-Digest"))

        // Structured fields can be used in their canonicalized form
        .canonicalizedHeader("My-Structured-Header")
        // Individual items of structured dictionary
        .dictionaryMember("My-Dictionary", "key-1")

        // Single headers from related request, when signing response
        .relatedRequestHeader("Content-Type")
        .build();
```

##### Signature parameters
For details, see [SignatureParameters.Builder](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/signature/SignatureParameters.Builder.html)
```java
var signatureParameters = SignatureParameters.builder()
        // Created parameter populated to now()
        .createdNow()
        // or to given Instant
        .created(Instant.now())
        // or to UNIX timestamp
        .created(Instant.now().getEpochSecond())

        // Expires after 30 from created
        .expiresAfter(30)
        // or at given Instant
        .expires(Instant.now().plusSeconds(30))
        // or at given UNIX timestamp
        .expires(Instant.now().getEpochSecond() + 30)

        // Generate random nonce
        .randomNonce()
        // or use provided one
        .nonce(UUID.randomUUID().toString())

        // Use given algorithm when signing
        .algorithm(SignatureAlgorithm.ED_25519)
        // Use it and expose in alg parameter
        .visibleAlgorithm(SignatureAlgorithm.ED_25519)

        // Key ID
        .keyId("test-key-ed25519")
        .build();
```

##### Signature context
For details, see [SignatureContext.Builder](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/signature/SignatureContext.Builder.html)
```java
var requestContext = SignatureContext.builder()
        .method("POST")
        // Can be provided as String or URI
        .targetUri("https://example.com/foo")
        // Status for response signature
        .status(200)

        // Header values can be provided one-by-one
        .header("Content-Type", "application/json")
        .header("Content-Length", "25")
        // Or as a map
        .headers(Map.of("Header-One", "valueOne", "Header-Two", "valueTwo"))
        // Or as a "multimap" or "MultivaluedMap", often used in frameworks
        .headers(Map.of("Header-One", List.of("valueOne", "valueTwo")))

        // Context of related request if used in response signature
        .relatedRequest(relatedRequestContext)
        .build();   
```

##### Signature specification
For details, see [SignatureSpec.Builder](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/signature/SignatureSpec.Builder.html)
```java
var signatureSpec = SignatureSpec.builder()
        .signatureLabel("my-signature")
        // Can be given as PrivateKey object
        // or as PKCS#8-base-64-encoded String, taken from a PEM file
        // or PKCS#8-encoded byte array
        // HMAC keys should be provided as byte[] or base-64-encoded String
        .privateKey(privateKey)
        .context(requestContext)
        .parameters(signatureParameters)
        // Components defined here must be present in the context
        .components(signatureComponents)
        // Components defined here are added to the signature
        // only if related values are present in the context
        .usedIfPresentComponents(optionalComponents)
        .build();
```

##### Signing
For details, see [SignatureSpec.sign()](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/signature/SignatureSpec.html#sign())
```java
try {
    var signature = signatureSpec.sign();
    httpHeaders.add(SignatureHeaders.SIGNATURE_INPUT,signature.getSignatureInput());
    httpHeaders.add(SignatureHeaders.SIGNATURE,signature.getSignature());
    // Signature base can be used for debugging
    log.info("Signature base: {}",signature.getSignatureBase());
} catch (SignatureException e) {
    log.error("Problem when creating signature. spec={}", signatureSpec, e);
}
```

#### Verifying signature

##### Signature components
Required components must be present in verified signature, e.g. a header must
be included in the `Signature-Input` and it's value must be present in the
context.
```java
var requiredComponents = SignatureComponents.builder()
        // See Signature components above
        .build();
```

Required if present components must be present in verified signature only if
they are present in the context, e.g. if a given header is set then it must
be included in the signature, and it's OK if such header is not present.
```java
var requiredIfPresentComponents = SignatureComponents.builder()
        // See Signature components above
        .build();
```

Apart from those defined above, verified signature can contain any components.

##### Signature parameters

Required parameters must be present in verified signature
For full list, see [SignatureParameterType](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/signature/SignatureParameterType.html)
```java
var requiredParameters = List.of(SignatureParameterType.KEY_ID, SignatureParameterType.CREATED, SignatureParameterType.NONCE);
```

Forbidden parameters must not be present in verified signature
```java
var forbiddenParameters = List.of(SignatureParameterType.ALGORITHM);
```

##### Signature context
```java
var signatureContext = SignatureContext.builder()
        // See Signature context above
        .build();
```

##### Public key getter
For details, see [PublicKeyInfo.Builder](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/signature/PublicKeyInfo.Builder.html)
and [VerificationSpec.Builder.publicKeyGetter()](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/signature/VerificationSpec.Builder.html#publicKeyGetter(com.visma.autopay.http.signature.CheckedFunction))
```java
// Exceptions thrown by the getter are used as the cause in Signature Exception
// thrown when verifying
private PublicKeyInfo getPublicKey(String keyId) throws MyGetterException {
    // Can be given as PublicKey object
    // or as X.509-base-64-encoded String, taken from a PEM file
    // or X.509-encoded byte array
    // HMAC keys should be provided as byte[] or base-64-encoded String
    var publicKey = publicKeyRepository.getPublicKey(keyId);

    return PublicKeyInfo.builder()
            .algorithm(SignatureAlgorithm.ED_25519)
            .publicKey(publicKey)
            .build();
}
```

##### Verification specification
For details, see [VerificationSpec.Builder](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/signature/VerificationSpec.Builder.html)
```java
var verificationSpec = VerificationSpec.builder()
        .signatureLabel("my-signature")
        .requiredComponents(requiredComponents)
        .requiredIfPresentComponents(requiredIfPresentComponents)
        
        // Required parameters can be provided as vararg 
        .requiredParameters(SignatureParameterType.NONCE, SignatureParameterType.KEY_ID)
        // or a collection
        .requiredParameters(Set.of(SignatureParameterType.NONCE, SignatureParameterType.KEY_ID))
        // Same goes for forbidden parameters
        .forbiddenParameters(SignatureParameterType.ALGORITHM)
        
        // Maximum age of verified signature in seconds. Age is computed basing
        // on created value. Signature is rejected if older than given age,
        // regardless of its expiration set in parameters
        .maximumAge(30)
        
        // Maximum clock "skew" for created parameter. It's for detecting
        // signatures from the "future".
        .maximumSkew(30)
        
        .context(signatureContext)
        .publicKeyGetter(this::getPublicKey)
        .build();
```

##### Verifying
For details, see [VerificationSpec.verify()](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/signature/VerificationSpec.html#verify())
```java
try {
    verificationSpec.verify();
} catch (SignatureException e) {
    if (e.getCause() instanceof MyGetterException) {
        // Thrown by getPublicKey()
        log.warn("Problem obtaining public key. spec={}", verificationSpec, e);
    } else {
        log.warn("Invalid signature. spec={}", verificationSpec, e);
    }
}
```

### Security providers

Default security provider is used for all operations: signing, verifying,
parsing keys provided as PKCS#8 / X.509. To use specific one, add it as first.
```java
Security.insertProviderAt(new BouncyCastleProvider(), 1);
```

Support for Edwards-Curve signatures (`SignatureAlgorithm.ED_25519`) was added
to JRE in Java 15. For older versions a third party provided must be used.

For ECDSA signatures (`SignatureAlgorithm.ECDSA_P256_SHA_256`),
`SHA256withECDSAinP1363Format` is used. If your provider uses different name,
like Bouncy Castle's `SHA256withPLAIN-ECDSA`, then a delegate must be
implemented.

```java
public class BouncyCastleP1363Provider extends Provider {
    public BouncyCastleP1363Provider() {
        super("BcP1363", "1.0", "Bouncy Castle - P1363 Bridge");
        // org.bouncycastle.jcajce.provider.asymmetric.ec.SignatureSpi
        put("Signature.SHA256withECDSAinP1363Format", SignatureSpi.ecCVCDSA256.class.getName());
    }
}

Security.insertProviderAt(new BouncyCastleProvider(), 1);
Security.insertProviderAt(new BouncyCastleP1363Provider(), 1);
```
See https://github.com/bcgit/bc-java/issues/751


## Digest Fields

Values for `Content-Digest` header can be computed by using `DigestCalculator`
class. Only "secure" algorithms are supported: `sha-256` and `sha-512`.
For details, see [DigestCalculator.calculateDigestHeader()](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/digest/DigestCalculator.html#calculateDigestHeader(byte%5B%5D,com.visma.autopay.http.digest.DigestAlgorithm))
```java
var jsonBody = objectMapper.writeValueAsBytes(requestOrResponseObject);
var contentDigest = DigestCalculator.calculateDigestHeader(jsonBody, DigestAlgorithm.SHA_256);
httpHeaders.add(DigestHeaders.CONTENT_DIGEST, contentDigest);
```

Digest algorithm can be chosen by using `Want-Content-Digest` header.
For details, see [DigestCalculator.calculateDigestHeader()](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/digest/DigestCalculator.html#calculateDigestHeader(byte%5B%5D,java.lang.String))
```java
var jsonBody = objectMapper.writeValueAsBytes(requestOrResponseObject);
var wantDigest = request.getHeader(DigestHeaders.WANT_CONTENT_DIGEST);

try {
    var contentDigest=DigestCalculator.calculateDigestHeader(jsonBody, wantDigest);
    responseHeaders.add(DigestHeaders.CONTENT_DIGEST, contentDigest);
} catch (DigestException e) {
    log.warn("Problems when computing wanted digest. wantDigest={}", wantDigest, e);    
}
```

For verification, `DigestVerifier` can be used.
For details, see [DigestVerifier.verifyDigestHeader()](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/digest/DigestVerifier.html#verifyDigestHeader(java.lang.String,byte%5B%5D))
```java
var jsonBody = objectMapper.writeValueAsBytes(requestOrResponseObject);
var contentDigest = request.getHeader(DigestHeaders.CONTENT_DIGEST);

try {
    DigestVerifier.verifyDigestHeader(contentDigest, jsonBody);
} catch (DigestException e) {
    log.warn("Invalid digest. digest={}", contentDigest, e);
}
```

## Structured Fields

### Class list

| Structured item | Java class                                                                                                                          | Internal storage                        |
|-----------------|-------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| List            | [StructuredList](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/structured/StructuredList.html)             | List\<StructuredItem\>                  | 
| Inner List      | [StructuredInnerList](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/structured/StructuredInnerList.html)   | List\<StructuredItem\>                  |
| Parameters      | [StructuredParameters](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/structured/StructuredParameters.html) | LinkedHashMap\<String, StructuredItem\> |
| Dictionary      | [StructuredDictionary](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/structured/StructuredDictionary.html) | LinkedHashMap\<String, StructuredItem\> |
| Integer         | [StructuredInteger](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/structured/StructuredInteger.html)       | long                                    |
| Decimal         | [StructuredDecimal](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/structured/StructuredDecimal.html)       | BigDecimal                              |
| String          | [StructuredString](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/structured/StructuredString.html)         | String                                  |
| Token           | [StructuredToken](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/structured/StructuredToken.html)           | String                                  |
| Byte Sequence   | [StructuredBytes](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/structured/StructuredBytes.html)           | byte[]                                  |
| Boolean         | [StructuredBoolean](https://visma-autopay.github.io/http-signatures/com/visma/autopay/http/structured/StructuredBoolean.html)       | boolean                                 |


### Creating items

Each item class has factory `of()` methods which accept types easily convertible
to internal storage.
```java
StructuredList.of(List.of("element-one", "element-two"));
StructuredList.of(55, "element-two", StructuredToken.of("hello"));

StructuredDictionary.of(Map.of("key1", "value1", "key2", 22));
StructuredDictionary.of("key1", "value1", "key2", 22, "key3", StructuredDecimal.of("22"));

StructuredInteger.of(55);
StructuredDecimal.of("22.34");
StructuredDecimal.of(new BigDecimal("22.34"));
StructuredString.of("This is a string");
StructuredToken.of("hello");
StructuredBytes.of(new byte[] {1, 2, 4});
```

Items with parameters can be created by using `withParams()` factory methods.
```java
StructuredInteger.withParams(23, Map.of("intparam", 10, "boolparam", true));
StructuredDecimal.withParams("15.2", StructuredParameters.of("one", 1, "two", "dos", "three", StructuredToken.of("tr")));
```

### Conversion rules

Lists, dictionaries and parameters can be created from Java varargs, collections
maps of objects. Those objects are converted to their Structured counterparts
by using the following rules.

| Java class                 | StructuredItem class |
|----------------------------|----------------------|
| Long, Integer, Short, Byte | StructuredInteger    |
| BigDecimal, Double, Float  | StructuredDecimal    |
| Boolean                    | StructuredBoolean    |
| byte[]                     | StructuredBytes      |
| String                     | StructuredString     |
| Collection                 | StructuredInnerList  |
| Enum                       | StructuredToken      |
 

### Accessing values

Item classes have accessor methods which names correspond to returned types.
```java
boolean boolValue = structuredBoolean.boolValue();
long longValue = structuredInteger.longValue();
int intValue = structuredInteger.intValue();
String strValue = structuredString.stringValue();
```

List- and map-backed items have their specialised accessors.
```java
List<Long> longValues = structuredList.longList();
List<StructuredItem> items = structuredList.itemList();

Map<String, StructuredItem> itemMap = structuredDictionary.itemMap();
Set<String> keys = structuredDictionary.keySet();
Map<String, Integer> intValuesMap = structuredDictionary.intMap();
Optional<StructuredItem> item = structuredDictionary.getItem("key");
Optional<String> stringValue = structuredDictionary.getString("strkey");

Map<String, String> stringParams = structuredParameters.stringMap();
Optional<String> stringParam = structuredParameters.getString("strkey");
structuredParameters.entrySet().stream()
        .filter(entry -> entry.getValue() instanceOf StructuredBoolean)
        .forEeach(entry -> doSomething(entry.getKey(), entry.getValue()));
```

Parameters can be accessed by using `parameters()` and then any map-like\
processing or individually.
```java
Optional<String> stringParam1 = item.parameters().getString("strparam");
Optional<String> stringParam2 = item.stringParam("strparam");
Optional<Boolean> boolParam = item.boolParam("boolparam");
```

### Serialization

For producing serialized values use `serialize()` method.
```java
var contentType = StructuredToken.withParams("text/plain", Map.of("charset", StructuredToken.of("utf-8")));
httpHeaders.put("Content-Type", contentType.serialize());
```

### Parsing

Each item class has a static `parse()` method which parses given string to item
class
```java
StructuredInteger.parse("45");
StructuredToken.parse("text/plan;charset=utf-8");
StructuredList.parse("21, ?1, ok");
StructuredDictionary.parse("key=value;param=ok, key2=value2");
```

If type is unknown then more generic methods can be used.
```java
// can return Structured Integer, Decimal, Bytes, String or Token 
StructuredItem item = StructuredItem.parse(header);
```

Lists and Dictionaries can be "guessed" by using `StructuredField.parse()`.
In case of ambiguity, the simplest implementation is returned (Item then List
then Dictionary).
```java
// StructuredDictionary
StructuredField field = StructuredField.parse("one=1");
        
// StructuredList is returned, but it's also a valid dictionary with `true` values
StructuredField.parse("one, two");

// StructuredToken is returned, but it's also a valid single-item list
StructuredField.parse("one");
```
