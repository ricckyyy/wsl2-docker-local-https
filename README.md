# WSL2+Docker でローカル HTTPS 環境を作る（test.example.local:8443）

WSL2 に直接インストールした Docker を使って、カスタムドメイン + HTTPS でアクセスできるローカル開発環境を作った記録です。
「ポートが届かない」という罠にもはまったので、トラブルシューティングも合わせてまとめます。

## 環境

- Windows 10 / 11
- WSL2（Ubuntu 22.04）
- Docker（WSL 内に直接インストール、Docker Desktop は使わない）
- nginx:alpine（Docker コンテナ）

---

## 構成イメージ

```
ブラウザ
  ↓ https://test.example.local:8443
Windows hosts（127.0.0.1 → test.example.local）
  ↓
WSL2 ポートフォワード（8443）
  ↓
Docker コンテナ（nginx:alpine）
```

---

## 手順

### 1. プロジェクトディレクトリを作る

```bash
mkdir -p ~/test-nginx/certs
cd ~/test-nginx
```

### 2. 自己署名 SSL 証明書を生成する

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/server.key \
  -out certs/server.crt \
  -subj "/CN=test.example.local" \
  -addext "subjectAltName=DNS:test.example.local"
```

> `-addext "subjectAltName=..."` を付けないと Chrome でエラーになるので注意。

### 3. nginx.conf を作る

```nginx
server {
    listen 8443 ssl;
    server_name test.example.local;

    ssl_certificate     /etc/nginx/certs/server.crt;
    ssl_certificate_key /etc/nginx/certs/server.key;

    location / {
        return 200 'Hello from test.example.local:8443\n';
        add_header Content-Type text/plain;
    }
}
```

### 4. docker-compose.yml を作る

```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "8443:8443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs:/etc/nginx/certs:ro
    restart: unless-stopped
```

### 5. Docker を起動してコンテナを立ち上げる

WSL2 で systemd を使っていない場合、Docker デーモンは手動で起動します。

```bash
sudo service docker start
docker compose up -d
```

起動確認：

```bash
docker ps
# PORTS 列に 0.0.0.0:8443->8443/tcp が表示されれば OK
```

### 6. Windows の hosts ファイルに追記する

管理者権限の PowerShell で：

```powershell
Add-Content -Path 'C:\Windows\System32\drivers\etc\hosts' -Value '127.0.0.1 test.example.local'
```

反映確認：

```powershell
ping test.example.local
# 127.0.0.1 から返ってくれば OK
ipconfig /flushdns  # キャッシュが残っている場合はクリア
```

### 7. ブラウザでアクセス

```
https://test.example.local:8443
```

自己署名証明書なので証明書エラーが出ます。

- **Chrome**: ページ上で `thisisunsafe` とキーボード入力（見えないが入力できる）
- **Firefox**: 「詳細設定」→「セキュリティ例外を承認」
- **Edge**: 「詳細情報」→「test.example.local に進む」

---

## トラブルシューティング

### 切り分けフロー

```
ブラウザでアクセスできない
  │
  ├─① ping test.example.local → 127.0.0.1 以外 → hosts を確認
  │
  ├─② Test-NetConnection localhost 8443 → False
  │     │
  │     ├─ docker ps の PORTS が空 → ports: の設定を確認
  │     ├─ docker ps の PORTS が 8443/tcp（0.0.0.0 なし） → networks: internal の問題
  │     └─ docker ps の PORTS が 0.0.0.0:8443 → WSL2 ポートフォワードの問題
  │
  └─③ curl -k https://localhost:8443（WSL 内）→ 失敗 → nginx 自体を確認
```

---

### ① hosts が効いているか確認（Windows PowerShell）

```powershell
ping test.example.local
```

`127.0.0.1` 以外のIPが返る、または名前解決できない → hosts の編集が反映されていない。

```powershell
ipconfig /flushdns
```

DNS キャッシュをクリアして再度 `ping` で確認します。

---

### ② docker ps の PORTS 列を確認（WSL 内）

```bash
docker ps
```

| PORTS 列の表示 | 意味 | 対処 |
|--------------|------|------|
| `0.0.0.0:8443->8443/tcp` | 正常 | ③へ |
| 空（8443 なし） | `ports:` が未設定 | docker-compose.yml に `ports: - "8443:8443"` を追加 |
| `8443/tcp`（0.0.0.0なし） | ホスト非公開 | `networks: internal` のみに接続されている |

**`8443/tcp`（ホスト非公開）の場合の対処：**

`networks: internal` だけに接続されているとホストに公開されません。`default` ネットワークも追加します。

```yaml
services:
  nginx:
    ports:
      - "8443:8443"
    networks:
      - internal
      - default

networks:
  internal:
    internal: true
```

---

### ③ WSL2 のポートが Windows に届いているか確認（Windows PowerShell）

```powershell
netstat -an | findstr 8443
```

`0.0.0.0:8443` が表示されない場合、WSL2 → Windows のポートフォワードが機能していません。

**よくある原因：**

- VPN クライアント（Cisco AnyConnect、Zscaler 等）が WSL2 のネットワークを壊している
- `.wslconfig` の `networkingMode=mirrored` が原因になっているケースもある

**管理者権限がある場合：** 手動でフォワードを追加

```powershell
$wslIp = wsl hostname -I
netsh interface portproxy add v4tov4 `
  listenport=8443 listenaddress=0.0.0.0 `
  connectport=8443 connectaddress=$wslIp
```

**管理者権限がない場合：** WSL の IP に直接アクセスできるか確認

```bash
# WSL 内で IP を確認
hostname -I
```

```
# ブラウザで直接アクセス
https://<WSL の IP>:8443
```

---

### ④ WSL 内から nginx に直接アクセス確認

```bash
curl -k https://localhost:8443
```

成功する → nginx は正常、Windows 側のネットワーク設定の問題。

失敗する → コンテナの IP で試す：

```bash
docker inspect <コンテナ名> | grep IPAddress
curl -k https://<コンテナIP>:8443
```

これも失敗する → nginx の設定または証明書のパスを確認。

---

### `.wslconfig` の networkingMode について

`networkingMode=mirrored`（Windows 11 の WSL2 設定）を使っている場合、Docker の `ports:` が正しく機能しないことがあります。

```ini
# %USERPROFILE%\.wslconfig
[wsl2]
networkingMode=mirrored  # ← これが原因の場合がある
```

`NAT`（デフォルト）に戻すと解決することがあります：

```ini
[wsl2]
networkingMode=NAT
```

変更後は WSL を再起動します：

```powershell
wsl --shutdown
```

---

## まとめ

| 確認ポイント | コマンド | 実行場所 |
|------------|---------|---------|
| hosts の反映 | `ping test.example.local` | Windows |
| ポートの疎通 | `Test-NetConnection -ComputerName localhost -Port 8443` | Windows |
| PORTS 列の確認 | `docker ps` | WSL |
| WSL2 ポートフォワード | `netstat -an \| findstr 8443` | Windows |
| nginx の動作確認 | `curl -k https://localhost:8443` | WSL |
