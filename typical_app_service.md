# AWS上にサービスを構築する場合のアーキテクチャに関する一考察

以下のシステム構成図は、AWSクラウド環境でのプロダクション、ステージング、および開発の3つの異なるVPC（Virtual Private Cloud）におけるフルスタックアプリケーションのデプロイメントと運用管理の全体像を示しています。各環境は独立したVPCに配置され、セキュリティとスケーラビリティを確保しています。

```mermaid
graph LR
    subgraph AWS Cloud
        subgraph Prod VPC
            subgraph Prod Public Subnet
                Prod_ALB[Prod ALB <br> <br> Auto Scaling]
                Prod_APIG[Prod API Gateway]
                Prod_CloudFront[Prod CloudFront]
            end
            subgraph Prod Private Subnet
                Prod_Frontend[Prod Frontend Server -ECS- <br> <br> Security Group: Allow 80 from Prod ALB <br> <br> Auto Scaling]
                Prod_Backend[Prod Backend Server -ECS- <br> <br> Security Group: Allow 80 from Prod ALB <br> <br> Auto Scaling]
                Prod_RDS[Prod RDS Database <br> <br> Security Group: Allow 3306 from Prod Backend <br> <br> Read Replicas]
                Prod_ElastiCache[Prod ElastiCache <br> <br> Security Group: Allow 6379 from Prod Backend]
                Prod_SQS[Prod Amazon SQS]
                Prod_Lambda[Prod AWS Lambda]
            end
            IAM -->|Manage Access| Prod_VPC
            Shield -->|DDoS Protection| Prod_VPC
        end

        subgraph Staging VPC
            subgraph Staging Public Subnet
                Staging_ALB[Staging ALB]
                Staging_APIG[Staging API Gateway]
                Staging_CloudFront[Staging CloudFront]
            end
            subgraph Staging Private Subnet
                Staging_Frontend[Staging Frontend Server -ECS-]
                Staging_Backend[Staging Backend Server -ECS-]
                Staging_RDS[Staging RDS Database]
                Staging_ElastiCache[Staging ElastiCache]
                Staging_SQS[Staging Amazon SQS]
                Staging_Lambda[Staging AWS Lambda]
            end
            IAM -->|Manage Access| Staging_VPC
        end

        subgraph Dev VPC
            subgraph Dev Public Subnet
                Dev_ALB[Dev ALB]
                Dev_APIG[Dev API Gateway]
                Dev_CloudFront[Dev CloudFront]
            end
            subgraph Dev Private Subnet
                Dev_Frontend[Dev Frontend Server -ECS-]
                Dev_Backend[Dev Backend Server -ECS-]
                Dev_RDS[Dev RDS Database]
                Dev_ElastiCache[Dev ElastiCache]
                Dev_SQS[Dev Amazon SQS]
                Dev_Lambda[Dev AWS Lambda]
            end
            IAM -->|Manage Access| Dev_VPC
        end

        Cognito[AWS Cognito]
        Org[AWS Organizations]
        KMS[AWS Key Management Service]
        CW[CloudWatch]
        ECR[Elastic Container Registry]
        Athena[Amazon Athena]
        QuickSight[Amazon QuickSight]
        Glue[AWS Glue]
        S3[Amazon S3]
        XRay[AWS X-Ray]
        WAF[AWS WAF]
        SNS[Amazon SNS]
    end

    subgraph CI/CD Pipeline
        GitHub[GitHub] -->|Trigger| CP[CodePipeline]
        CP -->|Source| CB[CodeBuild]
        CB -->|Build and Test| CB
        CB -->|Build Frontend Docker Image| ECR
        CB -->|Build Backend Docker Image| ECR
        CP -->|Deploy to Dev| CDDev[CodeDeploy]
        CDDev -->|Update ECS Task Definition with ECR Image| Dev_Frontend
        CDDev -->|Update ECS Task Definition with ECR Image| Dev_Backend
        CP -->|Deploy to Staging| CDStaging[CodeDeploy]
        CDStaging -->|Update ECS Task Definition with ECR Image| Staging_Frontend
        CDStaging -->|Update ECS Task Definition with ECR Image| Staging_Backend
        CP -->|Deploy to Prod| CDProd[CodeDeploy]
        CDProd -->|Update ECS Task Definition with ECR Image| Prod_Frontend
        CDProd -->|Update ECS Task Definition with ECR Image| Prod_Backend
    end

    subgraph Log Analysis and Monitoring
        Dev[Developer] -->|Analyze Logs| Athena
        Dev[Developer] -->|Visualize Data| QuickSight
        CW -->|Alarms| SNS -->|Notify| Dev[Developer]
    end

    Browser[Web Browser] -->|HTTPS| Prod_CloudFront
    Prod_CloudFront -->|HTTPS| Prod_ALB
    Prod_ALB -->|HTTP| Prod_Frontend
    Android[Android App] -->|HTTPS| Prod_APIG
    iOS[iOS App] -->|HTTPS| Prod_APIG
    Prod_Frontend -->|API Call| Prod_APIG
    Prod_APIG -->|HTTP| Prod_ALB
    Prod_ALB -->|HTTP| Prod_Backend
    Prod_Backend -->|SQL| Prod_RDS
    Prod_Backend -->|Cache| Prod_ElastiCache
    Prod_Backend -->|Async Tasks| Prod_SQS
    Prod_SQS -->|Trigger| Prod_Lambda

    Browser[Web Browser] -->|HTTPS| Staging_CloudFront
    Staging_CloudFront -->|HTTPS| Staging_ALB
    Staging_ALB -->|HTTP| Staging_Frontend
    Staging_Frontend -->|API Call| Staging_APIG
    Staging_APIG -->|HTTP| Staging_ALB
    Staging_ALB -->|HTTP| Staging_Backend
    Staging_Backend -->|SQL| Staging_RDS
    Staging_Backend -->|Cache| Staging_ElastiCache
    Staging_Backend -->|Async Tasks| Staging_SQS
    Staging_SQS -->|Trigger| Staging_Lambda

    Browser[Web Browser] -->|HTTPS| Dev_CloudFront
    Dev_CloudFront -->|HTTPS| Dev_ALB
    Dev_ALB -->|HTTP| Dev_Frontend
    Dev_Frontend -->|API Call| Dev_APIG
    Dev_APIG -->|HTTP| Dev_ALB
    Dev_ALB -->|HTTP| Dev_Backend
    Dev_Backend -->|SQL| Dev_RDS
    Dev_Backend -->|Cache| Dev_ElastiCache
    Dev_Backend -->|Async Tasks| Dev_SQS
    Dev_SQS -->|Trigger| Dev_Lambda

    Cognito -->|Authenticate| IAM
    IAM -->|Authorize| Prod_APIG
    IAM -->|Authorize| Staging_APIG
    IAM -->|Authorize| Dev_APIG
    Org -->|Account Management| Prod_VPC
    Org -->|Account Management| Staging_VPC
    Org -->|Account Management| Dev_VPC
    KMS -->|Encryption Keys| Prod_RDS
    KMS -->|Encryption Keys| Prod_Backend
    KMS -->|Encryption Keys| Staging_RDS
    KMS -->|Encryption Keys| Staging_Backend
    KMS -->|Encryption Keys| Dev_RDS
    KMS -->|Encryption Keys| Dev_Backend
    XRay -->|Trace| Prod_APIG
    XRay -->|Trace| Prod_Backend
    XRay -->|Trace| Staging_APIG
    XRay -->|Trace| Staging_Backend
    XRay -->|Trace| Dev_APIG
    XRay -->|Trace| Dev_Backend
    WAF -->|Protect| Prod_CloudFront
    WAF -->|Protect| Staging_CloudFront
    WAF -->|Protect| Dev_CloudFront
    IAM -->|Audit Logs| CW[CloudWatch]
    Org -->|Audit Logs| CW[CloudWatch]
    Prod_APIG -->|Masked Logs| CW[CloudWatch]
    Prod_Backend -->|Masked Logs| CW[CloudWatch]
    Prod_Frontend -->|Masked Logs| CW[CloudWatch]
    Staging_APIG -->|Masked Logs| CW[CloudWatch]
    Staging_Backend -->|Masked Logs| CW[CloudWatch]
    Staging_Frontend -->|Masked Logs| CW[CloudWatch]
    Dev_APIG -->|Masked Logs| CW[CloudWatch]
    Dev_Backend -->|Masked Logs| CW[CloudWatch]
    Dev_Frontend -->|Masked Logs| CW[CloudWatch]
    CW -->|Store Logs| S3
    Glue -->|Catalog Logs| S3
    Athena -->|Query Logs| S3
    QuickSight -->|Visualize| S3
```

### 全体構成の概要

1. **AWS Cloud**:
    - **Prod VPC (Production Environment)**: プロダクション環境のVPC。高可用性とスケーラビリティを確保しています。
    - **Staging VPC (Staging Environment)**: ステージング環境のVPC。プロダクション環境と同様の設定で、テストおよび検証に使用されます。
    - **Dev VPC (Development Environment)**: 開発環境のVPC。開発者が新機能や修正を試すための環境です。

### 各環境の詳細構成

#### Prod VPC (Production Environment)
- **Public Subnet**:
    - **Prod ALB**: Application Load Balancer。自動スケーリングをサポートし、トラフィックをFrontendおよびBackendサーバーに分散します。
    - **Prod API Gateway**: APIリクエストのエントリーポイント。認証とリクエストのルーティングを行います。
    - **Prod CloudFront**: CDNサービス。静的コンテンツのキャッシングと高速配信を提供します。

- **Private Subnet**:
    - **Prod Frontend Server (ECS)**: フロントエンドアプリケーションをホストします。セキュリティグループによりALBからのトラフィックを許可します。
    - **Prod Backend Server (ECS)**: バックエンドアプリケーションをホストします。同様に、ALBからのトラフィックを許可します。
    - **Prod RDS Database**: リレーショナルデータベースサービス。バックエンドサーバーからのSQLクエリを受け付けます。
    - **Prod ElastiCache**: メモリキャッシュサービス。バックエンドサーバーからのキャッシュリクエストを処理します。
    - **Prod Amazon SQS**: メッセージキューサービス。非同期タスクの管理に使用されます。
    - **Prod AWS Lambda**: サーバーレスコンピューティングサービス。SQSメッセージの処理などに使用されます。

#### Staging VPC (Staging Environment) & Dev VPC (Development Environment)
- **Public Subnet**:
    - **Staging/Dev ALB**: Application Load Balancer。プロダクションと同様の役割。
    - **Staging/Dev API Gateway**: API Gateway。プロダクションと同様の役割。
    - **Staging/Dev CloudFront**: CloudFront。プロダクションと同様の役割。

- **Private Subnet**:
    - **Staging/Dev Frontend Server (ECS)**: フロントエンドサーバー。プロダクションと同様の役割。
    - **Staging/Dev Backend Server (ECS)**: バックエンドサーバー。プロダクションと同様の役割。
    - **Staging/Dev RDS Database**: RDSデータベース。プロダクションと同様の役割。
    - **Staging/Dev ElastiCache**: ElastiCache。プロダクションと同様の役割。
    - **Staging/Dev Amazon SQS**: Amazon SQS。プロダクションと同様の役割。
    - **Staging/Dev AWS Lambda**: AWS Lambda。プロダクションと同様の役割。

### 共通サービス
- **AWS Cognito**: ユーザー認証と認可を管理します。
- **IAM (Identity and Access Management)**: AWSリソースへのアクセスを管理します。各環境のVPCにアクセス権限を付与します。
- **AWS Organizations**: 複数のAWSアカウントを統合管理します。各環境のVPCにアカウント管理を提供します。
- **AWS KMS (Key Management Service)**: データの暗号化キーを管理します。各環境のRDSおよびBackendサーバーに対して暗号化キーを提供します。
- **AWS CloudWatch**: ログとメトリクスの収集および監視を行います。各コンポーネントの監視データを収集します。
- **AWS X-Ray**: 分散トレースを提供し、各環境のAPIGおよびBackendサーバーのパフォーマンスを分析します。
- **AWS WAF (Web Application Firewall)**: CloudFrontの保護を行います。各環境のCloudFrontに対するセキュリティ対策を提供します。
- **Elastic Container Registry (ECR)**: Dockerイメージを保存します。
- **Amazon Athena**: クエリサービス。ログデータをSQLでクエリし、分析を行います。
- **Amazon QuickSight**: データの可視化を行います。
- **AWS Glue**: データカタログとETLサービス。ログデータの管理を行います。
- **Amazon S3**: オブジェクトストレージサービス。ログやデータを保存します。
- **Amazon SNS**: 通知サービス。CloudWatchアラームなどの通知を開発者に送信します。

### CI/CD Pipeline
- **GitHub**: ソースコードリポジトリ。コードの変更が発生すると、パイプラインがトリガーされます。
- **CodePipeline (CP)**: 継続的デリバリーのパイプライン。ビルド、テスト、デプロイを自動化します。
- **CodeBuild (CB)**: 継続的インテグレーションサービス。コードをビルドし、Dockerイメージを作成します。
- **CodeDeploy (CD)**: アプリケーションのデプロイを管理します。各環境（Dev, Staging, Prod）にECSタスク定義を更新してデプロイします。

### データフローの詳細
1. **ユーザーアクセス**:
   - **Webブラウザ**: CloudFrontを経由してALBに接続し、Frontendサーバーにリクエストを送信します。
   - **モバイルアプリ (Android/iOS)**: API Gatewayを経由してバックエンドにリクエストを送信します。

2. **Frontendサーバー**: API Gatewayを呼び出してバックエンドサーバーにアクセスします。
3. **Backendサーバー**: RDSデータベースに対してSQLクエリを実行し、ElastiCacheを使用してキャッシュデータを管理し、Amazon SQSを使って非同期タスクを処理します。
4. **Lambda関数**: SQSメッセージをトリガーとして非同期処理を実行します。

### ログの分析と監視
- **CloudWatch**: 各環境のログとメトリクスを収集します。
- **Athena**: S3に保存されたログデータに対してSQLクエリを実行します。
- **QuickSight**: データを可視化し、レポートを作成します。
- **SNS**: CloudWatchアラームをトリガーとして開発者に通知を送信します。

この構成により、各環境（Dev, Staging, Prod）でのセキュアでスケーラブルなアプリケーションデプロイメントが実現され、ユーザー認証、データ管理、監視および分析が効果的に行われます。

### ユーザー登録のシーケンス (モバイルアプリ経由)

```mermaid
sequenceDiagram
    participant User as User
    participant MobileApp as Mobile App
    participant Cognito as Amazon Cognito
    participant APIG as API Gateway
    participant Backend as Backend Server
    participant RDS as RDS Database

    User ->> MobileApp: Open App and Sign Up
    MobileApp ->> Cognito: Sign Up (email, password)
    Cognito -->> MobileApp: Confirmation Code Sent
    MobileApp -->> User: Display Confirmation Code Input

    User ->> MobileApp: Enter Confirmation Code
    MobileApp ->> Cognito: Confirm Sign Up (code)
    Cognito -->> MobileApp: Sign Up Confirmed

    User ->> MobileApp: Log In
    MobileApp ->> Cognito: Authenticate (email, password)
    Cognito -->> MobileApp: Access Token

    MobileApp ->> APIG: Request Protected Resource (Access Token)
    APIG ->> Cognito: Validate Token
    Cognito -->> APIG: Token Valid
    APIG ->> Backend: Forward Request
    Backend ->> RDS: Query Data
    RDS -->> Backend: Return Data
    Backend -->> APIG: Return Data
    APIG -->> MobileApp: Return Data
    MobileApp -->> User: Display Data
```

### 各ステップの説明 (モバイルアプリ経由)

1. **ユーザー登録**
   - ユーザーがモバイルアプリを開き、サインアップを開始します。
   - モバイルアプリはユーザーのメールアドレスとパスワードを使ってCognitoにサインアップリクエストを送信します。
   - Cognitoは確認コードを生成し、ユーザーのメールアドレスに送信します。
   - モバイルアプリはユーザーに確認コードの入力を求めます。

2. **サインアップの確認**
   - ユーザーが確認コードを入力します。
   - モバイルアプリは確認コードを使ってCognitoにサインアップ確認リクエストを送信します。
   - Cognitoはサインアップを確認し、モバイルアプリに応答します。

3. **ログインと認証**
   - ユーザーがモバイルアプリでログインします。
   - モバイルアプリはユーザーのメールアドレスとパスワードを使ってCognitoに認証リクエストを送信します。
   - Cognitoは認証を行い、アクセストークンを発行してモバイルアプリに返します。

4. **保護されたリソースへのアクセス**
   - モバイルアプリはアクセストークンを使ってAPI Gatewayに保護されたリソースへのリクエストを送信します。
   - API GatewayはCognitoにトークンの検証を依頼します。
   - Cognitoはトークンを検証し、その結果をAPI Gatewayに返します。
   - API Gatewayはリクエストをバックエンドサーバーに転送します。
   - バックエンドサーバーはRDSデータベースに対してデータクエリを実行します。
   - RDSデータベースはクエリ結果をバックエンドサーバーに返します。
   - バックエンドサーバーはデータをAPI Gatewayに返します。
   - API Gatewayはデータをモバイルアプリに返します。
   - モバイルアプリはユーザーにデータを表示します。

### ユーザー登録のシーケンス (Webアプリ経由)

```mermaid
sequenceDiagram
    participant User as User
    participant WebApp as Web App
    participant Cognito as Amazon Cognito
    participant APIG as API Gateway
    participant Backend as Backend Server
    participant RDS as RDS Database

    User ->> WebApp: Open Web App and Sign Up
    WebApp ->> Cognito: Sign Up (email, password)
    Cognito -->> WebApp: Confirmation Code Sent
    WebApp -->> User: Display Confirmation Code Input

    User ->> WebApp: Enter Confirmation Code
    WebApp ->> Cognito: Confirm Sign Up (code)
    Cognito -->> WebApp: Sign Up Confirmed

    User ->> WebApp: Log In
    WebApp ->> Cognito: Authenticate (email, password)
    Cognito -->> WebApp: Access Token

    WebApp ->> APIG: Request Protected Resource (Access Token)
    APIG ->> Cognito: Validate Token
    Cognito -->> APIG: Token Valid
    APIG ->> Backend: Forward Request
    Backend ->> RDS: Query Data
    RDS -->> Backend: Return Data
    Backend -->> APIG: Return Data
    APIG -->> WebApp: Return Data
    WebApp -->> User: Display Data
```

### 各ステップの説明 (Webアプリ経由)

1. **ユーザー登録**
   - ユーザーがWebアプリを開き、サインアップを開始します。
   - Webアプリはユーザーのメールアドレスとパスワードを使ってCognitoにサインアップリクエストを送信します。
   - Cognitoは確認コードを生成し、ユーザーのメールアドレスに送信します。
   - Webアプリはユーザーに確認コードの入力を求めます。

2. **サインアップの確認**
   - ユーザーが確認コードを入力します。
   - Webアプリは確認コードを使ってCognitoにサインアップ確認リクエストを送信します。
   - Cognitoはサインアップを確認し、Webアプリに応答します。

3. **ログインと認証**
   - ユーザーがWebアプリでログインします。
   - Webアプリはユーザーのメールアドレスとパスワードを使ってCognitoに認証リクエストを送信します。
   - Cognitoは認証を行い、アクセストークンを発行してWebアプリに返します。

4. **保護されたリソースへのアクセス**
   - Webアプリはアクセストークンを使ってAPI Gatewayに保護されたリソースへのリクエストを送信します。
   - API GatewayはCognitoにトークンの検証を依頼します。
   - Cognitoはトークンを検証し、その結果をAPI Gatewayに返します。
   - API Gatewayはリクエストをバックエンドサーバーに転送します。
   - バックエンドサーバーはRDSデータベースに対してデータクエリを実行します。
   - RDSデータベースはクエリ結果をバックエンドサーバーに返します。
   - バックエンドサーバーはデータをAPI Gatewayに返します。
   - API GatewayはデータをWebアプリに返します。
   - Webアプリはユーザーにデータを表示します。