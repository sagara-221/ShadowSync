# インターフェース不整合分析レポート

**作成日**: 2026-05-10  
**優先度**: 🚨 最高（ブロッカー）  
**ステータス**: 重大な不整合を発見  

---

## エグゼクティブサマリー

親AI-DLCが定義した`data-accumulation-interface.md`と、data-accumulationサブシステムの`requirements.md`の間に、**3つの重大なインターフェース不整合**が発見されました。これらは実装時にロガーツールとの通信が失敗する原因となります。

### 発見された不整合

1. 🚨 **osapiトピック粒度の差異**（重大）
2. 🚨 **Chromeトピック名の差異**（重大）
3. 🚨 **ペイロードフィールド名の差異**（重大）

---

## 不整合の詳細分析

### 不整合1: osapiトピック粒度の差異

#### 親AI-DLC定義（data-accumulation-interface.md）

```
単一トピック:
shadowsync/logs/{user_id}/{device_id}/osapi
```

**意図**: osapiからの全てのイベントを単一トピックで送信

#### data-accumulation実装（requirements.md）

```
3つのトピックに分割:
shadowsync/logs/{user_id}/{device_id}/window
shadowsync/logs/{user_id}/{device_id}/audio
shadowsync/logs/{user_id}/{device_id}/snapshot
```

**意図**: イベントタイプごとに異なるトピックで送信

#### 影響分析

| 項目 | 影響度 | 詳細 |
|------|--------|------|
| **ロガーツール実装** | 🚨 高 | osapiは親定義に従って`/osapi`トピックに送信 |
| **IoT Rules** | 🚨 高 | data-accumulationは`/window`, `/audio`, `/snapshot`を期待 |
| **メッセージルーティング** | 🚨 高 | メッセージが届かない（トピック不一致） |
| **実装工数** | 中 | ロガーツール側の変更が必要 |

#### 根本原因

**設計思想の違い**:
- **親定義**: シンプルさ重視、単一トピックでイベントタイプは`event_type`フィールドで識別
- **data-accumulation**: ルーティング効率重視、トピックレベルでイベントタイプを分離

#### 推奨解決策

**オプションA: 親定義に従う（推奨）**

```
トピック: shadowsync/logs/{user_id}/{device_id}/osapi

ペイロード:
{
  "user_id": "user-123",
  "device_id": "device-123",
  "timestamp": "2026-05-10T12:00:00Z",
  "logger_type": "osapi",
  "event_type": "window_changed | audio_session_changed | periodic_snapshot",
  "data": { ... }
}

IoT Rule:
SELECT *, topic(5) as topic_suffix
FROM 'shadowsync/logs/+/+/osapi'
WHERE topic_suffix = 'osapi'
```

**メリット**:
- ✅ 親定義と完全一致
- ✅ ロガーツール実装がシンプル
- ✅ トピック数が少ない（管理が容易）
- ✅ 将来的なイベントタイプ追加が容易

**デメリット**:
- ⚠️ IoT RuleでWHERE句による振り分けが必要
- ⚠️ 若干のルーティングオーバーヘッド

**オプションB: data-accumulation定義に従う**

```
トピック:
- shadowsync/logs/{user_id}/{device_id}/window
- shadowsync/logs/{user_id}/{device_id}/audio
- shadowsync/logs/{user_id}/{device_id}/snapshot

IoT Rule:
SELECT *, topic(5) as event_type
FROM 'shadowsync/logs/+/+/+'
WHERE topic(5) IN ('window', 'audio', 'snapshot')
```

**メリット**:
- ✅ トピックレベルでルーティング（効率的）
- ✅ IoT Ruleがシンプル

**デメリット**:
- ❌ 親定義と不一致（ロガーツール変更必要）
- ❌ トピック数が増加（管理が複雑）
- ❌ 将来的なイベントタイプ追加時にトピック追加が必要

**推奨**: **オプションA（親定義に従う）**

**理由**:
1. 親AI-DLCが全体アーキテクチャを統括
2. ロガーツールは親定義に従って実装済みの可能性
3. シンプルさと拡張性のバランスが良い

---

### 不整合2: Chromeトピック名の差異

#### 親AI-DLC定義（data-accumulation-interface.md）

```
トピック: shadowsync/logs/{user_id}/{device_id}/chrome
logger_type: "chrome"
```

#### data-accumulation実装（requirements.md）

```
トピック: shadowsync/logs/{user_id}/{device_id}/browser
logger_type: "browser"
```

#### 影響分析

| 項目 | 影響度 | 詳細 |
|------|--------|------|
| **ロガーツール実装** | 🚨 高 | Chrome拡張は`/chrome`トピックに送信 |
| **IoT Rules** | 🚨 高 | data-accumulationは`/browser`を期待 |
| **メッセージルーティング** | 🚨 高 | メッセージが届かない（トピック不一致） |
| **DynamoDBスキーマ** | 中 | `logger_type: "browser"`と定義済み |

#### 根本原因

**命名の不一致**:
- **親定義**: ツール名を使用（`chrome`）
- **data-accumulation**: 機能名を使用（`browser`）

#### 推奨解決策

**オプションA: 親定義に従う（推奨）**

```
トピック: shadowsync/logs/{user_id}/{device_id}/chrome
logger_type: "chrome"

DynamoDBスキーマ:
logger_type: "chrome" (not "browser")
```

**メリット**:
- ✅ 親定義と完全一致
- ✅ ツール名が明確（Chrome拡張であることが分かる）
- ✅ 将来的に他ブラウザ拡張（Firefox, Edge等）を追加しやすい

**デメリット**:
- ⚠️ DynamoDBスキーマの変更が必要

**オプションB: data-accumulation定義に従う**

```
トピック: shadowsync/logs/{user_id}/{device_id}/browser
logger_type: "browser"
```

**メリット**:
- ✅ 汎用的な命名（ブラウザ全般）

**デメリット**:
- ❌ 親定義と不一致（Chrome拡張変更必要）
- ❌ 将来的に他ブラウザ拡張を追加する際に区別できない

**推奨**: **オプションA（親定義に従う）**

**理由**:
1. 親AI-DLCが全体アーキテクチャを統括
2. ツール名の方が明確で拡張性が高い
3. Chrome拡張は親定義に従って実装済みの可能性

---

### 不整合3: ペイロードフィールド名の差異

#### 親AI-DLC定義（data-accumulation-interface.md）

```json
{
  "user_id": "usr_123456",
  "device_id": "PC-001",
  "timestamp": "2026-05-08T23:30:00Z",
  "logger_type": "osapi | chrome | screenshot",
  "event_type": "window_changed | page_visited | periodic_ss",
  "data": {
    // 各ロガー固有のデータ
  }
}
```

**フィールド名**: `data`

#### data-accumulation実装（requirements.md）

**DynamoDBスキーマ**:
```
| activity_data | Map | ✓ | イベント固有のデータ | {...} |
```

**フィールド名**: `activity_data`

#### 影響分析

| 項目 | 影響度 | 詳細 |
|------|--------|------|
| **ロガーツール実装** | 🚨 高 | ロガーツールは`data`フィールドで送信 |
| **Lambda処理** | 🚨 高 | Lambdaは`activity_data`を期待 |
| **DynamoDBスキーマ** | 🚨 高 | `activity_data`と定義済み |
| **データ変換** | 中 | Lambda内でフィールド名変換が必要 |

#### 根本原因

**命名の不一致**:
- **親定義**: シンプルな命名（`data`）
- **data-accumulation**: 明示的な命名（`activity_data`）

#### 推奨解決策

**オプションA: 親定義に従う（推奨）**

```json
MQTTペイロード:
{
  "data": { ... }
}

DynamoDBスキーマ:
{
  "data": { ... }  // activity_data → data に変更
}
```

**メリット**:
- ✅ 親定義と完全一致
- ✅ シンプルな命名
- ✅ ロガーツールの変更不要

**デメリット**:
- ⚠️ DynamoDBスキーマの変更が必要
- ⚠️ `data`は汎用的すぎる可能性

**オプションB: Lambda内でフィールド名変換**

```python
# Lambda内で変換
mqtt_payload = {
  "data": { ... }
}

dynamodb_item = {
  "activity_data": mqtt_payload["data"]  # data → activity_data
}
```

**メリット**:
- ✅ 親定義とDynamoDBスキーマの両方を維持
- ✅ ロガーツールの変更不要

**デメリット**:
- ⚠️ Lambda内で変換処理が必要
- ⚠️ 処理の複雑度が増加

**オプションC: data-accumulation定義に従う**

```json
MQTTペイロード:
{
  "activity_data": { ... }  // data → activity_data
}
```

**メリット**:
- ✅ DynamoDBスキーマと一致
- ✅ 明示的な命名

**デメリット**:
- ❌ 親定義と不一致（ロガーツール変更必要）

**推奨**: **オプションB（Lambda内でフィールド名変換）**

**理由**:
1. 親定義を尊重（ロガーツール変更不要）
2. DynamoDBスキーマの明示的な命名を維持
3. Lambda内の変換は軽微な処理

---

## 統合的な推奨解決策

### 推奨アプローチ: 親定義を優先、Lambda内で吸収

```
【原則】
1. MQTTインターフェース: 親AI-DLC定義に完全準拠
2. DynamoDB内部スキーマ: 必要に応じて変換
3. Lambda: インターフェース変換レイヤーとして機能
```

### 具体的な修正内容

#### 1. トピック定義の修正

**修正前（data-accumulation）**:
```
shadowsync/logs/{user_id}/{device_id}/browser
shadowsync/logs/{user_id}/{device_id}/window
shadowsync/logs/{user_id}/{device_id}/audio
shadowsync/logs/{user_id}/{device_id}/snapshot
shadowsync/logs/{user_id}/{device_id}/screenshot
```

**修正後（親定義準拠）**:
```
shadowsync/logs/{user_id}/{device_id}/chrome
shadowsync/logs/{user_id}/{device_id}/osapi
shadowsync/logs/{user_id}/{device_id}/screenshot
```

#### 2. logger_typeの修正

**修正前（data-accumulation）**:
```
logger_type: "browser" | "osapi" | "screenshot"
```

**修正後（親定義準拠）**:
```
logger_type: "chrome" | "osapi" | "screenshot"
```

#### 3. ペイロードフィールド名の処理

**MQTTペイロード（親定義準拠）**:
```json
{
  "user_id": "user-123",
  "device_id": "device-123",
  "timestamp": "2026-05-10T12:00:00Z",
  "logger_type": "chrome",
  "event_type": "page_navigation",
  "data": {
    "url": "https://github.com/...",
    "title": "GitHub"
  }
}
```

**Lambda変換処理**:
```python
def transform_mqtt_to_dynamodb(mqtt_payload):
    """
    MQTTペイロード（親定義）をDynamoDBスキーマに変換
    """
    return {
        "user_id": mqtt_payload["user_id"],
        "timestamp_event_id": f"{mqtt_payload['timestamp']}#{uuid.uuid4()}",
        "timestamp": mqtt_payload["timestamp"],
        "device_id": mqtt_payload["device_id"],
        "logger_type": mqtt_payload["logger_type"],
        "event_type": mqtt_payload["event_type"],
        "activity_data": mqtt_payload["data"]  # data → activity_data
    }
```

**DynamoDBスキーマ（変更なし）**:
```
activity_data: Map  // 内部的には明示的な命名を維持
```

---

## 修正が必要なドキュメント

### 1. requirements.md

**修正箇所**:

#### Section: FR-1.1 IoT Coreメッセージ受信

**修正前**:
```
- Chrome拡張ロガー（chrome-extension）:
  - トピック: shadowsync/logs/{user_id}/{device_id}/browser
  
- OS APIロガー（osapi）:
  - トピック: shadowsync/logs/{user_id}/{device_id}/{window|audio|snapshot}
```

**修正後**:
```
- Chrome拡張ロガー（chrome-extension）:
  - トピック: shadowsync/logs/{user_id}/{device_id}/chrome
  
- OS APIロガー（osapi）:
  - トピック: shadowsync/logs/{user_id}/{device_id}/osapi
```

#### Section: FR-3.3 共通スキーマ定義

**修正前**:
```
logger_type: "browser" | "osapi" | "screenshot"
```

**修正後**:
```
logger_type: "chrome" | "osapi" | "screenshot"
```

#### Section: 5.1 IoT Coreトピック

**全てのトピック定義を親定義に合わせて修正**

### 2. unit-of-work.md

**修正箇所**: トピック定義の全ての参照

### 3. unit-of-work-dependency.md

**修正箇所**: トピック定義の全ての参照

### 4. 新規ドキュメント作成

**interface-mapping.md**（推奨）:
```markdown
# インターフェースマッピング定義

## MQTTペイロード → DynamoDBスキーマ

| MQTTフィールド | DynamoDBフィールド | 変換 |
|--------------|------------------|------|
| data | activity_data | フィールド名変換 |
| logger_type: "chrome" | logger_type: "chrome" | そのまま |
| logger_type: "osapi" | logger_type: "osapi" | そのまま |
```

---

## 影響範囲の評価

### 影響を受けるコンポーネント

| コンポーネント | 影響度 | 修正内容 |
|--------------|--------|---------|
| **requirements.md** | 🚨 高 | トピック定義、logger_type修正 |
| **unit-of-work.md** | 🚨 高 | トピック定義修正 |
| **IoT Rules** | 🚨 高 | トピックパターン修正 |
| **Lambda Router** | 中 | フィールド名変換追加 |
| **DynamoDBスキーマ** | 低 | 変更なし（activity_data維持） |
| **ロガーツール** | ✅ なし | 親定義に従っているため変更不要 |

### 実装工数の見積もり

| タスク | 見積もり | 優先度 |
|-------|---------|--------|
| ドキュメント修正 | 2-3時間 | 🚨 最高 |
| Lambda変換ロジック実装 | 1-2時間 | 高 |
| IoT Rules修正 | 1時間 | 高 |
| テスト | 2-3時間 | 高 |
| **合計** | **6-9時間** | - |

---

## リスク評価

### 修正しない場合のリスク

| リスク | 影響度 | 発生確率 | 結果 |
|-------|--------|---------|------|
| ロガーツールからのメッセージが届かない | 🚨 致命的 | 100% | システム全体が機能しない |
| データが正しく保存されない | 🚨 致命的 | 100% | データ損失 |
| 統合テスト失敗 | 🚨 高 | 100% | リリース不可 |

### 修正した場合のリスク

| リスク | 影響度 | 発生確率 | 対策 |
|-------|--------|---------|------|
| Lambda変換処理のバグ | 中 | 低 | ユニットテスト充実 |
| パフォーマンス劣化 | 低 | 低 | 軽微な変換処理のみ |

---

## 推奨アクション

### Immediate Actions（即座に実行）

1. **ドキュメント修正**
   - [ ] requirements.mdのトピック定義修正
   - [ ] requirements.mdのlogger_type修正
   - [ ] unit-of-work.mdのトピック定義修正
   - [ ] interface-mapping.md作成

2. **設計変更の承認**
   - [ ] 親AI-DLCに修正内容を報告
   - [ ] 修正方針の承認取得

3. **実装計画の更新**
   - [ ] Lambda変換ロジックの設計
   - [ ] IoT Rules定義の更新
   - [ ] テスト計画の更新

### Before Construction Phase

4. **詳細設計**
   - [ ] Lambda変換ロジックの詳細設計
   - [ ] エラーハンドリング設計
   - [ ] ログ出力設計

5. **テスト計画**
   - [ ] ユニットテストケース作成
   - [ ] 統合テストケース作成
   - [ ] E2Eテストケース作成

---

## 結論

親AI-DLCのインターフェース定義とdata-accumulationの実装仕様の間に、**3つの重大な不整合**が発見されました。これらは実装時にシステムが機能しない原因となります。

### 推奨解決策

1. **MQTTインターフェース**: 親AI-DLC定義に完全準拠
2. **DynamoDB内部スキーマ**: 必要に応じてLambda内で変換
3. **修正工数**: 6-9時間（ドキュメント修正含む）

### 次のステップ

1. 本レポートを親AI-DLCに提出
2. 修正方針の承認取得
3. ドキュメント修正の実施
4. Constructionフェーズ開始

---

**作成者**: data-accumulation AI-DLC  
**承認必要**: はい（親AI-DLC）  
**優先度**: 🚨 最高（ブロッカー）  
**ステータス**: 修正待ち
