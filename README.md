# How to create an App into ChatGPT App SDK

The **new OpenAl Apps SDK** lets developers **bring their products directly into ChatGPT** with <mark>**custom Ul components**</mark>, <mark>**API access**</mark>, and <mark>**user context**</mark> that can persist across chats. 
It's built on Model Context Protocol (**MCP**), which defines **how ChatGPT communicates with our app through** <mark>**tools**</mark>, <mark>**resources**</mark>, and <mark>**structured data**</mark>.

A step-by-step flow of how your OAuth-protected FastMCP server works with ChatGPT and Auth0

- [What OAuth means in the ChatGPT Apps SDKs settings?](https://github.com/adriensieg/OpenAI-ChatGPT-App-SDK-OAuth2.1/blob/master/README.md#what-oauth-means-in-the-chatgpt-apps-sdks-settings)
- [What's the point of using OAuth at all?]()
- [Step-by-step flow of how your OAuth-protected FastMCP server works with ChatGPT and Auth0](https://github.com/adriensieg/OpenAI-ChatGPT-App-SDK-OAuth2.1/tree/master?tab=readme-ov-file#step-by-step-flow-of-how-your-oauth-protected-fastmcp-server-works-with-chatgpt-and-auth0)
    - [Concepts Represented in Flow]()
    - [Mermaid Flow]()
- [Use Cases](https://github.com/adriensieg/OpenAI-ChatGPT-App-SDK-OAuth2.1/blob/master/README.md#use-cases)

## What OAuth means in the ChatGPT Apps SDKs settings?

When ChatGPT gives us the option between **"No Authentication"** and **"OAuth"**, it's referring to *how ChatGPT authenticates itself to our service, ~~not how our end users authenticate to us~~
- **No Authentication**:ChatGPT makes unauthetnicated request to our endpoint
- **OAuth**: ChatGPT will perform an OAuth 2.1 **Client credentials** or **Authorization Code** to obtain an Access token that it will include in requests to our MCP endpoint. \

The **OAuth flow** authenticates ChatGPT (the client) to our MCP service. It does not ~~authenticate~~ or ~~identify the individual human~~ ChatGPT user to us. We won't **receive any user identity** info unless OpenAI explictly passes it. 
OAuth by itself does **not identify a user**; it just delegates authoization. 
    - In *traditional web apps*, you often combine **OAuth + OpenID Connect** (OIDC) to both authenticate and authorize users.
    - In the *ChatGPT SDK integration*, **only OAuth 2.0 is used** — not OIDC. So there's **no user identity payload** (no `ID token`, **no claims about the user**).

#### Conclusion:
- We're authorizing ChatGPT to access your service.
- We're not authenticating ChatGPT users individually.
- Our app sees a single client (ChatGPT), not multiple end-users

### Endpoints to create
- Authorization endpoint (`/authorize`)
- Token endpoint (`/oauth/token`)
- JWKS endpoint (`/.well-known/jwks.json`)
- Registration endpoint (`/oidc/register`) if DCR is enabled
- OIDC discovery (`/.well-known/openid-configuration`)

## Step-by-step flow of how your OAuth-protected FastMCP server works with ChatGPT and Auth0

#### Concepts Represented in Flow

| Concept                                | Description                                                                                                        |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| **OAuth 2.1**                          | Core protocol governing authorization between client, resource owner, and server.                                  |
| **OIDC (OpenID Connect)**              | Identity layer built atop OAuth 2.0—adds ID tokens, discovery, and userinfo.                                       |
| **DCR (Dynamic Client Registration)**  | Allows ChatGPT to register dynamically with Auth0 as a unique OAuth client at runtime.                             |
| **PKCE (Proof Key for Code Exchange)** | Prevents interception of authorization codes; replaces static client secrets for public clients.                   |
| **JWT (JSON Web Token)**               | Self-contained access token signed by Auth0; includes `iss`, `sub`, `aud`, `exp`, `scope`.                         |
| **JWKS (JSON Web Key Set)**            | Auth0’s public keys used by your MCP server to verify token signatures.                                            |
| **RS256 (RSA Signature with SHA-256)** | Algorithm used by Auth0 for signing JWTs—server verifies with public key.                                          |
| **Scopes**                             | Fine-grained permission claims (e.g., `mcp:execute`, `openid`, `profile`, `email`).                                |
| **Bearer Tokens**                      | Access tokens passed in `Authorization: Bearer <token>`; whoever holds them can access.                            |
| **WWW-Authenticate Header**            | Standard 401 response header prompting the client to re-authorize.                                                 |
| **SSE (Server-Sent Events)**           | Long-lived HTTP stream used by FastMCP for bi-directional ChatGPT ↔ MCP communication.                             |
| **Access Token Validation**            | Your middleware verifies JWT signature, audience, issuer, expiration, and scopes.                                  |
| **Refresh Tokens**                     | Long-lived tokens allowing ChatGPT to renew access tokens without user re-login (requires `offline_access` scope). |

```mermaid
sequenceDiagram
    autonumber
    participant U as 🧑 User
    participant C as 💬 ChatGPT (OAuth Client / Apps SDK)
    participant M as ⚙️ MCP Server (FastMCP + Auth Middleware)
    participant A as 🔐 Auth0 (Authorization & OIDC Server)

    Note over U,C: Step 0 — User installs / connects MCP App in ChatGPT

    %% Resource Discovery
    C->>M: (1) GET /.well-known/oauth-protected-resource
    M-->>C: (2) JSON → { resource, authorization_servers, scopes_supported, bearer_methods_supported }
    Note over C: ChatGPT learns Auth0 is the authorization server and what scopes to request

    %% Auth Server Discovery & DCR
    C->>A: (3) GET /.well-known/openid-configuration
    A-->>C: (4) Returns OIDC metadata (auth/token/jwks/registration endpoints)
    Note over C,A: ChatGPT now knows how to register dynamically and where endpoints live

    %% Dynamic Client Registration (DCR)
    C->>A: (5) POST /oidc/register  {redirect_uris, grant_types, token_endpoint_auth_method:"none", ...}
    A-->>C: (6) Returns client_id (unique per connector), auth_method:"none"
    Note over C: ChatGPT is now a dynamically registered public client

    %% Authorization Code Flow with PKCE
    C->>C: (7) Generate code_verifier + code_challenge (PKCE)
    C->>A: (8) GET /authorize?response_type=code&client_id=&code_challenge=&scope=&audience=&redirect_uri=
    A->>U: (9) Display login + consent UI
    U->>A: (10) Authenticate + approve scopes
    A-->>C: (11) Redirect to redirect_uri with ?code=<authorization_code>

    %% Token Exchange
    C->>A: (12) POST /oauth/token {grant_type=authorization_code, code, code_verifier, redirect_uri, client_id}
    A-->>C: (13) JSON → {access_token (JWT), id_token, refresh_token?, token_type, expires_in}
    Note over C,A: PKCE verified, client authenticated as public, token bound to resource audience

    %% Token Usage (Access Protected Resource)
    C->>M: (14) GET /sse  Authorization: Bearer <access_token>
    Note right of M: OAuth middleware intercepts request
    M->>A: (15) GET /.well-known/jwks.json (if cache empty)
    A-->>M: (16) JWKS {keys:[{kid,n,e,alg, ...}]}
    M->>M: (17) Validate JWT signature (RS256)<br/>• match kid → key<br/>• verify iss/aud/exp<br/>• parse scopes
    alt Token invalid or expired
        M-->>C: (18a) 401 Unauthorized + WWW-Authenticate header
        C->>A: (18b) Refresh or restart auth flow
    else Token valid
        M-->>C: (19) 200 OK (SSE stream open)
    end

    %% MCP Tool Execution via SSE
    C->>M: (20) Send tool_call event {tool:"hello", args:{name:"Alice"}}
    M->>M: (21) Execute hello() → "Hello, Alice from W9010MNCPX!"
    M-->>C: (22) Send tool_result SSE event
    C->>U: (23) Display "Hello, Alice from W9010MNCPX!"

    %% Refresh & Lifecycle
    Note over C,A: Access token expires → ChatGPT uses refresh_token (offline_access scope) to renew silently
    C->>A: (24) POST /oauth/token {grant_type=refresh_token, refresh_token, client_id}
    A-->>C: (25) Returns new access_token
    Note over C,M: New token used in next Authorization header

```

## Use Cases
Linkedin Post: 
- Geofencing
- Waiting Time
- Food intent
- Ordering - Stripe Payment
