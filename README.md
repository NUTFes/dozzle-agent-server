# NUTFes Dozzle Agent Server

NUTFes内の複数プロジェクトのコンテナログを、[Dozzle Agent](http://dozzle.dev/guide/agent) 機能を用いて集約し、Cloudflare Accessでセキュアに一元管理するためのサーバー基盤リポジトリです。

## 構成概要

各種プロダクトのデプロイ先は3台のProxmoxノード上のLXCを想定しています。本サーバー（ダッシュボード）は `pve01` ノードに専用のLXCを立てて稼働させています。

- **Dozzle Server（本リポジトリ）稼働先:** `pve01` ノード内の **LXC Container 100 (dozzle)**
- **連携対象のProxmoxノード一覧:**
  - `pve01`: [https://proxmox-pve01.nutmeg.cloud](https://proxmox-pve01.nutmeg.cloud)
  - `pve02`: [https://proxmox-pve02.nutmeg.cloud](https://proxmox-pve02.nutmeg.cloud)
  - `pve03`: [https://proxmox-pve03.nutmeg.cloud](https://proxmox-pve03.nutmeg.cloud)

各プロジェクトのLXC内にDozzle Agentを配置し、`pve01` (CT 100) のDozzle Serverから各Agentのポート(`7007`)へ接続してログを収集するアーキテクチャとなります。

---

## 導入方法（各プロジェクト開発者向け）

自分の担当しているプロジェクトのログをこの中央ダッシュボードへ出力するには、プロジェクトの `docker-compose.yml` に Agent コンテナの定義を追記するだけです。Dozzle Agentは軽量に設計されており、安全にログを転送します。

### 1. プロジェクトへのAgentの追加手順（コピペで完了）

既存の `docker-compose.yml` に以下を **そのままコピペして追加** してください。
`[YOUR_PROJECT_NAME]` の部分だけ、ご自身のプロジェクトやコンテナ名に合わせて変更してください。

```yaml
services:
  # --- ここから下を追記 ---

  dozzle-agent:
    image: amir20/dozzle:latest
    command: agent
    container_name: dozzle-agent-[YOUR_PROJECT_NAME]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    ports:
      - "7007:7007"
    environment:
      - DOZZLE_HOSTNAME=[YOUR_PROJECT_NAME]
    restart: always
    healthcheck:
      test: ["CMD", "/dozzle", "healthcheck"]
      interval: 3s
      timeout: 3s
      retries: 5

# --- 追記ここまで ---
```

---

## 運用・構築手順（インフラ管理者向け）

### リポジトリ構成イメージ

```mermaid
flowchart LR
    subgraph pve01 Node
        subgraph LXC Container 100 (This Repo)
            C[Cloudflared Tunnel] -->|http| D[Dozzle API Server]
        end
    end

    subgraph Proxmox Nodes (pve01, pve02, pve03)
        subgraph App LXCs
            A1[Project A Agent\nport:7007]
            A2[Project B Agent\nport:7007]
            A3[Project C Agent\nport:7007]
        end
    end

    User -->|Cloudflare Access Auth| C
    D -->|Connect to Agent| A1
    D -->|Connect to Agent| A2
    D -->|Connect to Agent| A3
```

### 1. サーバー（LXC 100）への配置とセットアップ

Proxmox `pve01` 上のコンテナID `100` (dozzle) へアクセスし、本リポジトリをcloneします。

```bash
git clone https://github.com/NUTFes/dozzle-agent-server.git
cd dozzle-agent-server
cp .env.example .env
```

### 2. 環境変数の設定

`.env` ファイルを開き、環境変数を設定します。

- **`DOZZLE_REMOTE_AGENT`**: ログを収集したい各プロジェクト(Agent)のエンドポイントをカンマ区切りで列挙します。各LXCのIPアドレスを指定します。
  _フォーマット:_ `[ホストIPアドレス]:7007`
  _例:_ `192.168.10.2:7007,192.168.10.3:7007,192.168.10.4:7007`
- **`TUNNEL_TOKEN`**: Cloudflare Zero Trust ダッシュボードで払い出された Cloudflared Tunnel のトークン

### 3. モニタリングの起動

以下のコマンドでDozzle ServerとCloudflaredトンネルのコンテナを起動します。

```bash
docker compose up -d
```

設定したCloudflare TunnelのPublic Hostnameにアクセスすると、認証通過後にすべてのプロジェクト(Agent)のログが一元化されたダッシュボードで閲覧可能になります。新しくAgentが追加された場合は、`.env`の `DOZZLE_REMOTE_AGENT` にIPを追記し、このサーバーの `docker compose up -d` を再実行して反映させます。
