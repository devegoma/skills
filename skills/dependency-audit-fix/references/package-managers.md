# パッケージマネージャー別のコマンド選択

以下は候補。対象リポジトリが使うバージョンのヘルプと設定を確認し、対応する形式で実行する。監査のためのロックファイル再生成や別マネージャーへの変換は行わない。

## JavaScript / TypeScript

| 検出根拠 | 監査候補 | 修正候補と判断 |
| --- | --- | --- |
| `package-lock.json` / `npm-shrinkwrap.json` | `npm audit --json` | `npm audit fix --dry-run --json` で提案を確認し、互換性を確認した `npm audit fix`。`fixAvailable` の真偽・更新対象・メジャー変更情報も読む。dry-run だけで互換性を断定しない。 |
| `pnpm-lock.yaml` | `pnpm audit --json` | 対応版の `pnpm audit --fix` は overrides を書き込むことがある。実行前に一時コピーで提案を調べ、依存元の要求範囲を破る override は適用しない。安全なら適用し、必要な install で解決結果を同期する。 |
| `yarn.lock` と Yarn 1 | `yarn audit --json` | 組み込みの audit fix はない。監査で示された安全な修正版に向け、依存元または直接依存を既存制約内で限定更新する。 |
| `yarn.lock` と Yarn 2 以降 | `yarn npm audit --all --recursive --json` | audit fix はない。既存制約内での限定更新を使用する。Yarn 1 のコマンドやロック形式に切り替えない。 |
| `bun.lock` / `bun.lockb` | 対応版の `bun audit`、対応していれば `--json` | fix の有無をヘルプで確認する。なければ既存制約内での限定更新を検討する。監査非対応の版では、対象ロック形式を読める監査ツールがなければ未対応として停止する。 |

JavaScript の限定更新は、修正に必要なパッケージと依存元に絞り、対象バージョンと解決差分を事前確認する。対応版の `pnpm update <package>`、Yarn 1 の `yarn upgrade <package>`、Yarn 2 以降の `yarn up -R <package>`、`bun update <package>` 等を候補とし、各版の制約・再帰・ワークスペースの意味を確認する。単にコマンド名が update だから安全とは判断しない。

audit の exit code が非ゼロでも、正常な監査で脆弱性を検出した結果であることがある。Yarn 1 の JSON Lines 等、通常の JSON オブジェクトと異なる形式にも対応する。重大度フィルターや production 限定オプションで初回と再監査の範囲を変えない。

## Python

| 検出根拠 | 監査候補 | 修正候補と判断 |
| --- | --- | --- |
| `requirements*.txt` / pip | `pip-audit -r <requirements> -f json` | 対応版の `pip-audit --fix --dry-run -r <requirements>` で対象を確認し、`--fix` を検討する。元のピンが許す更新範囲を確認し、制約外への書き換えは適用しない。 |
| `uv.lock` / uv | 対応版の `uv export` で全対象グループを含む監査用 requirements を一時出力し、`pip-audit -r <temporary-file> -f json` | 一時出力への fix はプロジェクトの修正にならない。安全な更新が判明したら、`uv lock --upgrade-package <package>` 等で元の制約内を限定更新し、再出力して監査する。 |
| `poetry.lock` / Poetry、`Pipfile.lock` / Pipenv | 元のロックから監査用の requirements を一時出力し、`pip-audit` で監査する。対応版の `pipenv audit` 等があれば優先可能。 | 元のマネージャーで対象を限定してロックを更新する。全体のロック再解決が必要なら差分を事前確認する。 |

`pip-audit` を実行するツール環境と監査対象を分ける。対象指定なしで実行すると別環境を監査するおそれがある。export のオプション、プラットフォームマーカー、依存グループ、editable やローカル依存の未対応範囲を確認する。要求ファイルの監査に必要な依存解決ができなかった場合は未完了とする。

## その他のエコシステム

| 検出根拠 | 監査候補 | 修正候補と判断 |
| --- | --- | --- |
| `Cargo.lock` / Cargo | `cargo audit --json` | 利用版の `cargo audit fix --help` で fix の有無や追加ツール要件を確認する。なければ安全な `cargo update -p <crate>` を検討する。 |
| `composer.lock` / Composer | `composer audit --format=json` | 組み込みの汎用 audit fix を仮定しない。対象を絞った `composer update <package>` と必要な依存の更新を確認する。 |
| `Gemfile.lock` / Bundler | `bundler-audit check`、DB 更新を伴うなら `bundler-audit check --update` | `--update` は脆弱性 DB の更新であり依存の修正ではない。安全な対象に `bundle update <gem> --conservative` 等を検討する。 |
| `*.csproj` / NuGet | 利用 SDK に応じ `dotnet list package --vulnerable --include-transitive` または `dotnet package list --vulnerable --include-transitive`。対応版は JSON 出力を使用する。 | SDK の audit fix 相当の有無を確認し、なければ制約内で直接依存・依存元を限定更新する。復元時の監査対象設定と中央パッケージ管理も確認する。 |
| `go.mod` / Go | `govulncheck -json ./...` | 呼び出し到達性を評価するツールなので、ロック内の全 advisory 件数と混同しない。安全な修正版への対象を絞った `go get <module>@<version>` 等を検討する。 |

一覧にないマネージャーでは、プロジェクトの公式な監査手段と修正機能を確認して同じループへ接続する。対応するツール・修正手段が確認できなければ、その理由を報告して停止する。コマンドやオプションを推測で実行しない。
