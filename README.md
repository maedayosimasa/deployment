# deployment/

`bim-aiagent01`と`legal-knowledge-builder`(Legal Knowledge Builder)を1台のホスト(AWS EC2等)上で連携起動するための、両リポジトリとは独立した第3のディレクトリ。各リポジトリの`docker-compose.yml`はそれぞれの更新頻度・git履歴を保ったまま変更せず、ここでは`include:`で取り込んで繋ぐだけの薄い接着層のみを持つ。

## ディレクトリ配置

```
<parent>/
├── bim-aiagent01/
├── legal-knowledge-builder/   (または任意名、.envのLKB_DIRで指定)
└── deployment/                 ← このディレクトリ
```

## 初回セットアップ

1. 3つのディレクトリを上記の通り並べて配置する(それぞれ`git clone`)。
2. `cp .env.example .env`し、値を埋める(秘密情報のためコミットしない)。
3. **`knowledge/`(Legal Knowledge Builderの成果物、約260MB)を転送する。** `legal-knowledge-builder/knowledge/`はそのリポジトリの`.gitignore`対象で、`git clone`しただけでは存在しない。WindowsのOneDrive依存のingestパイプライン(`uv run legal-knowledge-builder build all`)でしか生成できず、EC2上では再生成できないため、WSL側で構築済みの`knowledge/`を`rsync`または`scp`でこのホストの`legal-knowledge-builder/knowledge/`へ転送しておくこと。
   ```bash
   rsync -avz --progress ~/bim_ai_agent/legal-knowledge-builder/knowledge/ user@ec2-host:~/legal-knowledge-builder/knowledge/
   ```
   このディレクトリは`docker-compose.yml`でコンテナへbind mountされる(イメージには焼き込まない)。法令データを更新した場合も、Dockerイメージの再ビルドではなくこの`rsync`のやり直しだけで反映される。
4. (実ドメインでHTTPS公開する場合、推奨)DNSのAレコードを設定する。取得済みドメイン(例: `bim-aiagent.com`)の管理画面で以下3つをこのEC2のグローバルIP(**Elastic IP推奨**、後述)へ向ける。Caddyの自動証明書取得(Let's Encrypt HTTP-01チャレンジ)が外部から80番ポートに到達できる必要があるため、起動前に反映させておくこと(浸透待ちが必要な場合がある)。
   - `bim-aiagent.com` → フロントエンド
   - `www.bim-aiagent.com` → フロントエンド(同上)
   - `api.bim-aiagent.com` → backend API
   EC2のセキュリティグループ(インバウンドルール)は `22(SSH)`・`80(HTTP)`・`443(HTTPS)` のみ開ければよい。`5173`(frontend)・`8000`(backend/tailscale)・`8100`(legal-knowledge-builder)はCaddyがコンテナ名で到達するため外部公開不要——これらのポートをセキュリティグループで開けたままにしないこと(認証の無いAPIがインターネットに直接晒される)。
5. (実ドメインでHTTPS公開する場合、必須)Basic認証のパスワードハッシュを生成する。**backend(`main.py`)側にはアプリケーション認証が一切無く**、`/archicad/elements/move`等の書き込み系エンドポイントや`/agent/chat`(Anthropic API課金が発生)を認証無しで誰でも直接叩けてしまう状態のため、Caddy側でBasic認証をかけて塞ぐ(`Caddyfile`参照)。
   ```bash
   docker run --rm caddy:2-alpine caddy hash-password --plaintext '実際のパスワード'
   ```
   出力された`$2a$...`形式の文字列を`.env`の`CADDY_BASIC_AUTH_HASH`にそのまま貼り、`CADDY_BASIC_AUTH_USER`に任意のユーザー名を設定する。ブラウザは`bim-aiagent.com`(frontend)と`api.bim-aiagent.com`(backend)それぞれに初回アクセス時ネイティブのBasic認証ダイアログを1回ずつ出す(別オリジンのため資格情報のキャッシュもオリジン単位、ログインフォームではなくブラウザ標準のポップアップ)。
6. 起動する。
   ```bash
   cd deployment
   # ドメイン無し(IP直打ちでの動作確認、または開発用):
   docker compose -f compose.yaml -f compose.prod.yaml up -d --build
   # 実ドメインでHTTPS公開する場合(Caddy追加):
   docker compose -f compose.yaml -f compose.prod.yaml -f compose.proxy.yaml up -d --build
   ```
   (`.env`の`COMPOSE_FILE`に使うファイルを`:`区切りで設定していれば`docker compose up -d --build`だけでよい。例: `COMPOSE_FILE=compose.yaml:compose.prod.yaml:compose.proxy.yaml`)

## 動作確認

- `curl http://localhost:8100/health` — Legal Knowledge Builderの検索API
- `curl http://localhost:8000/health` — bim-aiagent01 backend
- backendから法令検索APIへ実際に到達できるかは`GET /legal/status`(bim-aiagent01側)で確認する。`http://legal-knowledge-builder:8100`という共有ネットワーク越しのコンテナ名で解決される(`compose.yaml`のコメント参照)。
- Caddy利用時: `docker compose logs caddy`で証明書取得(`certificate obtained successfully`)を確認し、`curl -I https://bim-aiagent.com`・`curl https://api.bim-aiagent.com/health`が外部から到達できるか確認する。

## 既知の制約・今後の課題

- **`include:`にはDocker Compose v2.20.0以降が必要。** EC2のOS標準リポジトリのDocker(古いバージョンのことが多い)ではなく、Docker公式aptリポジトリから最新のDocker Engine/Compose pluginを入れること。
- **`include:`のパス内の`${LKB_DIR}`変数展開は、Docker Composeのバージョンによっては意図通り動かないことが既知の問題として報告されている。** `docker compose config`で解決結果を確認し、うまくいかない場合はディレクトリ名を既定値(`legal-knowledge-builder`)に合わせるのが確実。このWSL開発環境も含め、`bim-aiagent01`・`legal-knowledge-builder`・`deployment`の3つは`~/bim_ai_agent/`配下に既定値のディレクトリ名(kebab-case)で揃えてあるため、`LKB_DIR`の設定は不要(未設定のままでよい)。
- **(2026-08-14対応済み)フロントエンドの`API_BASE`を環境変数化した。** `bim-aiagent01/frontend/src/api/client.ts`の`API_BASE`は`VITE_API_BASE_URL`(ブラウザ側で評価される値、Vite標準の`VITE_`接頭辞)を読み、未設定ならローカル開発既定値(`http://localhost:8000`)にフォールバックする。`compose.prod.yaml`の`frontend.VITE_API_BASE_URL`に`.env`の`BACKEND_PUBLIC_URL`(例: `https://bim.example.com:8000`)を渡す。**`FRONTEND_ORIGIN`と`BACKEND_PUBLIC_URL`は同じEC2ホストを指すが別の値であり、自動導出はしていないため両方手動で設定する必要がある。** また、現状フロントエンドは`npm run dev`(Viteの開発サーバー)のまま稼働している間のみこの値が反映される——将来`vite build`によるビルド済み静的配信に切り替えた場合、`VITE_*`はビルド時に埋め込まれる値になるため、コンテナの`environment`を変えるだけでは反映されなくなる点に注意(下記「本格的な本番ハードニング」と合わせて対応が必要)。
- **`compose.proxy.yaml`(Caddy)はTLS終端とドメインルーティングのみを担う。** frontend自体は引き続き`npm run dev`(Vite dev server)のまま(上記フロントエンド`VITE_ALLOWED_HOSTS`の注記参照)。Caddyのステート(`caddy_data`ボリューム)を消すと証明書を再取得することになる(Let's Encryptのレート制限に注意、失いたくない場合はEC2側のDockerボリュームもバックアップ対象に含めること)。
- **本格的な本番ハードニングは未着手。** 両リポジトリのDockerfileは現状開発用パターン(`uv run uvicorn --reload`、`npm run dev`)のまま(現時点では「まだ開発環境」という前提で意図的に据え置いている)。マルチステージビルド・読み取り専用マウント等への切り替えは別途の課題として残す。
- **永続化データの扱い**: `bim_cache.db`・ChromaDBインデックス等の"一時保存データベース"は、現状の開発用bind mount(各リポジトリの`docker-compose.yml`が定義する`./backend`・`./chroma/data`等)のまま据え置いている(デプロイし直すと消えてもよいという前提を確認済み。BIM再同期・検索インデックス再構築で復元できる)。`legal-knowledge-builder/knowledge/`だけが「消えたら復元不可能」という性質上の例外であるため、上記「初回セットアップ」で個別に扱っている。
