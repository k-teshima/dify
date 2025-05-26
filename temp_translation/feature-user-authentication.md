# ユーザー認証 (User Authentication)

## 1. 機能概要 (Feature Overview)

ユーザー認証機能は、ユーザーのログイン、登録、パスワード管理、セッション管理、およびAPIキー管理を取り扱います。コンソールユーザー向けには、直接的なメールアドレス/パスワード認証、メールベースのコードログイン、GoogleやGitHubなどの外部プロバイダー経由のOAuth認証といった複数の認証方法をサポートしています。Webアプリケーション（チャットボットなど）向けには、認証済みユーザーアクセスと、セッショントークンを使用したパブリック/匿名アクセスの両方のメカニズムを提供します。データソース向けのAPIキー認証も利用可能です。

## 2. アーキテクチャ図 (Architecture Diagram)

```mermaid
flowchart TD
    subgraph User/Client Interaction [ユーザー/クライアント操作]
        A[クライアントブラウザ/アプリ]
    end

    subgraph API Layer [APIレイヤー]
        B{APIゲートウェイ / ロードバランサー}
        C1[コンソール認証エンドポイント]
        C2[Webアプリ認証エンドポイント]
        C3[データソース認証エンドポイント]
    end

    subgraph Service Layer [サービスレイヤー]
        D[AccountService]
        E[WebAppAuthService]
        F[ApiKeyAuthService]
        G[PassportService (JWT)]
        H[OAuthプロバイダーサービス/ライブラリ]
    end

    subgraph Data & Session Stores [データストアとセッションストア]
        I[ユーザーデータベース (PostgreSQL)]
        J[セッションストア (Redis - リフレッシュトークン、レート制限用)]
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

**図の説明 (Diagram Explanation):**

*   **クライアントブラウザ/アプリ (Client Browser/App):** 認証のためのユーザーインターフェース。
*   **APIゲートウェイ / ロードバランサー (API Gateway / Load Balancer):** APIリクエストのエントリーポイント。
*   **コンソール認証エンドポイント (Console Auth Endpoints):** メインの管理者/管理コンソール用の認証を処理します（例：ログイン、OAuth、パスワードリセット）。
*   **Webアプリ認証エンドポイント (Web App Auth Endpoints):** デプロイされたWebアプリケーション（例：チャットボット、匿名アクセスのパスポートを含む）の認証を処理します。
*   **データソース認証エンドポイント (Data Source Auth Endpoints):** 外部データソースの認証（APIキー、OAuth）を処理します。
*   **AccountService:** ユーザーアカウント管理、パスワードハッシュ化、資格情報検証、およびコンソールセッション管理のためのコアサービス。
*   **WebAppAuthService:** Webアプリケーション固有の認証ロジックのためのサービス。
*   **ApiKeyAuthService:** データソースのAPIキーベース認証を管理するためのサービス。
*   **PassportService (JWT):** JWTトークンの発行と検証を行うライブラリ。
*   **OAuthプロバイダーサービス/ライブラリ (OAuth Providers Service/Libs):** 外部OAuthプロバイダー（Google、GitHub、Notionなど）と連携するモジュール/クラス。
*   **ユーザーデータベース (PostgreSQL) (User Database (PostgreSQL)):** ユーザーアカウント、テナント情報、ロールなどを保存します。
*   **セッションストア (Redis) (Session Store (Redis)):** リフレッシュトークン、レート制限カウンター、およびその他のセッション関連データを保存するために使用されます。

## 3. 関連モジュール (Related Modules)

| パス (Path)                                               | 役割 (Role)                                                                    | 主要なクラス/関数 (Major Classes/Functions)                                                                                                |
| -------------------------------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `api/services/account_service.py`                  | ユーザーアカウントおよびテナント管理、コンソール認証ロジック、トークン処理。     | `AccountService`, `TenantService`, `RegisterService`, `_store_refresh_token`, `get_account_jwt_token`, `authenticate` |
| `api/services/webapp_auth_service.py`              | Webアプリケーション認証ロジック、エンドユーザー管理。                | `WebAppAuthService` (注: このファイルは初期リストにはありませんでしたが、`api/controllers/web/login.py`から推測されます) |
| `api/services/auth/api_key_auth_service.py`        | データソース用のAPIキー認証管理。                       | `ApiKeyAuthService`                                                                                                    |
| `api/controllers/console/auth/login.py`            | コンソールログインエンドポイント（メール/パスワード、メールコード、リフレッシュトークン）。      | `LoginApi`, `LogoutApi`, `EmailCodeLoginApi`, `RefreshTokenApi`                                                          |
| `api/controllers/console/auth/oauth.py`            | コンソールOAuthログインエンドポイント（GitHub、Google）。                           | `OAuthLogin`, `OAuthCallback`, `_generate_account`                                                                       |
| `api/controllers/console/auth/forgot_password.py`  | コンソールパスワード忘れフローのエンドポイント。                                   | `ForgotPasswordSendEmailApi`, `ForgotPasswordCheckApi`, `ForgotPasswordResetApi`                                         |
| `api/controllers/console/auth/activate.py`         | コンソール招待ユーザーアクティベーションエンドポイント。                                | `ActivateApi`, `ActivateCheckApi`                                                                                      |
| `api/controllers/console/auth/data_source_bearer_auth.py` | データソースAPIキー認証エンドポイント。                          | `ApiKeyAuthDataSource`, `ApiKeyAuthDataSourceBinding`                                                                  |
| `api/controllers/console/auth/data_source_oauth.py` | データソースOAuth認証エンドポイント（例：Notion）。                | `OAuthDataSource`, `OAuthDataSourceCallback`, `OAuthDataSourceBinding`                                                   |
| `api/controllers/web/login.py`                     | Webアプリケーションログインエンドポイント（メール/パスワード、メールコード）。           | `LoginApi` (Web), `EmailCodeLoginApi` (Web)                                                                              |
| `api/controllers/web/passport.py`                  | Webアプリケーションパスポートエンドポイント（主にパブリック/匿名アクセス用のトークン発行）。 | `PassportResource`                                                                                                     |
| `api/libs/passport.py`                             | コアJWT発行および検証ロジック。                                | `PassportService`                                                                                                      |
| `api/libs/password.py`                             | パスワードハッシュ化および比較ユーティリティ。                                | `hash_password`, `compare_password`, `valid_password`                                                                  |
| `api/libs/oauth.py`                                | OAuthクライアント実装（例：GitHub、Google）。                    | `GitHubOAuth`, `GoogleOAuth`                                                                                           |
| `extensions/ext_redis.py`                          | セッションストア、レート制限などのためのRedisクライアント。                     | `redis_client`                                                                                                         |
| `extensions/ext_database.py`                       | データベース（SQLAlchemy）設定およびセッション管理。                     | `db`                                                                                                                   |

## 4. データフロー (Data Flow)

### a. コンソール メールアドレス/パスワード ログイン (Console Email/Password Login)

1.  **入力 (Input):** ユーザーがクライアントインターフェース経由でメールアドレスとパスワードを提供し、`/console/api/login` にPOSTリクエストを送信します。
2.  **処理 (Processing):**
    *   `api/controllers/console/auth/login.py -> LoginApi.post()` がリクエストを受信します。
    *   `api/services/account_service.py -> AccountService.authenticate()` を呼び出し、ユーザーデータベースに対して資格情報を検証します。これには `libs/password.compare_password` を使用したパスワード比較が含まれます。
    *   認証が成功すると、`AccountService.login()` が呼び出されます。
    *   `AccountService.get_account_jwt_token()`（内部で `api/libs/passport.py -> PassportService.issue()` を使用）がJWTアクセストークンを生成します。
    *   リフレッシュトークンが生成され、`AccountService._store_refresh_token()` を介してRedisに保存されます。
    *   ユーザーの最終ログインIPと時刻が更新されます。
3.  **出力 (Output):** `access_token` と `refresh_token` を含むJSONレスポンスがクライアントに送信されます。

### b. Webアプリ 匿名アクセス (パスポート) (Web App Anonymous Access (Passport))

1.  **入力 (Input):** Webアプリケーション（例：チャットボットUI）が `/api/passport` にGETリクエストを送信します。これには `X-App-Code` ヘッダーと、オプションでクエリパラメータとして `user_id`（セッションID）が含まれます。
2.  **処理 (Processing):**
    *   `api/controllers/web/passport.py -> PassportResource.get()` がリクエストを受信します。
    *   `X-App-Code` を検証し、データベースから `Site` および `App` の詳細を取得します。
    *   `user_id` が提供された場合、既存の `EndUser` を検索します。見つからない場合、または `user_id` が提供されない場合、新しい `EndUser` が一意の `session_id`（提供されなかった場合）で作成され、匿名としてマークされます。
    *   `api/libs/passport.py -> PassportService.issue()` が呼び出され、`app_id`、`app_code`、および `end_user_id` を含むJWTアクセストークンが生成されます。
3.  **出力 (Output):** `access_token` を含むJSONレスポンスがWebアプリケーションに送信されます。

### c. コンソール OAuth ログイン (例: GitHub) (Console OAuth Login (e.g., GitHub))

1.  **入力 (Input):** ユーザーがコンソールログインページで「GitHubでログイン」をクリックします。
2.  **リダイレクト (Redirection):**
    *   クライアントは `/console/api/oauth/login/github`（`api/controllers/console/auth/oauth.py -> OAuthLogin.get()`）にリダイレクトされます。
    *   このエンドポイントはGitHub認証URLを構築し（`api/libs/oauth.py -> GitHubOAuth.get_authorization_url()` を使用）、ユーザーのブラウザをGitHubにリダイレクトします。
3.  **外部認証 (External Authentication):** ユーザーはGitHubで認証し、アプリケーションを承認します。
4.  **コールバック (Callback):** GitHubは認証 `code` と共にユーザーを `/console/api/oauth/authorize/github`（`api/controllers/console/auth/oauth.py -> OAuthCallback.get()`）にリダイレクトします。
5.  **処理 (Processing):**
    *   `OAuthCallback.get()` は `GitHubOAuth.get_access_token()` を使用して `code` をアクセストークンと交換します。
    *   次に `GitHubOAuth.get_user_info()` を使用してGitHubからユーザー情報を取得します。
    *   `_generate_account()`（`api/controllers/console/auth/oauth.py` 内）が呼び出されます：
        *   OpenID（GitHubユーザーID）またはメールアドレスで既存のアカウントを検索しようとします。
        *   アカウントが存在せず、登録が許可されている場合、`RegisterService.register()` を介して新しい `Account` が作成されます。
        *   GitHub OpenIDは `AccountService.link_account_integrate()` を使用してアカウントにリンクされます。
        *   必要に応じてテナント/ワークスペースが作成または割り当てられます。
    *   `AccountService.login()` が呼び出され、JWTアクセスおよびリフレッシュトークンが生成されます。
6.  **出力 (Output):** ユーザーはトークンと共にコンソールのメインページにリダイレクトされます（通常はクエリパラメータとして渡されるか、フロントエンドがそれらを保存するように処理します）。

## 5. 拡張ポイント (Extension Points)

*   **コンソールログイン用の新しいOAuthプロバイダーの追加:**
    1.  `api/libs/oauth.py` に `OAuthSignIn` を継承するか、同様のインターフェースを実装する新しいOAuthクライアントクラスを作成します（例：`MyProviderOAuth(OAuthSignIn)`）。
    2.  `get_authorization_url`、`get_access_token`、`get_user_info` などのメソッドを実装します。
    3.  新しいプロバイダー設定（クライアントID、シークレット、URL）を `configs/config.py` または環境変数に追加します。
    4.  `api/controllers/console/auth/oauth.py` の `get_oauth_providers()` 関数内の `OAUTH_PROVIDERS` 辞書に新しいプロバイダーインスタンスを登録します。
    5.  新しいOAuthプロバイダー用のボタン/リンクを含むようにフロントエンドを更新します。
*   **データソース認証用の新しいOAuthプロバイダーの追加:**
    1.  コンソールOAuthと同様のパターンに従いますが、`api/controllers/console/auth/data_source_oauth.py` および必要に応じて対応するサービスレイヤーに適合させます。
    2.  `libs.oauth_data_source.py`（例：`NotionOAuth`）は、データソースOAuthの処理方法の例を示しています。
*   **セッション管理のカスタマイズ:**
    *   `configs/config.py` でトークンの有効期限を変更します（例：`ACCESS_TOKEN_EXPIRE_MINUTES`、`REFRESH_TOKEN_EXPIRE_DAYS`）。
    *   `AccountService` でリフレッシュトークンのRedis使用法を調整します。
*   **パスワードポリシーの変更:**
    *   `libs/password.py -> valid_password()` でパスワード検証ロジックを更新します。
*   **二要素認証（2FA）の追加:**
    *   これは大幅な拡張であり、おそらく以下を含むでしょう：
        *   2FA設定を保存するために `Account` モデルを変更します。
        *   2FA設定、検証のために `AccountService` に新しいサービスメソッドを追加します。
        *   2FA管理用の新しいAPIエンドポイントを作成します。
        *   `LoginApi` および潜在的にOAuthコールバックのログインフローを更新して、2FAコードの入力を求めるようにします。
```
