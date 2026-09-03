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
