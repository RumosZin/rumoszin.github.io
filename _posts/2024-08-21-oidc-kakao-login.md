---
title: "Spring boot에서 OIDC 카카오 로그인 구현하기"
author:
date: 2024-08-21 21:30:00 +0900
categories: [Backend, Auth]
image: /assets/img/240821-1.png
tags: [Spring, OIDC]
---

<br>

사이드 프로젝트에서 카카오 로그인을 구현해야 할 일이 생겼다. 이전 프로젝트들에서 소셜 로그인은 해본 적이 없어서, 도전 정신을 가지고 구현했다. OIDC 방식의 소셜 로그인을 팀원들이 신기해 했고, 더욱 잘 설명해주고자 글을 남긴다!

작년에 비해 올해 들어 인턴을 하면서 공식 문서를 세심히 살펴보는 습관이 생겼다. (나의 격언이 10개의 블로그보다 하나의 공식 문서일 정도) 마찬가지로 100개의 블로그들이 설명을 자세히 해도, 카카오의 공식 문서를 따라갈 수는 없다. 이름이 "카카오 로그인"인데 당연한 결과이다.

<br>

## **필요한 설정**

### **애플리케이션 설정**

[카카오 로그인 사용 시 필요한 설정](https://developers.kakao.com/docs/latest/ko/kakaologin/prerequisite)

자세한 설명을 읽고, 애플리케이션에 필요한 항목들을 선택하고 설정하면 된다. 잘 설명하고 있는 블로그들이 많으니 이 부분은 스킵한다.

<br>

### **Spring boot 프로젝트 설정**

{: file='build.gradle'}

```gradle
implementation 'io.jsonwebtoken:jjwt-api:0.12.6'
runtimeOnly 'io.jsonwebtoken:jjwt-impl:0.12.6'
runtimeOnly 'io.jsonwebtoken:jjwt-jackson:0.12.6'
```

{: file='application.yml'}

```yaml
oauth2:
  kakao:
    client-id: 64cc**********************
    redirect-uri: http://localhost:8080/api/auth/login/oauth/kakao
    token-uri: https://kauth.kakao.com/oauth/token
    metadata-uri: https://kauth.kakao.com/.well-known/openid-configuration
    public-key-uri: https://kauth.kakao.com/.well-known/jwks.json

jwt:
  secret-key: ****************************
  access-token-validity-in-seconds: 1800 # 30분
  refresh-token-validity-in-seconds: 86400 # 1일

```

[카카오 로그인 사용 시 필요한 설정](https://developers.kakao.com/docs/latest/ko/kakaologin/prerequisite) 이곳에서 각 값들을 위치를 자세히 설명하고 있으므로, 자세히 읽었으면 여기까지 문제 없을 것이다.

<br>

## **구현**

[카카오 로그인 공식 문서](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#request-code-request)에 나오는 너무나도 중요한 시퀀스 다이어그램이다. 우리는 이 그림을 보면서, 한 단계씩 해결하면 된다!

![Untitled](/assets/img/240821-1.png){: width="100%"}

<br>

### **Step 0**

Step 1보다 앞에 있는 클라이언트에서 서버로의 요청을 Step 0 이라고 하겠다. 나는 백엔드를 구현할 것이므로, [공식 문서의 요청 예제](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#request-code-sample)를 바탕으로, 클라이언트에서 보내는 요청을 만들 수 있다.

⚠️ **openId 설정을 했다면 뒤에 `scope=openid`가 있어야 하고, `client-secret`도 설정했다면 적어줘야 한다.**

https://kauth.kakao.com/oauth/authorize?client_id=64cc**********************&redirect_uri=http://localhost:8080/api/auth/login/oauth/kakao&response_type=code&scope=openid

<br>

### **Step 1 : 인가 코드 받기**

위의 요청이 들어오면, 카카오 Auth 서버는 client_id를 통해 우리 애플리케이션의 정보를 가지고 로그인 화면을 띄운다. 사용자가 로그인을 하면, 동의 화면을 출력한다. 동의하고 계속하기를 누르면, **지정해둔 Redirect URI로 인가 코드를 전달한다.**

여기까지는 카카오 Auth 서버에 의해 이루어지므로, 우리는 Step 2부터 구현하면 된다!

<br>

### **Step 2 : 토큰 받기**

Redirect URI로 들어오는 인가 코드를 받아서, 토큰을 받아보자. Redirect URI를, API로 설정하면 인가 코드를 적절히 처리할 수 있다! 아래와 같이 code로 인가 코드를 받자.

- 일관된 응답 구조를 가져가기 위해 `ApiResponse`를 만들었다. 이건 각자 프로젝트에서 합의한 대로 응답을 만들면 된다. 인가 코드를 받아서 처리하는 Step 2의 결과물은 토큰이므로, `LoginReponse`에서 `accessToken`을 정의한다.

{: file='/project/auth/controller/AuthController.java'}

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {
    private final AuthService authService;

    @GetMapping("/login/oauth/kakao")
    public ApiResponse<LoginResponse> loginWithKakao(@RequestParam("code") String code) {
        return ApiResponse.success(authService.loginWithKakao(code), ResponseCode.USER_LOGIN_SUCCESS.getMessage());
    }
}
```

<br>

✅ **SOLID의 인터페이스 분리 원칙에 따라, 상세한 서비스 코드는 노출되지 않도록 인터페이스를 두고, 실제 서비스 코드의 구현체를 두는 식으로 구조를 가져간다.**

`AuthService`의 `loginWithKakao`에서는 어떤 작업이 이루어져야 할까? 이 부분이 중요한 부분인데, 시퀀스 다이어그램에 나와있듯이 카카오 Auth 서버에 POST 요청을 보내서 토큰을 발급 받아야 한다.

✅ **`loginWithKakao`에서 토큰을 발급 받는 것을 바로 구현하지 않고, external 내의 `KakaoProvider`에서 구현했다. 외부에 요청을 보내는 것들은 external로 관리하고자 이런 결정을 내렸다!**

[토큰을 얻기 위한 요청 예제](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#request-token-sample) 형식을 보고 그대로 Spring Java 코드로 치환했다.

{: file='/project/auth/external/KakaoProvider.java'}

```java
@Component
@RequiredArgsConstructor
@CacheConfig(cacheNames = "kakaoPublicKeyList")
@Slf4j
public class KakaoProvider {
    private final KakaoLoginProperties kakaoLoginProperties; // application.yml에 정의된 카카오 로그인 관련 값들을 정의하는 Properties

    // 카카오 인가 코드로 카카오 토큰 발급
    public KakaoToken getTokenByCode(String code) {
        return WebClient.create()
                .post()
                .uri(kakaoLoginProperties.getTokenUri())
                .header(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_FORM_URLENCODED_VALUE)
                .body(BodyInserters.
                        fromFormData("grant_type", "authorization_code")
                        .with("client_id", kakaoLoginProperties.getClientId())
                        .with("redirect_uri", kakaoLoginProperties.getRedirectUri())
                        .with("code", code))
                .retrieve()
                .bodyToMono(KakaoToken.class)
                .block();
    }
}
```

`AuthService`의 `loginWithKakao`에서 `KakaoProvider`에 접근해서 토큰을 받아오는 것까지 성공했다면, Step 2의 토큰 받기를 성공한 것이다!

<br>

### **Step 3 : 사용자 로그인 처리**

OpenID Connect를 사용하므로, 받아온 토큰에 대해서 D 토큰 유효성 검증 과정을 거친다. OpenID Connect 방식이 무엇이고, 왜 사용하는지에 대해서는 다른 글에서 자세히 다룰 예정이다.

✅ **OpenID에서의 ID 토큰 유효성 검증이므로, OIDCProvider에서 따로 검증하도록 코드를 분리했다.**

토큰 검증 과정도 여러 스텝을 거친다. 하나씩 확인해보자!

#### **Step 3-1 : 토큰으로부터 페이로드 추출**

- JWT는 헤더, 페이로드, 서명으로 구성되어 있으므로, 온점을 기준으로 토큰을 나눈다. 세 부분으로 나누어지지 않으면 예외를 발생시킨다.
- 페이로드를 추출해서 Base64 방식으로 디코딩하고, KakaoIdTokenPayload 객체로 변환한다.

아래는 위에 적은 내용을 구현한 OIDCProvider의 일부이다.

{: file='/project/auth/external/OIDCProvider.java'}

```java
private KakaoIdTokenPayload extractPayloadFromTokenString(String token) {
    // 온점(.)을 기준으로 헤더, 페이로드, 서명을 분리
    String[] parts = token.split("\\.");
    if (parts.length != 3) {
        throw new AuthException(ResponseCode.BAD_REQUEST); // Invalid JWT token
    }

    // 페이로드를 추출하여 Base64 방식으로 디코딩
    byte[] payloadBytes = Base64.getUrlDecoder().decode(parts[1]);

    // 페이로드를 KakaoIDTokenPayload 객체로 변환
    try {
        return new ObjectMapper().readValue(payloadBytes, KakaoIdTokenPayload.class);
    } catch (IOException e) {
        throw new AuthException(ResponseCode.INTERNAL_SERVER_ERROR);
    }
}
```

#### **Step 3-2 : 토큰 페이로드 검증**

[공식 문서에서 ID 토큰 페이로드에 대해 설명하고 있다.](https://developers.kakao.com/docs/latest/ko/kakaologin/rest-api#request-token-response-id-token)

다른 값들은 페이로드에 담긴 회원 정보이고, 우리는 세 가지 값을 검증해야 한다.

- 토큰을 발급한 인증 기관 정보 iss
- 토큰이 발급된 앱의 키 aud
- 만료된 토큰인지 exp

토큰에서 페이로드를 추출하고, 페이로드에서 세 가지 값을 검증하는 `OIDCProvider`의 `verifyPayload`이다.

{: file='/project/auth/external/OIDCProvider.java'}

```java
private void verifyPayload(String token, String iss) {
    // 카카오 ID 토큰으로부터 페이로드 추출
    KakaoIdTokenPayload kakaoIDTokenPayload = extractPayloadFromTokenString(token);

    // iss: https://kauth.kakao.com와 일치해야 함
    if (!kakaoIDTokenPayload.getIss().equals(iss)) {
        throw new AuthException(ResponseCode.UNAUTHORIZED);
    }
    // aud: 서비스 앱 키와 일치해야 함
    if (!kakaoIDTokenPayload.getAud().equals(kakaoLoginProperties.getClientId())) {
        throw new AuthException(ResponseCode.UNAUTHORIZED);
    }
    // exp: 현재 UNIX 타임스탬프(Timestamp)보다 큰 값 필요(ID 토큰의 만료 여부 확인)
    if (kakaoIDTokenPayload.getExp().compareTo(System.currentTimeMillis() / 1000) < 0) {
        throw new AuthException(ResponseCode.UNAUTHORIZED);
    }
}
```

#### **Step 3-3 : 토큰 서명 검증**

서명은 토큰의 위변조 여부를 확인하는 부분이다. 여기서 조금 애를 먹었는데, 내 의문점들을 구글링 해보니 [kakao developers 측에서 답변한 것](https://devtalk.kakao.com/t/topic/123255)이 있었다! 답변으로 추천해주신 블로그의 예제를 참고할 수 있었다.

`KakaoIdTokenPayload` 객체를 반환하는데, sub은 회원번호, nickname은 카카오에서 회원의 이름이다. 이 객체를 빌더 패턴으로 생성하여 반환하므로, 이것을 반환 받은 시점부터 `loginWithKakao` 내에서 회원 정보를 가지고 있게 된다.

{: file='/project/auth/external/OIDCProvider.java'}

```java
private KakaoIdTokenPayload verifySignature(String token, KakaoIdTokenPublicKeyList kakaoIdTokenPublicKeyList) {
    // 카카오 ID 토큰 공개키 목록에서 헤더의 kid에 해당하는 공개키 값 검색
    KakaoIdTokenJwk kakaoIDTokenJWK = getOIDCPublicKey(extractHeaderKidFromTokenString(token), kakaoIdTokenPublicKeyList);

    // 서명 검증
    try {
        Claims payload = Jwts.parser()
                .verifyWith(getRSAPublicKeyFromJWK(kakaoIDTokenJWK)) // JWK로 RSA Public Key 생성
                .build()
                .parseSignedClaims(token)
                .getPayload();

        return KakaoIdTokenPayload.builder()
                .sub(payload.getSubject())
                .nickname(payload.get("nickname").toString())
                .build();
    } catch (SignatureException e) {
        throw e;
    } catch (SecurityException e) {
        throw new AuthException(ResponseCode.UNAUTHORIZED);
    }
}
```

#### **Step 3-4 : 데이터베이스 회원 정보 저장**

- 반환 받은 사용자 회원번호는 kakaoId이므로, 이것으로 User 데이터베이스를 검색해서 존재하는 회원이면 들어온 시간을 업데이트 하고, 회원가입한 사용자이면 새로 데이터를 저장한다.
- 사용자 kakaoId, username으로 refreshToken을 발행한다.
- 이때 발급받은 refreshToken을 따로 관리해야 하므로, 토큰 레포지토리를 이용했다. 발행한 refreshToken을 업데이트 한다.
- 최종적으로, accessToken을 발급받는다! 로그인 응답으로 accessToken을 반환한다.

✅ **Jwt를 이용해 토큰을 생성하는 부분은, JwtProvider에서 관리하도록 코드를 분리했다.**

<br>

Step 2 ~ 여기까지의 내용들을 순서대로 구현한 최종적인 loginWithKakao의 코드는 다음과 같다.

{: file='/project/auth/service/AuthServiceImpl.java'}

```java
@Service
@RequiredArgsConstructor
public class AuthServiceImpl implements AuthService {
    private final TokenRepository tokenRepository;
    private final UserRepository userRepository;

    private final KakaoProvider kakaoProvider;
    private final OIDCProvider OIDCProvider;
    private final JwtProvider jwtProvider;

    private final String ISS = "https://kauth.kakao.com";

    @Override
    public LoginResponse loginWithKakao(String code) {
        // 카카오 토큰 발급
        KakaoToken kakaoToken = kakaoProvider.getTokenByCode(code);

        // 카카오 ID 토큰 유효성 검증
        KakaoIdTokenPayload kakaoIDTokenPayload;
        try {
            kakaoIDTokenPayload = OIDCProvider.verify(kakaoToken.getIdToken(), ISS, kakaoProvider.getOIDCPublicKeyList());
        } catch (SignatureException e) {
            kakaoIDTokenPayload = OIDCProvider.verify(kakaoToken.getIdToken(), ISS, kakaoProvider.getUpdatedOIDCPublicKeyList());
        }

        final String kakaoId = kakaoIDTokenPayload.getSub();
        final String userName = kakaoIDTokenPayload.getNickname();

        // DB User 테이블 갱신
        // 기존 회원: UPDATE request_date
        // 신규 회원: INSERT
        userRepository.findByKakaoId(kakaoId)
                .map(u -> {
                    u.updateRequestDate(LocalDateTime.now());
                    return userRepository.save(u);
                })
                .orElseGet(() -> userRepository.save(User.builder()
                        .nickname(userName)
                        .createDate(LocalDateTime.now())
                        .requestDate(LocalDateTime.now())
                        .kakaoId(kakaoId)
                        .build()
                ));

        // refresh_token 발행
        String refreshToken = jwtProvider.createRefreshToken(kakaoId, userName);

        // DB Token 테이블 갱신
        Token refreshTokenRecord = tokenRepository.save(Token.builder()
                .refreshToken(refreshToken)
                .kakaoId(kakaoId)
                .expiresAt(jwtProvider.getExpirationFromToken(refreshToken).toInstant().atZone(ZoneId.systemDefault()).toLocalDateTime())
                .build()
        );

        // access_token 발행
        String accessToken = jwtProvider.createAccessToken(kakaoId, refreshTokenRecord.getId());

        return new LoginResponse(accessToken);
    }
}
```

<br>

최종적으로 로그인 결과 데이터베이스를 확인하면 다음과 같이 잘 나오는 것을 볼 수 있다!

![Untitled](/assets/img/240821-2.png){: width="100%"}

<br>

## **삽질**

인가 코드를 받아오는 것까지는 되는데, Step 2 카카오 토큰을 받아오는 부분에서 500 에러가 나는 경우가 있다. 링크로 들어가보면 아래와 같은 에러가 날 것이다.

```
{
  “error”:“invalid_client”¸
  “error_description”:'Bad client credentials'¸
  “error_code”:'KOE010'
}
```

### **해결 1**

[카카오 developers에서 공식적으로 알려주는 트러블 슈팅 방법 4가지](https://devtalk.kakao.com/t/koe010-bad-client-credentials/115388)가 있다. 이것을 차근차근 따라해본다! 90퍼센트는 여기서 해결 가능하다.

<br>

### **해결 2**

Redirect URI와, 요청 하는 https의 redirect_uri, spring boot의 application.yml의 세 값이 일치하는지 확인한다. (여기서 계속 삽질했다..)

- 내 애플리케이션 > 제품 설정 > 카카오 로그인
  ![Untitled](/assets/img/240821-3.png){: width="100%"}

- https://kauth.kakao.com/oauth/authorize?client_id=64cc@@@@@@@@@@@@&**redirect_uri=http://localhost:8080/api/auth/login/oauth/kakao**&response_type=code&scope=openid

- application.yml
  ![Untitled](/assets/img/240821-4.png){: width="100%"}

<br>
<br>

<script src="https://utteranc.es/client.js"
        repo="RumosZin/rumoszin.github.io"
        issue-term="pathname"
        theme="github-light"
        crossorigin="anonymous"
        async>
</script>
