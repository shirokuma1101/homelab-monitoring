# TrueNASアップデート後のNetdata設定再適用

TrueNAS SCALEをアップデートすると、`/etc/netdata/netdata.conf`がTrueNAS標準の内容に置き換わります。アップデート完了後に、[`Supporterino/truenas-graphite-to-prometheus`](https://github.com/Supporterino/truenas-graphite-to-prometheus)が提供する`netdata.conf`を再適用してください。

この作業はTrueNASのShellで実行します。

## 前提

- `truenas-graphite-to-prometheus`をTrueNAS管理ユーザーのホームディレクトリに配置している
- 管理ユーザーが`sudo`を実行できる
- Dockerホスト上の`graphite_exporter`が稼働している

以下では、リポジトリの配置先を`~/truenas-graphite-to-prometheus`とします。`truenas_admin`ユーザーの場合、通常は`/home/truenas_admin/truenas-graphite-to-prometheus`です。

## 再適用手順

### 1. リポジトリと設定ファイルを確認する

```bash
cd ~/truenas-graphite-to-prometheus
ls -l netdata.conf
```

`netdata.conf`が表示されることを確認します。

リポジトリをまだ配置していない場合は、次のコマンドで取得します。

```bash
cd ~
git clone https://github.com/Supporterino/truenas-graphite-to-prometheus.git
cd ~/truenas-graphite-to-prometheus
ls -l netdata.conf
```

### 2. `netdata.conf`を再適用する

```bash
sudo cp "$HOME/truenas-graphite-to-prometheus/netdata.conf" /etc/netdata/netdata.conf
sudo chown root:root /etc/netdata/netdata.conf
```

### 3. Netdataを再起動する

```bash
sudo systemctl restart netdata
systemctl status netdata --no-pager
```

出力が`active (running)`であれば、Netdataの再起動は完了です。起動に失敗した場合は、次のコマンドでログを確認します。

```bash
sudo journalctl -u netdata -n 100 --no-pager
```

### 4. Exporter設定を確認する

TrueNAS Web UIの**Reporting → Exporters**を開き、Graphite Exporterの設定を確認します。

| 項目 | 設定 |
| --- | --- |
| Prefix | `truenas` |
| Namespace（旧UIのHostnameに相当） | TrueNASごとに一意な名前を指定する。Prometheusメトリクスの`instance`ラベルとして使用される |
| Update Every | PrometheusまたはVictoriaMetrics側のscrape間隔に合わせる |
| Send Names Instead Of Ids | 未指定のまま（デフォルト値を使用） |
| Destination IP / Port | Dockerホスト上の`graphite_exporter`の宛先。この構成の受信ポートは`9109` |

### 5. メトリクスの更新を確認する

Netdataが稼働していることを再確認します。

```bash
systemctl is-active netdata
```

`active`と表示されたら、GrafanaまたはVictoriaMetricsでTrueNASのメトリクスが更新されていることを確認します。

## 複数台のTrueNASを識別する

複数台のTrueNASから同じ`graphite_exporter`へ送信する場合は、各TrueNASの**Reporting → Exporters**でNamespaceを一意にします。

例：

| TrueNAS | Namespace |
| --- | --- |
| メイン機 | `truenas-main` |
| レプリケーション先 | `truenas-repl` |

このマッピング設定はNamespaceを各TrueNASメトリクスの`instance`ラベルへ変換します。Exporter設定上部のNameは設定項目の表示名であり、`instance`には使用されません。

Prometheusまたはvmagentのscrape設定では、Exporterが生成したラベルを維持するために`honor_labels: true`を指定します。`static_configs`に固定の`instance`ラベルは設定しません。

```yaml
- job_name: truenas_exporter
  metrics_path: /metrics
  honor_labels: true
  static_configs:
    - targets:
        - 192.168.1.42:9108
```

設定を変更したら、Prometheusまたはvmagentの設定を再読み込みするか、サービスを再起動します。

### Targets画面の`instance`について

PrometheusのTargets画面には、引き続き`instance="192.168.1.42:9108"`と表示されます。これは`graphite_exporter`自体のスクレイプ先を示すターゲットラベルであり、異常ではありません。

`honor_labels: true`はTargets画面の表示を変更する設定ではありません。Exporterが公開する各メトリクスに`instance="truenas-main"`や`instance="truenas-repl"`が含まれている場合に、そのラベルを保存時に優先します。そのため、次のような違いが生じます。

| 対象 | `instance`の例 |
| --- | --- |
| PrometheusのTargets画面 | `192.168.1.42:9108` |
| `up`などスクレイプ対象自体のメトリクス | `192.168.1.42:9108` |
| TrueNASのメトリクス | `truenas-main`、`truenas-repl` |

保存されたTrueNASメトリクスは、PromQLで確認できます。

```promql
count by (instance) ({job="truenas"})
```

Exporterの出力を直接確認する場合は、次のコマンドを実行します。

```bash
curl -s http://192.168.1.42:9108/metrics \
  | grep 'instance="truenas-repl"' \
  | head
```

TrueNAS固有の名前が`instance`ではなく`exported_instance`に入っている場合は、`honor_labels: true`が反映されていません。scrape設定のインデントと、Prometheusまたはvmagentへの設定反映を確認してください。

## 次回以降の短縮手順

リポジトリがすでに配置されている場合、TrueNAS SCALEのアップデート後に実行する基本手順は次のとおりです。

```bash
cd ~/truenas-graphite-to-prometheus
test -f netdata.conf || { echo "netdata.confが見つかりません" >&2; exit 1; }

sudo cp "$HOME/truenas-graphite-to-prometheus/netdata.conf" /etc/netdata/netdata.conf
sudo chown root:root /etc/netdata/netdata.conf
sudo systemctl restart netdata

systemctl status netdata --no-pager
```

## 注意事項

- TrueNAS SCALEのアップデート後は、`/etc/netdata/netdata.conf`が置き換わっている前提で毎回この手順を実施します。
- 複数台のTrueNASから送信する場合は、各Reporting ExporterのNamespaceを`truenas-main`、`truenas-repl`のように重複しない値にします。
- `graphite_mapping.conf`はTrueNAS側ではなく、Dockerホスト側の`graphite_exporter`で使用する設定です。この手順では変更しません。
- このリポジトリでは、対応するマッピングを[`graphite-exporter/mappings/truenas.yml`](../graphite-exporter/mappings/truenas.yml)として管理しています。
