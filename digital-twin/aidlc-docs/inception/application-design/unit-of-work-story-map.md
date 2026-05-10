# 作業単位ストーリーマップ: digital-twin

**プロジェクト**: ShadowSync - digital-twin  
**ステージ**: INCEPTION - 作業単位生成  
**バージョン**: 1.0  
**日付**: 2026-05-10  
**ステータス**: 承認済み

---

## 1. 目的

承認済みユーザーストーリーを作業単位へ割り当て、後続フェーズでの責務漏れを防ぐ。

---

## 2. ストーリーから作業単位への対応

| ストーリーID | タイトル | 主担当作業単位 | 関連作業単位 |
|---|---|---|---|
| DT-US-001 | デスクトップアプリで質問を開始する | DT-UOW-001 Desktop App | DT-UOW-002, DT-UOW-006 |
| DT-US-002 | Cognitoで認証して安全に利用する | DT-UOW-001 Desktop App / DT-UOW-002 Conversation API & Orchestrator | DT-UOW-006 |
| DT-US-003 | 過去の活動について自然言語で質問する | DT-UOW-002 Conversation API & Orchestrator | DT-UOW-001, DT-UOW-003, DT-UOW-004, DT-UOW-005 |
| DT-US-004 | 自分の活動ログだけをRAG検索する | DT-UOW-003 Retrieval Adapter | DT-UOW-002, DT-UOW-006 |
| DT-US-005 | 根拠付きの回答を受け取る | DT-UOW-004 Answer & Evidence | DT-UOW-003, DT-UOW-005, DT-UOW-006 |
| DT-US-006 | 回答根拠を段階的に確認する | DT-UOW-004 Answer & Evidence | DT-UOW-001, DT-UOW-003 |
| DT-US-007 | 会話履歴を保存し再利用する | DT-UOW-005 Conversation History | DT-UOW-001, DT-UOW-002, DT-UOW-006 |
| DT-US-008 | 失敗時に安全で理解可能な応答を受け取る | DT-UOW-002 Conversation API & Orchestrator | DT-UOW-001, DT-UOW-003, DT-UOW-004, DT-UOW-005, DT-UOW-006 |
| DT-US-009 | 他ユーザーデータ混入を防ぎ監査する | DT-UOW-006 Audit & Security Validation | DT-UOW-002, DT-UOW-003, DT-UOW-004, DT-UOW-005 |

---

## 3. 作業単位別ストーリー一覧

### DT-UOW-001 Desktop App

| 区分 | ストーリー |
|---|---|
| 主担当 | DT-US-001, DT-US-002 |
| 関連 | DT-US-003, DT-US-006, DT-US-007, DT-US-008 |

主なユーザー価値:
- 質問入力から回答確認までのデスクトップ体験。
- Cognitoログインと認証状態表示。
- 回答、根拠、履歴、エラーの表示。

### DT-UOW-002 Conversation API & Orchestrator

| 区分 | ストーリー |
|---|---|
| 主担当 | DT-US-003, DT-US-008 |
| 関連 | DT-US-001, DT-US-002, DT-US-004, DT-US-005, DT-US-007, DT-US-009 |

主なユーザー価値:
- 自然言語質問を受け付け、検索、回答生成、履歴保存を統括する。
- 認証済み `user_id` を確定し、クライアント指定 `user_id` を信用しない。
- 失敗を分類し、ユーザーに理解可能な応答を返す。

### DT-UOW-003 Retrieval Adapter

| 区分 | ストーリー |
|---|---|
| 主担当 | DT-US-004 |
| 関連 | DT-US-003, DT-US-005, DT-US-006, DT-US-008, DT-US-009 |

主なユーザー価値:
- 自分の活動ログだけを検索する。
- data-accumulation検索基盤の詳細を隠蔽する。
- 検索結果から根拠表示に必要なメタデータを保持する。

### DT-UOW-004 Answer & Evidence

| 区分 | ストーリー |
|---|---|
| 主担当 | DT-US-005, DT-US-006 |
| 関連 | DT-US-003, DT-US-004, DT-US-008, DT-US-009 |

主なユーザー価値:
- 活動ログに基づく回答を生成する。
- 根拠不足時に断定を避ける。
- 根拠を時刻、ソース種別、関連抜粋、参照元識別子として表示できる形にする。

### DT-UOW-005 Conversation History

| 区分 | ストーリー |
|---|---|
| 主担当 | DT-US-007 |
| 関連 | DT-US-003, DT-US-005, DT-US-008, DT-US-009 |

主なユーザー価値:
- 会話履歴を活動ログとは別に保存する。
- 後続質問の文脈として履歴を利用する。
- ユーザー別、セッション別に自分の履歴だけを取得する。

### DT-UOW-006 Audit & Security Validation

| 区分 | ストーリー |
|---|---|
| 主担当 | DT-US-009 |
| 関連 | DT-US-002, DT-US-004, DT-US-007, DT-US-008 |

主なユーザー価値:
- 他ユーザーデータ混入を防ぐ。
- 認可失敗、検索実行、回答生成、履歴保存失敗を監査可能にする。
- Security BaselineとPBT候補を各作業単位へ横断適用する。

---

## 4. 要件カバレッジ

| 要件 | 主な作業単位 |
|---|---|
| FR-1 デスクトップUI | DT-UOW-001 |
| FR-2 認証 | DT-UOW-001, DT-UOW-002, DT-UOW-006 |
| FR-3 Conversation API | DT-UOW-002 |
| FR-4 RAG検索 | DT-UOW-003 |
| FR-5 回答生成 | DT-UOW-004 |
| FR-6 根拠表示 | DT-UOW-001, DT-UOW-004 |
| FR-7 会話履歴 | DT-UOW-005 |
| FR-8 マルチテナント分離 | DT-UOW-002, DT-UOW-003, DT-UOW-005, DT-UOW-006 |
| FR-9 監査 | DT-UOW-006 |
| NFR-1 セキュリティ | DT-UOW-002, DT-UOW-003, DT-UOW-005, DT-UOW-006 |
| NFR-2 プライバシー | DT-UOW-003, DT-UOW-005, DT-UOW-006 |
| NFR-3 パフォーマンス | DT-UOW-001, DT-UOW-002, DT-UOW-003, DT-UOW-004 |
| NFR-4 信頼性 | DT-UOW-002, DT-UOW-003, DT-UOW-004, DT-UOW-005 |
| NFR-5 テスト容易性 | DT-UOW-003, DT-UOW-005, DT-UOW-006 |
| NFR-6 保守性 | 全作業単位 |

---

## 5. Security / PBT割り当て

| 観点 | 主担当 | 各作業単位での扱い |
|---|---|---|
| クライアント指定 `user_id` の無視 | DT-UOW-002 | Desktop Appは認可用 `user_id` を送らない |
| 検索時の `user_id` 分離 | DT-UOW-003 | Audit & SecurityでPBT候補化 |
| 履歴取得時の `user_id` 分離 | DT-UOW-005 | Audit & SecurityでPBT候補化 |
| 根拠への他ユーザーデータ混入防止 | DT-UOW-004 | Retrieval結果の境界を前提に検証 |
| ログサニタイズ | DT-UOW-006 | 全作業単位のログ出力条件へ反映 |
| レスポンス形式の一貫性 | DT-UOW-002 | Desktop Appと契約テスト候補 |
| シリアライズ不変条件 | DT-UOW-006 | sharedモデルのPBT候補 |

---

## 6. 完全性確認

- すべてのユーザーストーリーは少なくとも1つの主担当作業単位に割り当て済み。
- すべての機能要件は少なくとも1つの作業単位に割り当て済み。
- Security BaselineとPBT候補はDT-UOW-006を中心にしつつ、各機能作業単位の条件にも分散している。
- data-accumulationとの境界はDT-UOW-003 Retrieval Adapterに集約している。
