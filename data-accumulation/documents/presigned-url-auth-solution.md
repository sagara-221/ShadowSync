# Presigned URL API 認証方式の解決策

**作成日**: 2026-05-10  
**ステータス**: 提案  
**優先度**: 🚨 最高（ブロッカー）

---

## 問題の要約

現在の設計では、Presigned URL APIが**Cognito User Pool認証**を使用していますが、ss-tool（ローカルアプリ）は**X.509証明書認証**のみ対応しています。この不整合により、ss-toolが画像アップロード用のPresigned URLを取得できません。

---

## 推奨解決策: Lambda Function URL + カスタム認証ロジック

### アーキテクチャ概要

```
┌─────────────────────────────────────────────────────────────┐
│                    Presigned URL取得フロー                    │
└─────────────────────────────────────────────────────────────┘

【パターン1: ss-tool (X.509証明書)】
┌──────────┐     MQTTS      ┌──────────┐     Invoke    ┌─────────────┐
│ ss-tool  │───────────────>│ IoT Core │──────────────>│ Lambda      │
│ (X.509)  │  特殊トピック   │          │   IoT Rule    │ Presigned   │
└──────────┘                └──────────┘               │ URL Handler │
                                                        └─────────────┘
                                                              │
                                                              v
                                                        ┌─────────────┐
                                                        │ S3 Presigned│
                                                        │ URL生成     │
                                                        └─────────────┘
                                                              │
                                                              v
                                                        ┌─────────────┐
                                                        │ IoT Core    │
                                                        │ レスポンス   │
                                                        └─────────────┘
                                                              │
                                                              v
                                                        ┌──────────┐
                                                        │ ss-tool  │
                                                        │ 受信     │
                                                        └──────────┘

【パターン2: Chrome拡張 (Cognito)】
┌──────────┐     HTTPS      ┌─────────────────┐     Invoke    ┌─────────────┐
│ Chrome   │───────────────>│ Lambda Function │──────────────>│ Lambda      │
│ 拡張     │  Bearer Token   │ URL (IAM認証)   │               │ Presigned   │
│ (Cognito)│                 └─────────────────┘               │ URL Handler │
└──────────┘                                                   └─────────────┘
                                                                      │
                                                                      v
                                                                ┌─────────────┐
                                                                │ S3 Presigned│
                                                                │ URL生成     │
                                                                └─────────────┘
                                                                      │
                                                                      v
                                                                ┌─────────────┐
                                                                │ HTTPS       │
                                                                │ レスポンス   │
                                                                └─────────────┘
```

---

## 実装方針

### オプションA: IoT Core Request/Response パターン（推奨）

**ss-tool用の専用フロー**を構築し、Chrome拡張とは完全に分離します。

#### ss-toolのフロー

1. **Presigned URL要求**
   ```
   ss-tool → IoT Core (MQTTS)
   トピック: shadowsync/presigned-url/request/{user_id}/{device_id}
   ペイロード: {
     "request_id": "uuid",
     "file_name": "screenshot.webp",
     "content_type": "image/webp",
     "file_size": 1048576
   }
   ```

2. **IoT Rule → Lambda起動**
   ```sql
   SELECT *, topic(3) as user_id, topic(4) as device_id
   FROM 'shadowsync/presigned-url/request/+/+'
   ```

3. **Lambda処理**
   - user_idとdevice_idを検証
   - S3 Presigned URL生成
   - レスポンストピックにパブリッシュ

4. **Presigned URL受信**
   ```
   IoT Core → ss-tool (MQTTS)
   トピック: shadowsync/presigned-url/response/{user_id}/{device_id}
   ペイロード: {
     "request_id": "uuid",
     "presigned_url": "https://...",
     "s3_key": "raw/screenshots/...",
     "expires_at": "2026-05-10T13:00:00Z"
   }
   ```

#### Chrome拡張のフロー

1. **Lambda Function URL (HTTPS)**
   ```
   POST https://<lambda-function-url>
   Authorization: Bearer <cognito-token>
   
   Body: {
     "file_name": "screenshot.webp",
     "content_type": "image/webp",
     "file_size": 1048576
   }
   ```

2. **Lambda処理**
   - Cognitoトークン検証
   - user_id抽出
   - S3 Presigned URL生成
   - HTTPレスポンス

3. **レスポンス**
   ```json
   {
     "presigned_url": "https://...",
     "s3_key": "raw/screenshots/...",
     "expires_at": "2026-05-10T13:00:00Z"
   }
   ```

---

## 実装詳細

### 1. CloudFormation: IoT Rule（ss-tool用）

```yaml
# iac/ingestion-processing.yaml

PresignedUrlRequestRule:
  Type: AWS::IoT::TopicRule
  Properties:
    RuleName: PresignedUrlRequestRule
    TopicRulePayload:
      Sql: >-
        SELECT *, topic(3) as user_id, topic(4) as device_id
        FROM 'shadowsync/presigned-url/request/+/+'
      Actions:
        - Lambda:
            FunctionArn: !GetAtt PresignedUrlLambda.Arn
      RuleDisabled: false

PresignedUrlLambdaInvokePermission:
  Type: AWS::Lambda::Permission
  Properties:
    FunctionName: !Ref PresignedUrlLambda
    Action: lambda:InvokeFunction
    Principal: iot.amazonaws.com
    SourceArn: !GetAtt PresignedUrlRequestRule.Arn
```

### 2. CloudFormation: Lambda Function URL（Chrome拡張用）

```yaml
# iac/api.yaml

PresignedUrlLambda:
  Type: AWS::Lambda::Function
  Properties:
    FunctionName: shadowsync-presigned-url
    Runtime: python3.12
    Handler: handler.lambda_handler
    Code:
      S3Bucket: !Ref DeploymentBucket
      S3Key: lambda/presigned-url.zip
    Environment:
      Variables:
        S3_BUCKET_NAME: !Ref ScreenshotBucket
        PRESIGNED_URL_EXPIRATION: "3600"  # 1時間
        COGNITO_USER_POOL_ID: !Ref CognitoUserPool
    Role: !GetAtt PresignedUrlLambdaRole.Arn

PresignedUrlLambdaUrl:
  Type: AWS::Lambda::Url
  Properties:
    TargetFunctionArn: !Ref PresignedUrlLambda
    AuthType: AWS_IAM  # IAM認証
    Cors:
      AllowOrigins:
        - "*"
      AllowMethods:
        - POST
      AllowHeaders:
        - Content-Type
        - Authorization

PresignedUrlLambdaRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: "2012-10-17"
      Statement:
        - Effect: Allow
          Principal:
            Service: lambda.amazonaws.com
          Action: sts:AssumeRole
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
    Policies:
      - PolicyName: S3PresignedUrlPolicy
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Effect: Allow
              Action:
                - s3:PutObject
              Resource: !Sub "${ScreenshotBucket.Arn}/raw/screenshots/*"
      - PolicyName: IoTPublishPolicy
        PolicyDocument:
          Version: "2012-10-17"
          Statement:
            - Effect: Allow
              Action:
                - iot:Publish
              Resource: !Sub "arn:aws:iot:${AWS::Region}:${AWS::AccountId}:topic/shadowsync/presigned-url/response/*/*"
```

### 3. Lambda関数実装

```python
# lambda/presigned-url/handler.py

import json
import os
import boto3
import uuid
from datetime import datetime, timedelta
from typing import Dict, Any, Optional

s3_client = boto3.client('s3')
iot_client = boto3.client('iot-data')

S3_BUCKET = os.environ['S3_BUCKET_NAME']
EXPIRATION = int(os.environ.get('PRESIGNED_URL_EXPIRATION', '3600'))

def lambda_handler(event: Dict[str, Any], context: Any) -> Dict[str, Any]:
    """
    Presigned URL生成Lambda
    
    2つの呼び出しパターンに対応:
    1. IoT Core経由（ss-tool用）
    2. Lambda Function URL経由（Chrome拡張用）
    """
    
    # 呼び出し元の識別
    if 'user_id' in event and 'device_id' in event:
        # IoT Core経由（ss-tool）
        return handle_iot_request(event)
    elif 'requestContext' in event:
        # Lambda Function URL経由（Chrome拡張）
        return handle_http_request(event)
    else:
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'Invalid request format'})
        }


def handle_iot_request(event: Dict[str, Any]) -> Dict[str, Any]:
    """
    IoT Core経由のリクエスト処理（ss-tool用）
    """
    try:
        # パラメータ抽出
        user_id = event['user_id']
        device_id = event['device_id']
        request_id = event.get('request_id', str(uuid.uuid4()))
        file_name = event.get('file_name', 'screenshot.webp')
        content_type = event.get('content_type', 'image/webp')
        
        # Presigned URL生成
        s3_key, presigned_url, expires_at = generate_presigned_url(
            user_id, device_id, file_name, content_type
        )
        
        # レスポンスをIoT Coreにパブリッシュ
        response_topic = f"shadowsync/presigned-url/response/{user_id}/{device_id}"
        response_payload = {
            'request_id': request_id,
            'presigned_url': presigned_url,
            's3_key': s3_key,
            'expires_at': expires_at
        }
        
        iot_client.publish(
            topic=response_topic,
            qos=1,
            payload=json.dumps(response_payload)
        )
        
        print(f"Published presigned URL to {response_topic}")
        return {'statusCode': 200, 'message': 'Success'}
        
    except Exception as e:
        print(f"Error in handle_iot_request: {str(e)}")
        # エラーレスポンスもIoT Coreにパブリッシュ
        error_topic = f"shadowsync/presigned-url/response/{user_id}/{device_id}"
        error_payload = {
            'request_id': event.get('request_id'),
            'error': str(e)
        }
        iot_client.publish(
            topic=error_topic,
            qos=1,
            payload=json.dumps(error_payload)
        )
        return {'statusCode': 500, 'message': str(e)}


def handle_http_request(event: Dict[str, Any]) -> Dict[str, Any]:
    """
    Lambda Function URL経由のリクエスト処理（Chrome拡張用）
    """
    try:
        # Cognitoトークンからuser_id抽出
        user_id = extract_user_id_from_cognito(event)
        if not user_id:
            return {
                'statusCode': 401,
                'body': json.dumps({'error': 'Unauthorized'})
            }
        
        # リクエストボディ解析
        body = json.loads(event.get('body', '{}'))
        device_id = body.get('device_id', 'chrome-ext')
        file_name = body.get('file_name', 'screenshot.webp')
        content_type = body.get('content_type', 'image/webp')
        
        # Presigned URL生成
        s3_key, presigned_url, expires_at = generate_presigned_url(
            user_id, device_id, file_name, content_type
        )
        
        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'presigned_url': presigned_url,
                's3_key': s3_key,
                'expires_at': expires_at
            })
        }
        
    except Exception as e:
        print(f"Error in handle_http_request: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }


def generate_presigned_url(
    user_id: str,
    device_id: str,
    file_name: str,
    content_type: str
) -> tuple[str, str, str]:
    """
    S3 Presigned URL生成
    
    Returns:
        (s3_key, presigned_url, expires_at)
    """
    # S3キー生成
    timestamp = datetime.utcnow()
    s3_key = (
        f"raw/screenshots/{user_id}/{device_id}/"
        f"{timestamp.strftime('%Y/%m/%d/%H-%M-%S')}.webp"
    )
    
    # Presigned URL生成
    presigned_url = s3_client.generate_presigned_url(
        'put_object',
        Params={
            'Bucket': S3_BUCKET,
            'Key': s3_key,
            'ContentType': content_type
        },
        ExpiresIn=EXPIRATION
    )
    
    expires_at = (timestamp + timedelta(seconds=EXPIRATION)).isoformat() + 'Z'
    
    return s3_key, presigned_url, expires_at


def extract_user_id_from_cognito(event: Dict[str, Any]) -> Optional[str]:
    """
    Lambda Function URLのIAM認証からuser_id抽出
    
    Chrome拡張はCognito IDプールで一時認証情報を取得し、
    IAM認証でLambda Function URLを呼び出す。
    """
    try:
        # IAM認証情報から抽出
        request_context = event.get('requestContext', {})
        authorizer = request_context.get('authorizer', {})
        iam = authorizer.get('iam', {})
        
        # Cognito IDプールのIdentity IDを取得
        user_id = iam.get('userId')  # Cognito Identity ID
        
        if not user_id:
            # Authorizationヘッダーから抽出（フォールバック）
            headers = event.get('headers', {})
            auth_header = headers.get('authorization', '')
            # Cognitoトークンをデコードしてuser_idを抽出
            # （実装は省略、boto3のCognitoクライアントを使用）
            pass
        
        return user_id
        
    except Exception as e:
        print(f"Error extracting user_id: {str(e)}")
        return None
```

### 4. ss-tool側の実装例（Python）

```python
# ss-tool/presigned_url_client.py

import json
import uuid
import time
from typing import Optional
from awscrt import mqtt
from awsiot import mqtt_connection_builder

class PresignedUrlClient:
    def __init__(self, endpoint: str, cert_path: str, key_path: str, 
                 ca_path: str, client_id: str, user_id: str, device_id: str):
        self.user_id = user_id
        self.device_id = device_id
        self.pending_requests = {}
        
        # MQTT接続
        self.mqtt_connection = mqtt_connection_builder.mtls_from_path(
            endpoint=endpoint,
            cert_filepath=cert_path,
            pri_key_filepath=key_path,
            ca_filepath=ca_path,
            client_id=client_id,
            clean_session=False,
            keep_alive_secs=30
        )
        
        # 接続
        connect_future = self.mqtt_connection.connect()
        connect_future.result()
        
        # レスポンストピックをサブスクライブ
        response_topic = f"shadowsync/presigned-url/response/{user_id}/{device_id}"
        subscribe_future, _ = self.mqtt_connection.subscribe(
            topic=response_topic,
            qos=mqtt.QoS.AT_LEAST_ONCE,
            callback=self._on_response
        )
        subscribe_future.result()
    
    def request_presigned_url(self, file_name: str, content_type: str = "image/webp",
                             file_size: int = 0, timeout: int = 10) -> Optional[dict]:
        """
        Presigned URLをリクエスト
        
        Returns:
            {
                'presigned_url': str,
                's3_key': str,
                'expires_at': str
            }
        """
        request_id = str(uuid.uuid4())
        
        # リクエストペイロード
        payload = {
            'request_id': request_id,
            'file_name': file_name,
            'content_type': content_type,
            'file_size': file_size
        }
        
        # リクエストトピックにパブリッシュ
        request_topic = f"shadowsync/presigned-url/request/{self.user_id}/{self.device_id}"
        publish_future, _ = self.mqtt_connection.publish(
            topic=request_topic,
            payload=json.dumps(payload),
            qos=mqtt.QoS.AT_LEAST_ONCE
        )
        publish_future.result()
        
        # レスポンス待機
        self.pending_requests[request_id] = None
        start_time = time.time()
        
        while time.time() - start_time < timeout:
            if self.pending_requests[request_id] is not None:
                response = self.pending_requests.pop(request_id)
                return response
            time.sleep(0.1)
        
        # タイムアウト
        self.pending_requests.pop(request_id, None)
        return None
    
    def _on_response(self, topic: str, payload: bytes, **kwargs):
        """レスポンス受信コールバック"""
        try:
            response = json.loads(payload.decode('utf-8'))
            request_id = response.get('request_id')
            
            if request_id in self.pending_requests:
                self.pending_requests[request_id] = response
        except Exception as e:
            print(f"Error processing response: {e}")
    
    def disconnect(self):
        """MQTT切断"""
        disconnect_future = self.mqtt_connection.disconnect()
        disconnect_future.result()


# 使用例
if __name__ == "__main__":
    client = PresignedUrlClient(
        endpoint="your-iot-endpoint.iot.us-east-1.amazonaws.com",
        cert_path="certs/device.pem.crt",
        key_path="certs/device.pem.key",
        ca_path="certs/AmazonRootCA1.pem",
        client_id="ss-tool-001",
        user_id="user-123",
        device_id="device-123"
    )
    
    # Presigned URL取得
    result = client.request_presigned_url("screenshot.webp")
    
    if result:
        print(f"Presigned URL: {result['presigned_url']}")
        print(f"S3 Key: {result['s3_key']}")
        
        # 画像アップロード
        import requests
        with open("screenshot.webp", "rb") as f:
            response = requests.put(
                result['presigned_url'],
                data=f,
                headers={'Content-Type': 'image/webp'}
            )
            print(f"Upload status: {response.status_code}")
    
    client.disconnect()
```

### 5. Chrome拡張側の実装例（JavaScript）

```javascript
// chrome-extension/presigned-url-client.js

import { CognitoIdentityClient } from "@aws-sdk/client-cognito-identity";
import { fromCognitoIdentityPool } from "@aws-sdk/credential-provider-cognito-identity";
import { SignatureV4 } from "@aws-sdk/signature-v4";
import { Sha256 } from "@aws-crypto/sha256-js";

class PresignedUrlClient {
  constructor(identityPoolId, region, lambdaFunctionUrl) {
    this.identityPoolId = identityPoolId;
    this.region = region;
    this.lambdaFunctionUrl = lambdaFunctionUrl;
    
    // Cognito認証情報プロバイダー
    this.credentialsProvider = fromCognitoIdentityPool({
      client: new CognitoIdentityClient({ region }),
      identityPoolId: identityPoolId
    });
  }
  
  async requestPresignedUrl(fileName, contentType = "image/webp", fileSize = 0) {
    try {
      // 一時認証情報取得
      const credentials = await this.credentialsProvider();
      
      // リクエストボディ
      const body = JSON.stringify({
        device_id: "chrome-ext",
        file_name: fileName,
        content_type: contentType,
        file_size: fileSize
      });
      
      // SigV4署名
      const signer = new SignatureV4({
        credentials,
        region: this.region,
        service: "lambda",
        sha256: Sha256
      });
      
      const url = new URL(this.lambdaFunctionUrl);
      const signedRequest = await signer.sign({
        method: "POST",
        hostname: url.hostname,
        path: url.pathname,
        protocol: url.protocol,
        headers: {
          "Content-Type": "application/json",
          "host": url.hostname
        },
        body
      });
      
      // Lambda Function URL呼び出し
      const response = await fetch(this.lambdaFunctionUrl, {
        method: "POST",
        headers: signedRequest.headers,
        body
      });
      
      if (!response.ok) {
        throw new Error(`HTTP ${response.status}: ${await response.text()}`);
      }
      
      return await response.json();
      
    } catch (error) {
      console.error("Error requesting presigned URL:", error);
      throw error;
    }
  }
}

// 使用例
const client = new PresignedUrlClient(
  "us-east-1:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "us-east-1",
  "https://xxxxxxxxxx.lambda-url.us-east-1.on.aws/"
);

// Presigned URL取得
const result = await client.requestPresignedUrl("screenshot.webp");
console.log("Presigned URL:", result.presigned_url);

// 画像アップロード
const blob = await captureScreenshot();
const uploadResponse = await fetch(result.presigned_url, {
  method: "PUT",
  headers: {
    "Content-Type": "image/webp"
  },
  body: blob
});

console.log("Upload status:", uploadResponse.status);
```

---

## メリット・デメリット

### メリット

1. **完全な認証方式分離**
   - ss-tool: IoT Core（X.509証明書）
   - Chrome拡張: Lambda Function URL（Cognito + IAM）

2. **コスト最適化**
   - Lambda Function URLは無料
   - API Gateway不要

3. **シンプルな実装**
   - 各クライアントに最適な認証フロー
   - 複雑なカスタムオーソライザー不要

4. **スケーラビリティ**
   - IoT Coreの自動スケーリング
   - Lambda Function URLの自動スケーリング

### デメリット

1. **2つの異なるフロー**
   - ss-tool: Request/Response パターン（非同期）
   - Chrome拡張: HTTP API（同期）

2. **ss-toolの実装複雑度**
   - MQTTのRequest/Responseパターン実装が必要
   - タイムアウト処理が必要

3. **テストの複雑さ**
   - 2つの異なるフローをテストする必要

---

## 実装スケジュール

### Phase 1: Lambda関数実装（1-2日）
- [ ] Lambda関数コード作成
- [ ] ユニットテスト作成
- [ ] ローカルテスト

### Phase 2: CloudFormation実装（1日）
- [ ] IoT Rule定義
- [ ] Lambda Function URL定義
- [ ] IAMロール・ポリシー定義

### Phase 3: クライアント実装（2-3日）
- [ ] ss-tool側のPresignedUrlClient実装
- [ ] Chrome拡張側のPresignedUrlClient実装
- [ ] 統合テスト

### Phase 4: E2Eテスト（1-2日）
- [ ] ss-toolからのフローテスト
- [ ] Chrome拡張からのフローテスト
- [ ] エラーケーステスト

**合計見積もり**: 5-8日

---

## 代替案: API Gateway + カスタムオーソライザー

もしAPI Gatewayの機能（スロットリング、キャッシング等）が必要な場合は、以下の代替案も検討可能です。

### アーキテクチャ

```
[ss-tool (X.509)] → [API Gateway (カスタムオーソライザー)] → [Lambda]
[Chrome拡張 (Cognito)] → [API Gateway (Cognitoオーソライザー)] → [Lambda]
```

### 追加コスト

- API Gateway: $3.50/百万リクエスト
- 月間10万リクエスト想定: 約$0.35/月

### 実装複雑度

- カスタムオーソライザーLambda実装が必要
- X.509証明書検証ロジックが必要

---

## 推奨事項

**推奨**: オプションA（IoT Core Request/Response + Lambda Function URL）

**理由**:
1. コスト最適化（API Gateway不要）
2. 各クライアントに最適な認証フロー
3. 親AI-DLCの「料金がかからない、かつ構成がシンプル」要件に合致
4. 実装がシンプル

---

**作成者**: 親AI-DLC  
**承認必要**: はい  
**次のステップ**: data-accumulation AI-DLCによる実装
