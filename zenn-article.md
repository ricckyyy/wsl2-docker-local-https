# WSL2+Dockerでローカル HTTPS 環境を作る（test.example.local:8443）

WSL2 に直接インストールした Docker を使って、カスタムドメイン + HTTPS でアクセスできるローカル開発環境を作った記録です。
「ポートが届かない」という罠にもはまったので、トラブルシューティングも合わせてまとめます。

## 環境

- Windows 10
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
  -keyout ~/test-nginx/certs/server.key \
  -out ~/test-nginx/certs/server.crt \
  -subj "/CN=test.example.local" \
  -addext "subjectAltName=DNS:test.example.local"
```

`-addext "subjectAltName=..."` を付けないと Chrome でエラーになるので注意。

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

WSL2 で systemd を使っていない場合、Docker デーモンは手動で起動する必要があります。

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

## トラブルシューティング：ポートが届かない

別の環境で試したとき、ブラウザからアクセスできない問題が起きました。
以下の順で確認すると原因が絞れます。

### ① hosts が効いているか確認

Windows の PowerShell で実行します。

```powershell
ping test.example.local
```

`127.0.0.1` 以外のIPが返る、または名前解決できない → hosts の編集が反映されていない。

```powershell
ipconfig /flushdns
```

DNS キャッシュをクリアして再度 `ping` で確認します。

### ② ポートが開いているか確認

```powershell
Test-NetConnection -ComputerName localhost -Port 8443
```

`TcpTestSucceeded : False` の場合、以下を確認。

### ③ WSL2 のポートが Windows 側に出ているか確認

```powershell
netstat -an | findstr 8443
```

`0.0.0.0:8443` が表示されない場合、WSL2 → Windows へのポートフォワードが機能していません。

VPN クライアントやセキュリティソフトが WSL2 のポートフォワードを壊すことがあります。
その場合は手動でフォワードを追加します（管理者 PowerShell）：

```powershell
# WSL2 の IP を取得
$wslIp = wsl hostname -I

# ポートフォワードを追加
netsh interface portproxy add v4tov4 `
  listenport=8443 listenaddress=0.0.0.0 `
  connectport=8443 connectaddress=$wslIp
```

確認：

```powershell
netsh interface portproxy show all
```

### ④ ファイアウォールがブロックしていないか確認

管理者 PowerShell でポートの受信を許可：

```powershell
New-NetFirewallRule -DisplayName "WSL 8443" -Direction Inbound -LocalPort 8443 -Protocol TCP -Action Allow
```

### ⑤ WSL 内から curl で疎通確認

```bash
curl -k https://localhost:8443
```

これが成功する（`Hello from...` が返る）なら nginx は正常で、Windows 側の問題に絞れます。

---

## まとめ

| 確認ポイント | コマンド |
|-------------|---------|
| hosts の反映 | `ping test.example.local` |
| ポートの疎通 | `Test-NetConnection -Port 8443` |
| WSL2 ポートフォワード | `netstat -an \| findstr 8443` |
| nginx の動作 | WSL 内で `curl -k https://localhost:8443` |

WSL2 直接インストールの Docker は Docker Desktop と違い、ポートフォワードが自動でされないケースがあります。
`netstat` でポートが出ているか確認するのが最初の切り分けポイントです。
