# Spring Authorization Server Demo

This repository is an example on how to use and extend Spring Authorization Server.

The Spring Authorization Server is a framework that provides implementations of the [OAuth 2.1](https://oauth.net/2.1/)
and [OpenID Connect 1.0](https://openid.net/specs/openid-connect-core-1_0.html) specifications.

The general flow is the one shown below. Depending on the grant type, might be a
user that requests a token. Then, a client (can be a web or mobile app) requests the user's
token to the Authorization Server, and uses the token in subsequent calls
to the Resource Server (backend). The Resource Server applies the authorization rules
and decides whether the user has enough permissions to access the desire resources.

```mermaid
flowchart LR

A[User] --> B(Client)
B <-->|Getting a token through a grant type| C[Authorization Server]
B --> E[Resource Server]
```
Once you start the Authorization Server locally, you can find all the available endpoints
in http://localhost:8080/.well-known/openid-configuration.

What will I explain in this repo?
1. How does the client gets the token?
2. How does the Resource Server validates and gets data with the token?

## How Does the Client Gets the Token?
There are several ways to obtain a token and those are called grant types. This example shows how to get a token using 3 grant types:

1. **Authorization code (PKCE):** a way to obtain a token when a user is present in the flow.
2. **Client credentials:** a way to obtain a token for a service (no user involved).
3. **Refresh Token:** a way to get a new access token (after the used token is expired) without asking the user to authenticate again.

### Authorization Code Grant Type

```mermaid
sequenceDiagram
    User->>Client: Request a resource
    Client->>User: Redirect to Login page + challenge (PKCE)
    User->>AuthorizationServer: Login page
    AuthorizationServer->>Client: Redirects to redirect_uri + Authorization code
    Client->>AuthorizationServer: POST request to get the token<br/>code + client credentials + verifier (PKCE)
    AuthorizationServer->>Client: Return the token
```

The authorization code grant type flow looks like the following in practice.

1. Go to the browser and put your login page http://localhost:8080/oauth2/authorize?response_type=code&client_id=client&scope=openid&redirect_uri=http://127.0.0.1:8080/login/oauth2/code/client&code_challenge=hawxbDkWfRXTsok_Rlc1SKf5r0nf2ZV5drFknC3EAYY&code_challenge_method=S256
   > **Note**: For the purpose of this example, I used [pkce](https://example-app.com/pkce) to generate code_challenge from a random string. Keep the random string, you will use it in a following step.
2. Add the user credentials and sign in.
3. The browser will redirect you to a URL that looks like http://127.0.0.1:8080/login/oauth2/code/client?code=fPo0m-l9TD2KyMKFKsq3ywJ00m93gOv_V-GoG6wjh95LgjIgJMu3a4y11tT6qKWy7mn3LxLePcQvR8X52LYWHZQfq5HERNba9tb3oDvDfjIY010EFkVuNXyo3n5q2O0T&continue
4. POST request to get the token. </br>
    Request: `curl -X POST "http://localhost:8080/oauth2/token?grant_type=authorization_code&code=pmEUUzywceppVJH5XgP-aZ55FzxhB60YpSx3DWDd2vwPioU70_SrgT1HLLV9Rnr6cmps1mpYsq0BiTzWwd7HfRYmwvtJ3iOjgvUk2gmlTu_luiiLepTMFhmNO-2xbY79&client_id=client&redirect_uri=http://127.0.0.1:8080/login/oauth2/code/client&code_verifier=14d1023bac9c91bac5bbfee59263ac9a1297ea8e305d316d9c946274" -u "client:secret"` </br>
    Response:</br>
    ```json
    {
        "access_token": "eyJraWQiOiJiYWY5MjU1My01OThjLTRlYzQtYjZkNC1hYmE1MDMyMjI0YzUiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyIiwiYXVkIjoiY2xpZW50IiwibmJmIjoxNjg5MjcxNTYwLCJzY29wZSI6WyJvcGVuaWQiXSwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwIiwiZXhwIjoxNjg5MjcxODYwLCJpYXQiOjE2ODkyNzE1NjB9.B71rrkWgEWt4_rSxajf9EEBAVJthmbrNHyBaOxppA2AsxfMjDUxS0hZsXLB7NYJZgC0_nWZDLTIYDvJhkXaX77vIv--ppWhL89vdH64eMhlStJ_SBgQBoa_sLqYVdOVj22rezpQlessZpwwE0jOQlaDyP2Am7L2bZL0CgCtvXVtbPNSuCkqXHUkJLhk-NUvTmng19OPncRE_4eOG8tSolXQd6ldidgKvJ2Nz9EU3Lv3_UAwdpovVA99Cd4xcSUEwmX43DzI-daNPiM3cMGtczoG7f7CIbGdM7aDRBDoWcozACTNrp0wlSv6uiXPR13NJHsnBNYEiuPi-V3dtYRbsjA",
        "refresh_token": "waKQKQbjHSsH_AcCxr8v7A9YBITb71QVsl6WRTeUUdFuEZcRtasWQY40GQ0GcSOzBznM0Q_Ry9zFCFLlZYVbi4uDlCWODWatFadNwlZI5Y9_dBZZiV9ptrSR9f6_zowg",
        "scope": "openid",
        "id_token": "eyJraWQiOiJiYWY5MjU1My01OThjLTRlYzQtYjZkNC1hYmE1MDMyMjI0YzUiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyIiwiYXVkIjoiY2xpZW50IiwiYXpwIjoiY2xpZW50IiwiYXV0aF90aW1lIjoxNjg5MjcxNTI5LCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAiLCJleHAiOjE2ODkyNzMzNjAsImlhdCI6MTY4OTI3MTU2MCwic2lkIjoicDBWVlRfNVFIRFlPVmc0UVp5dm9CV0NsaENycmdBV0RhbFpRY2lXQUFFZyJ9.cTB6Q08PZCkW7xU5Ilpqf4_vmwTa59Ry7PTzTN8otOuLtj7Jni64oGiT-bi2sZGvH5lMSG5n7RHkDekmeWkhLKdCTCTHiMLh_Ur3Y-wRF5YHs12AmLEQJlu7lRY6FMSuxhFEWtdGKh_oiusK8fRfuFSIqnZaqQgwhHDft0g4qXoW6fh63UEQdAuk-YZHHM1dIp3wZDuF2Swc9UlXO2rB36zkpEX90oMCcIEq3U3u8LTPjMkz71as1QaKPuMes70dtchLERf5bhVI9V4kPd1AilRwg4gaKtmixdWofOpf_cRRmVDVd3QDnpQXW9kkTkPKZTZs84O3e9x8DmfELr7qgw",
        "token_type": "Bearer",
        "expires_in": 299
    }
    ```
   The code_verifier is the random string used in step 1 to create the code_challenge.

### Client Credentials Grant Type

```mermaid
sequenceDiagram
    Client->>AuthorizationServer: Request token (client + secret)
    AuthorizationServer->>Client: Return token
```
The client credentials grant type flow looks like the following in practice.

Request: `curl -X POST "http://localhost:8080/oauth2/token?grant_type=client_credentials" -u "client:secret"` </br>
Response: </br>
```json
{
    "access_token": "eyJraWQiOiI1Mjk2M2NiNi02MTI4LTQwNzQtYjkyYS1jNmI5NGFmYWI0ZWQiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJjbGllbnQiLCJhdWQiOiJjbGllbnQiLCJuYmYiOjE2ODkyNzI0MDQsImlzcyI6Imh0dHA6Ly9sb2NhbGhvc3Q6ODA4MCIsImV4cCI6MTY4OTI3MjcwNCwiaWF0IjoxNjg5MjcyNDA0fQ.oOLi5aPpvnCgMr_yJ8WmUJFRJkuOvKba1CObzJScuiFLrc6-5pB6I0O0XkV2qlwc1ybSJNPJ0cRgWjOujkSn8hZQm_glaOtWErB7sQQHCcvT_GJheDI7PYz6zK_hplGavuunLRW0v5bRHCJxQPNt0PprEtIXg57flGzuijBZthrJRASAWuHdopJ_nO-UUMPWm_3BERrq7hovsusd7W1YGNcj_-59rmU84BfCce3yzJMBbO32nQZMWWnDzrk_F-PrwX6tTkjsRFnv9q7_hCfB59e9OmmlOvslZLtPpvya7oGgZwPrOkfV4Ll6X1u8Bj9EcuV4X_1zqup_bIyBNxW4QQ",
    "token_type": "Bearer",
    "expires_in": 299
}
```

### Refresh Token Grant Type

```mermaid
sequenceDiagram
    User->>Client: Request a resource
    Client->>AuthorizationServer: Request token (refresh token)
    AuthorizationServer->>Client: Return access + refresh token
```

The refresh token grant type flow looks like the following in practice.

Request: `curl -X POST "http://localhost:8080/oauth2/token?grant_type=refresh_token&refresh_token=rBdt7JNpirEjjs_vASc_pGvbYEo3mRwMG4AbQarj2UGlzcjiqfAM8pzaqxUMcg3CbAp-rgSrtG05BNKonNBUwLGEeB-I2Pduf8cWDdRVOeZtWzum8SqgXqwp2sIUjMVM" -u "client:secret"`
Response: </br>
```json
{
    "access_token": "eyJraWQiOiIyODYzZDIzZC03NzQ4LTRiZWQtYmI1Ni02N2U5NGZkZjVlYWQiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyIiwiYXVkIjoiY2xpZW50IiwibmJmIjoxNjg5MzIwNDc2LCJzY29wZSI6WyJvcGVuaWQiXSwiaXNzIjoiaHR0cDovL2xvY2FsaG9zdDo4MDgwIiwiZXhwIjoxNjg5MzIwNzc2LCJpYXQiOjE2ODkzMjA0NzZ9.KWyzGH8vQOGhGfxB-kWcn8kwvpev6knQtJG1bLPrhN40G-3GSSIKf2jX0CqOSVTNyNjqD5miPwfB2W2QxJZ4NcgY9Dxu1-yQaLajZ30l3kvAnyOy6pPPg87LsJEdtMbz0Ye9GJ8ggEZsZsPZcD0Yq6dt4GCLJ5HdL23nyPWQL_q555MI0fPkRx_MpBMVnHEpvmj7fOzwQUY35sV3zrW2rPF4BAxBQCFiSFrZunmn87LPuzFi0IyeW1lYzgoG7n_HYlBnjEo_ttBKc6bSoZSjzqqFcOpNYX5HX_4FkfM8iQWGe_uYd4SmE63pUVg9LJn0BXfieGwbX1dJrfRBNp_PXQ",
    "refresh_token": "rBdt7JNpirEjjs_vASc_pGvbYEo3mRwMG4AbQarj2UGlzcjiqfAM8pzaqxUMcg3CbAp-rgSrtG05BNKonNBUwLGEeB-I2Pduf8cWDdRVOeZtWzum8SqgXqwp2sIUjMVM",
    "scope": "openid",
    "id_token": "eyJraWQiOiIyODYzZDIzZC03NzQ4LTRiZWQtYmI1Ni02N2U5NGZkZjVlYWQiLCJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyIiwiYXVkIjoiY2xpZW50IiwiYXpwIjoiY2xpZW50IiwiYXV0aF90aW1lIjoxNjg5MzIwMzc1LCJpc3MiOiJodHRwOi8vbG9jYWxob3N0OjgwODAiLCJleHAiOjE2ODkzMjIyNzYsImlhdCI6MTY4OTMyMDQ3Niwic2lkIjoiX29acXA2aHdZbDZBRGFIZVpkd0RpY2E3WFJ1VGhtRW9Ob0E1UGlldldVcyJ9.IP22RBiDWQOfiCeZ6pRS_FAnbcJkQFnGmefoysV0PgO8p-DZI74m6TBm5gGb9ZI4FAjLUTzrw515FVN0xG49igPLCfUw5ApVqqIGY_IifR1oJXe1Nj56XonTjY2bj-Srs0b6smFHEwX37qmVhojuJ1e6ztx8b6bXbcQIXzbAxDdE27rMujBzo-7xnI2ez7hvMf1IK6blq6VOf9At1RyxAx7bru0gpjf2YmsuvN6YFtNA9VO-E54cbUTlkqTAb5exPnpeVDp_VKf8GhRnOU8i_JmY1J6YAV2EfHKXmvzhiA35t_JEGF8W9K2kqla4VOjaCHbPDlbpKUyQw5MHaiA71A",
    "token_type": "Bearer",
    "expires_in": 299
}
```

## How does the Resource Server validates and gets data with the token?

A token can be opaque and non-opaque.

- An opaque token doesn't contain any data and in this case the Authorization server implements
the introspection endpoint to return data. This data is crucial for the Resource Server to apply the authorization rules.
- A non-opaque token contains data and a popular implementation of non-opaque tokens is the [JWT](https://jwt.io/introduction).

## Pending Implementation

- [ ] Creating a `redirect_uri` of the client.
- [ ] Storing users and clients in the database.
- [ ] Adding a Resource Server example.
- [ ] Modifying the token's payload.

# References

1. [OAuth 2.1](https://oauth.net/2.1/)
2. [Spring Security Fundamentals 2022 by Laur Spilca](https://www.youtube.com/playlist?list=PLEocw3gLFc8X_a8hGWGaBnSkPFJmbb8QP)
3. [Spring Authorization Server](https://docs.spring.io/spring-authorization-server/docs/current/reference/html/index.html)
4. [PCKE](https://oauth.net/2/pkce/)
