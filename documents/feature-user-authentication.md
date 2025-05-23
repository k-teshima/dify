# User Authentication

## 1. Feature Overview

The User Authentication feature handles user login, registration, password management, session control, and API key management. It supports multiple authentication methods: direct email/password credentials, email-based code login, and OAuth-based authentication via external providers like Google and GitHub for console users. For web applications (e.g., chatbots), it provides mechanisms for both authenticated user access and public/anonymous access using session-based tokens. API key authentication is also available for data sources.

## 2. Architecture Diagram

```mermaid
flowchart TD
    subgraph User/Client Interaction
        A[Client Browser/App]
    end

    subgraph API Layer
        B{API Gateway / Load Balancer}
        C1[Console Auth Endpoints]
        C2[Web App Auth Endpoints]
        C3[Data Source Auth Endpoints]
    end

    subgraph Service Layer
        D[AccountService]
        E[WebAppAuthService]
        F[ApiKeyAuthService]
        G[PassportService (JWT)]
        H[OAuth Providers Service/Libs]
    end

    subgraph Data & Session Stores
        I[User Database (PostgreSQL)]
        J[Session Store (Redis for Refresh Tokens, Rate Limiting)]
    end

    A --> B;
    B --> C1;
    B --> C2;
    B --> C3;

    C1 --> D;
    C1 --> G;
    C1 --> H;
    C1 --> J;

    C2 --> E;
    C2 --> D;
    C2 --> G;
    C2 --> J;

    C3 --> F;
    C3 --> I;

    D --> I;
    D --> J;
    D --> G;

    E --> I;
    E --> J;
    E --> G;

    F --> I;
```

**Diagram Explanation:**

*   **Client Browser/App:** User interface for authentication.
*   **API Gateway / Load Balancer:** Entry point for API requests.
*   **Console Auth Endpoints:** Handles authentication for the main admin/management console (e.g., login, OAuth, password reset).
*   **Web App Auth Endpoints:** Handles authentication for deployed web applications (e.g., chatbots, including anonymous access passport).
*   **Data Source Auth Endpoints:** Handles authentication for external data sources (API Keys, OAuth).
*   **AccountService:** Core service for user account management, password hashing, credential validation, and console session management.
*   **WebAppAuthService:** Service for web application specific authentication logic.
*   **ApiKeyAuthService:** Service for managing API key based authentication for data sources.
*   **PassportService (JWT):** Library for issuing and verifying JWT tokens.
*   **OAuth Providers Service/Libs:** Modules/classes that interface with external OAuth providers (Google, GitHub, Notion).
*   **User Database (PostgreSQL):** Stores user accounts, tenant information, roles, etc.
*   **Session Store (Redis):** Used for storing refresh tokens, rate limiting counters, and potentially other session-related data.

## 3. Related Modules

| Path                                               | Role                                                                    | Major Classes/Functions                                                                                                |
| -------------------------------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `api/services/account_service.py`                  | User account & tenant management, console auth logic, token handling.     | `AccountService`, `TenantService`, `RegisterService`, `_store_refresh_token`, `get_account_jwt_token`, `authenticate` |
| `api/services/webapp_auth_service.py`              | Web application authentication logic, end-user management.                | `WebAppAuthService` (Note: This file was not in the initial list but is inferred from `api/controllers/web/login.py`) |
| `api/services/auth/api_key_auth_service.py`        | API key authentication management for data sources.                       | `ApiKeyAuthService`                                                                                                    |
| `api/controllers/console/auth/login.py`            | Console login endpoints (email/password, email code, refresh token).      | `LoginApi`, `LogoutApi`, `EmailCodeLoginApi`, `RefreshTokenApi`                                                          |
| `api/controllers/console/auth/oauth.py`            | Console OAuth login endpoints (GitHub, Google).                           | `OAuthLogin`, `OAuthCallback`, `_generate_account`                                                                       |
| `api/controllers/console/auth/forgot_password.py`  | Console forgot password flow endpoints.                                   | `ForgotPasswordSendEmailApi`, `ForgotPasswordCheckApi`, `ForgotPasswordResetApi`                                         |
| `api/controllers/console/auth/activate.py`         | Console invited user activation endpoints.                                | `ActivateApi`, `ActivateCheckApi`                                                                                      |
| `api/controllers/console/auth/data_source_bearer_auth.py` | Data source API key authentication endpoints.                          | `ApiKeyAuthDataSource`, `ApiKeyAuthDataSourceBinding`                                                                  |
| `api/controllers/console/auth/data_source_oauth.py` | Data source OAuth authentication endpoints (e.g., Notion).                | `OAuthDataSource`, `OAuthDataSourceCallback`, `OAuthDataSourceBinding`                                                   |
| `api/controllers/web/login.py`                     | Web application login endpoints (email/password, email code).           | `LoginApi` (Web), `EmailCodeLoginApi` (Web)                                                                              |
| `api/controllers/web/passport.py`                  | Web application passport endpoint for issuing tokens (often for public/anonymous access). | `PassportResource`                                                                                                     |
| `api/libs/passport.py`                             | Core JWT issuing and verification logic.                                | `PassportService`                                                                                                      |
| `api/libs/password.py`                             | Password hashing and comparison utilities.                                | `hash_password`, `compare_password`, `valid_password`                                                                  |
| `api/libs/oauth.py`                                | OAuth client implementations (e.g., GitHub, Google).                    | `GitHubOAuth`, `GoogleOAuth`                                                                                           |
| `extensions/ext_redis.py`                          | Redis client for session store, rate limiting, etc.                     | `redis_client`                                                                                                         |
| `extensions/ext_database.py`                       | Database (SQLAlchemy) setup and session management.                     | `db`                                                                                                                   |

## 4. Data Flow

### a. Console Email/Password Login

1.  **Input:** User provides email and password via a client interface, which makes a POST request to `/console/api/login`.
2.  **Processing:**
    *   `api/controllers/console/auth/login.py -> LoginApi.post()` receives the request.
    *   It calls `api/services/account_service.py -> AccountService.authenticate()` to validate credentials against the user database. This involves password comparison using `libs.password.compare_password`.
    *   If authentication is successful, `AccountService.login()` is called.
    *   `AccountService.get_account_jwt_token()` (which internally uses `api/libs/passport.py -> PassportService.issue()`) generates a JWT access token.
    *   A refresh token is generated and stored in Redis via `AccountService._store_refresh_token()`.
    *   User's last login IP and time are updated.
3.  **Output:** A JSON response containing the `access_token` and `refresh_token` is sent to the client.

### b. Web App Anonymous Access (Passport)

1.  **Input:** A web application (e.g., chatbot UI) makes a GET request to `/api/passport`, including an `X-App-Code` header and optionally a `user_id` (session ID) as a query parameter.
2.  **Processing:**
    *   `api/controllers/web/passport.py -> PassportResource.get()` receives the request.
    *   It validates the `X-App-Code` and retrieves the `Site` and `App` details from the database.
    *   If `user_id` is provided, it tries to find an existing `EndUser`. If not found or `user_id` is not provided, a new `EndUser` is created with a unique `session_id` (if one wasn't provided) and marked as anonymous.
    *   `api/libs/passport.py -> PassportService.issue()` is called to generate a JWT access token containing `app_id`, `app_code`, and `end_user_id`.
3.  **Output:** A JSON response containing the `access_token` is sent to the web application.

### c. Console OAuth Login (e.g., GitHub)

1.  **Input:** User clicks "Login with GitHub" on the console login page.
2.  **Redirection:**
    *   Client is redirected to `/console/api/oauth/login/github` (`OAuthLogin.get()` in `api/controllers/console/auth/oauth.py`).
    *   This endpoint constructs the GitHub authorization URL (using `GitHubOAuth.get_authorization_url()` from `api/libs/oauth.py`) and redirects the user's browser to GitHub.
3.  **External Authentication:** User authenticates on GitHub and authorizes the application.
4.  **Callback:** GitHub redirects the user back to `/console/api/oauth/authorize/github` (`OAuthCallback.get()` in `api/controllers/console/auth/oauth.py`) with an authorization `code`.
5.  **Processing:**
    *   `OAuthCallback.get()` exchanges the `code` for an access token using `GitHubOAuth.get_access_token()`.
    *   It then fetches user information from GitHub using `GitHubOAuth.get_user_info()`.
    *   `_generate_account()` (in `api/controllers/console/auth/oauth.py`) is called:
        *   It attempts to find an existing account by OpenID (GitHub user ID) or email.
        *   If no account exists and registration is allowed, a new `Account` is created via `RegisterService.register()`.
        *   The GitHub OpenID is linked to the account using `AccountService.link_account_integrate()`.
        *   Tenant/workspace is created or assigned if necessary.
    *   `AccountService.login()` is called to generate JWT access and refresh tokens.
6.  **Output:** The user is redirected to the console's main page with the tokens (usually passed as query parameters or handled by frontend to store them).

## 5. Extension Points

*   **Adding a new OAuth Provider for Console Login:**
    1.  Create a new OAuth client class in `api/libs/oauth.py` inheriting from `OAuthSignIn` or implementing a similar interface (e.g., `MyProviderOAuth(OAuthSignIn)`).
    2.  Implement methods like `get_authorization_url`, `get_access_token`, `get_user_info`.
    3.  Add the new provider configuration (client ID, secret, URLs) to `configs/config.py` or environment variables.
    4.  Register the new provider instance in the `OAUTH_PROVIDERS` dictionary within `get_oauth_providers()` function in `api/controllers/console/auth/oauth.py`.
    5.  Update the frontend to include a button/link for the new OAuth provider.
*   **Adding a new OAuth Provider for Data Source Authentication:**
    1.  Follow a similar pattern to console OAuth, but adapt it for `api/controllers/console/auth/data_source_oauth.py` and its corresponding service layers if needed.
    2.  The `libs.oauth_data_source.py` (e.g. `NotionOAuth`) shows an example of how data source OAuth is handled.
*   **Customizing Session Management:**
    *   Modify token expiry times in `configs/config.py` (e.g., `ACCESS_TOKEN_EXPIRE_MINUTES`, `REFRESH_TOKEN_EXPIRE_DAYS`).
    *   Adjust Redis usage for refresh tokens in `AccountService`.
*   **Modifying Password Policies:**
    *   Update password validation logic in `libs/password.py -> valid_password()`.
*   **Adding Two-Factor Authentication (2FA):**
    *   This would be a significant extension, likely involving:
        *   Modifying `Account` model to store 2FA configuration.
        *   Adding new service methods in `AccountService` for 2FA setup, verification.
        *   Creating new API endpoints for 2FA management.
        *   Updating the login flow in `LoginApi` and potentially OAuth callbacks to prompt for 2FA codes.
```
