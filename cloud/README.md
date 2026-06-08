# Cloud / Backend

エッジから受信した署名済みレコードを検証・保管・集計し、外部アンカリングと検証者向けインターフェースを提供するバックエンド。

---

## アーキテクチャ

```mermaid
flowchart TB
    subgraph EDGE["EDGE（デバイス）"]
        DEV["ESP32-S3\n署名済みレコード"]
    end

    subgraph SORACOM["SORACOM 基盤"]
        FUNNEL["Funnel\nデータ転送"]
        HARVEST["Harvest\nストレージ"]
        KRYPTON["Krypton\n鍵プロビジョニング"]
        ENDORSE["Endorse\nSIM 認証"]
        KRYPTON --> ENDORSE
    end

    subgraph BACKEND["Backend"]
        VERIFY["署名検証サービス\nECDSA + 証明書連鎖"]
        STORE["追記専用ストア\n不可変ログ"]
        CALC["較正・集計エンジン\nOTS / LTS 派生"]
        API["検証 API\n監査人向け"]
    end

    subgraph ANCHOR["外部アンカリング"]
        TSA["認定タイムスタンプ局\nRFC 3161（総務大臣認定）"]
        PUBLOG["公開ログ\nMerkle ルート掲示"]
    end

    DEV -->|LTE-M| FUNNEL
    FUNNEL --> HARVEST
    HARVEST --> VERIFY
    VERIFY --> STORE
    STORE --> CALC
    STORE -->|定期 Merkle ルート| TSA
    TSA --> PUBLOG
    CALC --> API
    STORE --> API
```

---

## 鍵管理・デバイス認証（SORACOM Krypton）

```mermaid
sequenceDiagram
    participant DEV as デバイス + IoT SIM
    participant SOR as SORACOM（Endorse + Krypton）
    participant CA as 自社 CA

    DEV->>SOR: ① SIM 認証で初期設定要求
    SOR->>CA: ② CA 配下でデバイス証明書発行リクエスト
    CA-->>SOR: 証明書発行
    SOR-->>DEV: ③ デバイス証明書・鍵を返却 → SE に格納
    Note over DEV: 以後、各レコードをこの鍵で ECDSA 署名
    Note over CA: トラストアンカー（CA ルート）は公開可能
```

- SIM をハードウェアの root of trust とし、デバイス固有の鍵・証明書をブート時に動的プロビジョニング
- 秘密情報をファームウェアに焼き込む必要がなく、全デバイスで共通イメージを使用可能
- 自社登録 CA のルートは公開でき、第三者がデバイス証明書の連鎖を検証可能

---

## 外部アンカリング

```mermaid
flowchart LR
    subgraph LOG["連鎖ログ"]
        R1["seq N\ncount: ...\nprev_hash → \nsig ✓"]
        R2["seq N+1\n..."]
        R3["seq N+2\n..."]
    end

    MERKLE["Merkle ルート\n集約ハッシュ"]
    TSA2["認定タイムスタンプ\nRFC 3161\n総務大臣認定"]
    PUBLOG2["公開ログ\n第三者が随時照合"]

    LOG -->|定期集約| MERKLE
    MERKLE --> TSA2
    MERKLE --> PUBLOG2
```

一定期間ごとに連鎖ログの Merkle ルートを外部 TSA へ提出し RFC 3161 タイムスタンプを取得。「このレコード群が、その時刻に存在し、後追いで捏造されていない」ことを独立に証明する。

**認定タイムスタンプ事業者（例）：**  
セイコーソリューションズ / アマノ / 三菱電機インフォメーションネットワーク / GMO グローバルサイン

---

## データフロー

```mermaid
flowchart TB
    RAW["生イベント\n（署名済み・不可変）"]
    CALIB["較正係数\n（calib_ver 管理）"]
    OTS["OTS\n（Opportunity to See）"]
    LTS["LTS / VAC\n（視認確率で補正）"]
    IMP["インプレッション乗数\n（DOOH 取引指標）"]

    RAW -->|検知補正| OTS
    CALIB --> OTS
    OTS -->|視認確率モデル| LTS
    LTS -->|時間帯テーブル| IMP
```

- 生イベントと派生集計（OTS/LTS）は別テーブルに分離
- 較正バージョン（`calib_ver`）で参照し、検証者が「生データ + 公開された較正手順」から最終値を再計算可能
- 「数字を後でいじった」疑義を構造的に排除

---

## 第三者検証フロー

```mermaid
flowchart LR
    V1["① 署名検証\n公開 CA に連鎖"]
    V2["② 連鎖検証\nseq / prev_hash"]
    V3["③ FW 照合\nfw_hash"]
    V4["④ 時刻アンカー\nTSA / 公開ログ"]
    V5["⑤ 集計再導出\n生イベント + calib_ver\n→ OTS/LTS 再計算\n→ 公表値と一致確認"]

    V1 --> V2 --> V3 --> V4 --> V5
```

監査人は **鍵と公開された方法論だけで**、数字の出所・不改変・最終集計を独立に確認できる。プライバシー（PII なし）と検証可能性（生データ全開示可）が相互に補強し合う。

---

## 技術スタック（予定）

| コンポーネント | 技術候補 |
|--------------|---------|
| 受信・転送 | SORACOM Funnel → Harvest |
| 署名検証 | Node.js / Go（ECDSA P-256） |
| 不可変ストア | AWS DynamoDB（append-only）/ Timestream |
| 集計エンジン | Python / Spark（較正・OTS/LTS 計算） |
| 外部アンカリング | RFC 3161 TSA クライアント |
| 検証 API | REST / GraphQL |

---

## 次工程

- [ ] SORACOM Funnel → バックエンド受信の試験（Phase 2）
- [ ] 署名検証サービスの実装
- [ ] 追記専用ストアの設計・構築
- [ ] 較正・集計エンジンの実装（Phase 2）
- [ ] RFC 3161 TSA アンカリングの統合（Phase 2）
- [ ] 検証 API の設計・公開（Phase 3）
