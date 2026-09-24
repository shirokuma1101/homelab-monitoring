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

このホストの Tailscale コンテナを SOCKS5 プロキシとして使い、別ホストの Prometheus または VictoriaMetrics が Node Exporter を直接スクレイプします。スクレイプ対象・job 名・ラベルはすべて DB 側の `prometheus.yml` で管理します。この構成では vmagent の remote write は使いません。

このホストの `.env` に認証キーと、プロキシを公開する **このホストの LAN IP** を設定します。Node Exporter の IP `192.168.2.10` を `TAILSCALE_PROXY_BIND_ADDRESS` に指定しないでください。

```dotenv
TAILSCALE_AUTHKEY=tskey-auth-...
TAILSCALE_PROXY_BIND_ADDRESS=<このホストのLAN IP>
```

DB 側の既存 `prometheus.yml` の `scrape_configs` に次の job を追加します。`<このホストのLAN IP>` は上記と同じ値に置き換えてください。

```yaml
  - job_name: node_exporter
    proxy_url: socks5://<このホストのLAN IP>:1055
    static_configs:
      - targets:
          - 192.168.2.10:9100
        labels:
          instance: shirokuma1101
```

`proxy_url` は DB が Prometheus でも VictoriaMetrics の内蔵 scraper でも利用できます。DB 側で設定を再読み込みした後、`up{job="node_exporter",instance="shirokuma1101"}` が `1` になることを確認します。`192.168.2.10` を Tailscale 経由で取得するには、tailnet で対応するサブネットルートが広告・承認され、アクセスが許可されている必要があります。

Tailscale はコンテナ内の userspace モードで動作し、ホストの TUN デバイスやルートは変更しません。`1055` 番は指定したホスト IP で公開されます。SOCKS5 プロキシには認証がないため、LAN のファイアウォールで DB ホストからの接続だけを許可してください。`.env` は Git の対象外で、Tailscale の状態は Docker ボリュームに保持されます。

### 既存の vmagent 構成からの切り替え

既存の `.env` は上書きせず、`TAILSCALE_PROXY_BIND_ADDRESS` を追記します。旧 `NODE_EXPORTER_*` と `VICTORIAMETRICS_REMOTE_WRITE_URL` は不要です。まずプロキシを起動し、DB 側からの接続を確認します。

```sh
docker compose --profile tailscale-node config --quiet
docker compose --profile tailscale-node up -d --no-deps tailscale
docker compose --profile tailscale-node logs --tail=100 tailscale
```

DB ホストから `curl --proxy socks5h://<このホストのLAN IP>:1055 http://192.168.2.10:9100/metrics` で接続を確認し、DB 側の `prometheus.yml` に上記 job を追加・再読み込みします。`up{job="node_exporter",instance="shirokuma1101"}` が `1` になった後、旧 vmagent が稼働していれば停止・削除します。

```sh
docker stop monitoring-vmagent
docker rm monitoring-vmagent
```

旧 vmagent が存在しない場合は最後の2行を省略します。`vmagent-data` ボリュームは削除しません。この手順は Grafana と Graphite Exporter を再作成しません。

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
