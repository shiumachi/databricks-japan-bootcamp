# Lakebase Workshop Ticket App

Databricks AppsからLakebase Postgresへ接続し、branchごとに分離されたデータを
更新するワークショップ用のサンプルアプリです。参加者はチケットのstatus、owner、
commentを更新し、自分のbranchだけに変更が反映されることを確認します。

## 参加者向けガイド

- [日本語インストールガイド](docs/participant-install-guide.ja.md)
- [English installation guide](docs/participant-install-guide.en.md)
- [日本語手動接続ガイド](docs/participant-install-guide.manual.ja.md)
- [English manual connection guide](docs/participant-install-guide.manual.en.md)

標準手順ではDatabricks AppsのDatabase resourceを使用します。Database resourceを
利用できない場合だけ、手動接続ガイドと`app.manual.yaml`を使用してください。

## 講師向け準備と片付け

ワークショップ環境はDatabricks workspace上のnotebookだけで準備できます。ローカルの
Databricks CLI、jq、psqlは不要です。

### 事前準備

1. LakebaseとDatabricks Appsを利用できるserverless workspaceを用意します。
2. 参加者専用のworkspace groupを作成し、参加者を追加します。groupはsetup notebookの
   実行前に存在している必要があります。顧客環境では`users`ではなく専用groupを使います。
3. Workspaceで**Create**→**Git folder**を選び、Repository URLに
   `https://github.com/databricks-solutions/databricks-japan-bootcamp.git`、Git providerに
   `GitHub`を指定します。
   **Advanced**で**Sparse checkout mode**を有効にし、Cone patternsへ`lakebase-workshop`を
   入力して作成します。
4. Git folder内の`lakebase-workshop/notebooks/facilitator_setup`を開いて一度Run allし、
   表示されたwidgetへ参加者数、project接頭辞、参加者グループ、権限モードを入力します。
   初回のRun allは確認だけで、環境を変更しません。
5. `5. 実行確認`を`環境構築を実行`へ変更して、もう一度Run allします。参加者8名ごとに
   master projectを作成し、schemaとsample dataの投入、`production` branchのprotected化、
   参加者グループへの権限付与と、付与した権限のAPIでの再確認を行います。
6. 最後に出力される参加者割り当て表をもとに、参加者の氏名、master project、
   branch名の対応表を配布します。

1つのprojectでunarchivedのbranchは最大10本です。setup notebookは講師確認用を含めて
1 projectあたり9本以内に収まるよう、参加者を8名ずつ分割します（30名なら8／8／8／6名）。

### 開始前の権限確認

標準の`can_manage`モードでは参加者が自分でbranchを作成するため、開始前に次の3点を
すべて確認します。

1. setup notebookの出力に、projectごとの`[verify] ... CAN_MANAGE (direct)`がある。
2. projectのPermissions画面で、参加者専用groupの権限が`CAN MANAGE`になっている。
   `CAN CREATE (Inherited)`だけの場合は開始しません。
3. 代表参加者または参加者相当のテストアカウントで、`production`からTTL付きbranchを
   1本作成できる。さらにそのbranchへ本人のroleを追加し、SQL Editorでクエリできる。
   確認用branchは確認後に削除します。

`CAN MANAGE`を確認できない場合は、継承された権限を変更せず、setup notebookを再実行して
参加者専用groupへ明示的に付与し、上の3点を確認し直します。`precreate`モードでは、
branch作成の代わりに、代表参加者が割り当て済みbranchを選択できることを確認します。

### 参加者本人のrole（SQL Editor）

SQL Editorは参加者本人のPostgres roleで接続します。branchは`production`のroleを
引き継ぎますが、`production`にあるのは講師とtable所有用のroleだけです。そのため、
参加者が作ったbranchには本人のroleがなく、そのままではSQL Editorを使えません。

SQL Editorのハンズオンの最初に、参加者が自分のbranchの**Roles & Databases**→
**Add role**で、**Principal**に本人を選び、`databricks_superuser`にチェックして追加します。Appのroleとは別に必要です。
superuserを付けるのは本人のTTL付きbranchだけにし、`production`や参加者groupには
付けません。

### 時間配分の目安

| 時間 | 内容 |
|---|---|
| 0〜5分 | 接続確認、参加者group、master projectの割り当て確認 |
| 5〜15分 | Lakebase、branch、Database resourceの説明 |
| 15〜75分 | App作成、branch、role、resource、Deploy、更新、Tables確認 |
| 75〜90分 | 完了確認、適用判断、QA |

### 完了確認

参加者ごとに、branch名、更新したticket ID、`production`の同じticketが変わっていない
ことの3点を回収します。大人数ではチェックフォームまたはチャット投稿を使います。
未完了の参加者は、接続バッジ、role、resource keyの順に確認します。

### トラブルシューティング

| 症状 | 確認点 | 復旧操作 |
|---|---|---|
| SQLite demo modeのまま | Database resourceとDeploy状態 | resourceを追加して再Deploy |
| `password authentication failed` | 自分のbranchにApp service principalのroleがあるか | App IDのroleを作成 |
| `permission denied for table tickets` | roleのsuperuserオプション | roleを修正または再作成 |
| `valueFrom: lakebase-demo`関連のエラー | resource key | keyを`lakebase-demo`へ修正して再Deploy |
| 全員がbranchを作成できず`CAN CREATE (Inherited)`だけが表示される | 参加者専用groupの権限 | setupを再実行して`CAN MANAGE`を明示的に付与 |
| SQL Editorを開けない、またはSQL Editorで`permission denied` | 自分のbranchに本人のroleがあるか、`databricks_superuser`が付いているか | 自分のbranchで本人のroleを追加する。Principalの候補で本人がグレーになっている場合はroleがすでにあるので、行の「⋮」→**Edit role**で`databricks_superuser`を付ける |

project権限の`CAN MANAGE`はbranchを作るための権限で、テーブルへのクエリ権限ではありません。
SQL Editorの問題は`CAN MANAGE`の付け直しでは直りません。

### 片付け

終了後はGit folder内の`lakebase-workshop/notebooks/facilitator_cleanup`を実行します。
初回は削除対象の一覧だけを表示し、`2. 削除確認`へproject接頭辞を再入力した場合だけ、
`production`のprotectedを解除してprojectを完全に削除します。削除は取り消せません。

## ローカルデモ

ローカルではSQLite demo modeで起動します。Lakebase接続やDatabricks credentialは
不要です。

```bash
uv sync --frozen
uv run --frozen python app.py
```

起動後、`http://localhost:8000`を開きます。`[DEMO]`で始まる3件のチケットが
表示されれば正常です。

## Unit Test

```bash
uv run --frozen python -m unittest discover -s tests -v
```

## ファイル構成

- `app.py`: application server、API、Lakebase接続処理
- `app.yaml`: Database resource方式のDatabricks Apps設定
- `app.manual.yaml`: 手動接続方式のフォールバック設定
- `requirements.txt`: Databricks Appsが導入するPython runtime依存関係
- `pyproject.toml`、`uv.lock`: ローカル検証用に固定したPython依存関係
- `static/`: browser UI
- `sql/`: PostgreSQL／SQLiteのschemaとsample data
- `notebooks/facilitator_setup.py`: 講師用の環境作成notebook
- `notebooks/facilitator_cleanup.py`: 講師用の環境削除notebook
- `notebooks/participant_connect_helper.py`: 手動接続用の診断helper
- `docs/`: 参加者向け日英インストールガイド
- `tests/`: unit test

ワークショップ設計資料とworkspace固有のdeployment scriptは、この公開版に含まれません。

## 第三者ライブラリ

このアプリは[Psycopg](https://github.com/psycopg/psycopg) 3.3.4を使用します。
licenseはLGPL-3.0-onlyです。依存関係は`pyproject.toml`と`uv.lock`で固定しています。

## Support and License

本コンテンツはDatabricksの公式support対象ではありません。不具合はこの
repositoryのGitHub Issueへ報告してください。利用条件はrepository rootの
[LICENSE.md](../LICENSE.md)と[NOTICE.md](../NOTICE.md)を参照してください。
