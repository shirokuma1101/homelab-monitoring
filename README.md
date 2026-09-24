# Homelab Monitoring

Docker ComposeでGrafanaとGraphite Exporterを起動し、TrueNASのメトリクスをVictoriaMetricsで可視化するための構成です。

## 構成

- **Grafana**: ダッシュボード表示
- **Graphite Exporter**: TrueNASから受信したGraphite形式のメトリクスをPrometheus形式へ変換
- **VictoriaMetrics**: メトリクス保存先（別途用意）

## 起動

`.env.example` を `.env` にコピーし、環境に合わせて編集します。

```sh
cp .env.example .env
```

少なくとも、Grafanaの管理者パスワードとVictoriaMetricsのURLを変更してください。

```sh
docker compose up -d
```

起動後、Grafanaは次のURLで利用できます。

```text
http://localhost:4000
```

## ポート

| ポート | 用途 |
| --- | --- |
| `4000` | Grafana（コンテナの3000番） |
| `9108` | Graphite ExporterのPrometheusメトリクス |
| `9109/tcp` | TrueNAS Graphite受信 |
| `9109/udp` | TrueNAS Graphite受信 |

## TrueNASの設定

TrueNASのReporting設定でGraphite送信を有効にし、Graphite Exporterを実行しているホストのアドレスとポート `9109` を指定します。

メトリクスのマッピングは [`graphite-exporter/mappings/truenas.yml`](graphite-exporter/mappings/truenas.yml) で管理しています。

TrueNAS SCALEのアップデート後は、上書きされたNetdata設定を再適用する必要があります。詳しい手順は[TrueNASアップデート後のNetdata設定再適用](docs/truenas-update.md)を参照してください。

## Tailscale 経由の Node Exporter

Node Exporter を取得する場合は、`.env` に以下を設定します。

```dotenv
TAILSCALE_AUTHKEY=tskey-auth-...
NODE_EXPORTER_TARGET=192.168.2.10:9100
NODE_EXPORTER_JOB_NAME=node_exporter
NODE_EXPORTER_INSTANCE=shirokuma1101
VICTORIAMETRICS_REMOTE_WRITE_URL=https://your-victoriametrics.example/api/v1/write
```

`NODE_EXPORTER_TARGET` には Node Exporter の到達可能な IP または MagicDNS 名とポートを指定してください。`NODE_EXPORTER_JOB_NAME` と `NODE_EXPORTER_INSTANCE` は任意に変更できます。省略した場合はそれぞれ `node_exporter` と `shirokuma1101` です。`192.168.2.10` を Tailscale 経由で取得するには、tailnet で `192.168.2.0/24` などの対応するサブネットルートが広告・承認され、アクセスが許可されている必要があります。VictoriaMetrics の URL は単一ノードの remote write エンドポイントの例です。クラスター構成では適切な vminsert エンドポイントを指定してください。

```sh
docker compose --profile tailscale-node up -d
docker compose --profile tailscale-node logs -f tailscale vmagent
```

`vmagent` のログでスクレイプと remote write のエラーがないことを確認し、VictoriaMetrics で `up{job="node_exporter",instance="shirokuma1101"}` が `1` になることを確認します。Tailscale の認証キーは `.env` に保存され、`.env` は Git の対象外です。既存の Tailscale 状態は Docker ボリュームに保持されます。

Tailscale はコンテナ内の userspace モードで動作し、SOCKS5 プロキシ経由のスクレイプ通信だけを中継します。ホストの TUN デバイス、ルート、ファイアウォール設定は変更しません。コンテナの外へ Tailscale のポートは公開していません。

### 既存環境からの追加手順

既存の `.env` を残したまま、`TAILSCALE_AUTHKEY`、`NODE_EXPORTER_TARGET`、`VICTORIAMETRICS_REMOTE_WRITE_URL` を追記します。`NODE_EXPORTER_JOB_NAME` と `NODE_EXPORTER_INSTANCE` は変更したい場合だけ追記します。`.env.example` を既存の `.env` に上書きコピーしないでください。初回認証に使う Tailscale auth key は管理画面で発行し、`TAILSCALE_AUTHKEY` に設定します。`VICTORIAMETRICS_REMOTE_WRITE_URL` は既存の Grafana 用 `VICTORIAMETRICS_URL` とは別の書き込み先です。

```sh
docker compose --profile tailscale-node config --quiet
docker compose --profile tailscale-node pull tailscale vmagent
docker compose --profile tailscale-node up -d --no-deps tailscale vmagent
docker compose --profile tailscale-node ps
docker compose --profile tailscale-node logs --tail=100 tailscale vmagent
```

この起動コマンドは追加した2サービスだけを対象にし、既存の Grafana と Graphite Exporter は再作成しません。`up{job="node_exporter",instance="shirokuma1101"}` が `1` になれば取得を確認できます。戻す場合は `docker compose --profile tailscale-node stop vmagent tailscale` を実行します。データを残すため、`down -v` は使用しないでください。

## 設定ファイル

- `.env`: ローカル環境用の設定。Gitにはコミットしないでください。
- `.env.example`: 設定項目のサンプル
- `compose.yml`: コンテナ、ネットワーク、ボリュームの定義
- `grafana/provisioning/datasources/victoriametrics.yml`: Grafanaのデータソース設定

## 停止・ログ確認

```sh
docker compose ps
docker compose logs -f
docker compose down
```

`grafana-data` ボリュームにはGrafanaのデータが保存されます。
