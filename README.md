# VisibleTeam for herdr

herdrのペインで、Codexの開発チームを目に見える形で動かすプラグインです。

herdr内で動くMain（CodexまたはClaude Code）が、調査・実装・デザイン・レビュー・相談の
担当を別ペインのCodexとして起動します。担当の作業はすべて画面に表示され、
ユーザーはどのペインにも直接指示を入力できます。Mainは報告を回収して判断し、
役目を終えた担当のペインを閉じます。

```text
+--------------------------+--------------------------+
|                          | Worker   (investigation) |
|                          +--------------------------+
| Main                     | Worker   (implementation)|
| decide / integrate       +--------------------------+
|                          | Reviewer (independent)   |
+--------------------------+--------------------------+
```

## 特徴

- **隠れて動く担当がいない**。担当は独立した対話型Codexとして別ペインで動きます。
  Mainのサブエージェント機能は使いません。
- **Mainのコンテキストを守る**。大量のログや調査は担当側に留め、Mainは要点の報告だけを受け取ります。
- **独立レビュー**。実装とは別のReviewerがレビューし、再レビューも同じ会話で続けます。
- **役割ごとにモデルを選べる**。既定値は設定ファイルや起動時の指定で変更できます。
- **常駐プロセスなし**。Mainが同梱の補助スクリプトを呼んで管理します。
  オーケストレーターやダッシュボードは同梱しません。

## 役割

| 役割 | 既定のモデル | 担当すること | ペインを閉じる時点 |
|---|---|---|---|
| Main | 起動したセッションのモデル | 意図・設計方針・作業分割・統合・最終受理 | — |
| Worker | `gpt-6-luna` / `max` / Fast | 通常の調査・実装・テスト・デバッグ、CIなど長時間処理の監視 | レビュー全体の終了時。レビューしない作業は成果の受理後 |
| Design | `gpt-6.1-sol` / `max` | 重要なUI・操作・情報設計 | 成果の受理後 |
| Reviewer | `gpt-6.1-sol` / `high` | 実装から独立したレビュー | レビュー全体の終了時 |
| Advisor | `gpt-6-astra` / `xhigh` | 難しい計画の起草、設計判断や収束しない失敗の相談 | Mainの一連の作業の終了時 |

担当の数に固定の上限はありません。Mainが作業の独立性と調整のコストから決めます。
指摘の採否、Advisorの助言を採るかどうか、最終的な受理は常にMainが判断します。

## 必要な環境

- LinuxまたはmacOS
- Python 3.10以上（追加パッケージ不要）
- 同じホストで動くherdrとCodex CLI（両方が`PATH`から見えること）
- 担当に使うモデルを利用できるCodexアカウント

動作確認はherdr `0.9.0`、Codex CLI `0.154.0`、共有app-server `0.153.4`の組み合わせで
行っています。他のバージョンの動作は未確認です。SSH先で使う場合は、Main・herdr・担当を
すべてSSH先で動かしてください。

## インストール

### Codex

```bash
git clone https://github.com/phni3j9a/visible-team.git
cd visible-team
codex --enable plugins plugin marketplace add "$PWD" --json
codex --enable plugins plugin add visible-team@personal --json
```

Mainは、herdr内のペインで次のように起動します。

```bash
codex -c background_terminal_max_timeout=3600000
```

この設定でコマンド結果の待ち上限を1時間にします（既定は5分）。担当の報告を
長時間待つために必要で、Mainの起動時に指定しておく必要があります。

### Claude Code

```bash
claude plugin marketplace add phni3j9a/visible-team
claude plugin install visible-team@visible-team
```

担当はCodexで起動するため、Claude CodeをMainにする場合もCodex CLIが必要です。
更新は次のコマンドで取り込めます。

```bash
claude plugin marketplace update visible-team
claude plugin update visible-team@visible-team
```

## 使い方

このプラグインは明示的に呼び出したときだけ動きます。普段の開発依頼で
自動的に選ばれることはありません。

Codexでは次のように呼び出します。

```text
$visible-team:visible-team

このプロジェクトを調査して、必要な変更を実装してください。
担当の作業はherdrの別ペインで見えるようにしてください。
```

Claude Codeでは`/visible-team:visible-team`で呼び出します。

一度呼び出せば、その作業の追加指示で指定し直す必要はありません。
担当として起動されたCodexは、さらに別の担当を起動しません。

## モデルの設定

### 設定ファイル

`~/.config/visible-team/models.json`（`XDG_CONFIG_HOME`がある場合は
`$XDG_CONFIG_HOME/visible-team/models.json`）に、変えたい項目だけを書きます。

```json
{
  "reviewer": {"effort": "xhigh"}
}
```

役割は`worker`・`design`・`reviewer`・`advisor`、項目は`model`・`effort`・`service_tier`です。
書かなかった項目は既定値を使います。全項目の例は
[models.example.json](plugins/visible-team/skills/visible-team/references/models.example.json)にあります。

`service_tier`の扱いは次のとおりです。

- `"fast"`：Fastの起動引数（`-c 'service_tier="fast"' -c features.fast_mode=true`）を付けます。
- `null`：何も付けず、Codexの既存設定に従います。Fastを強制的に無効にするものではありません。

マージ後の設定は、herdrやモデルを起動せずに確認できます。

```bash
python3 plugins/visible-team/skills/visible-team/scripts/visible_team.py models
```

別のファイルを使う場合は、`--model-config /absolute/path/models.json`か環境変数
`VISIBLE_TEAM_MODEL_CONFIG`で指定します。読み込むファイルは1つだけで、
`--model-config`、環境変数、標準パスの順に選びます。指定したファイルがない場合や
内容が不正な場合は、ペインを作る前にエラーになります。

### 1回だけ変える

Mainに「今回のReviewerは○○モデル、推論強度○○で」と頼むと、その担当の起動時だけ
`spawn --model ... --effort ... --service-tier ...`で上書きします。設定ファイルは書き換えません。

優先順位は **起動時の指定 → 設定ファイル → 既定値** です。設定は`spawn`のたびに読むので、
変更は新しく起動する担当から反映され、起動済みの担当には影響しません。

## 担当の権限

担当は役割にかかわらず、次の設定で起動します。

```text
--sandbox workspace-write --ask-for-approval never -c default_permissions=":workspace"
```

- Mainの権限モード（Auto、フルアクセスなど）には追従せず、Main自身の設定も変えません。
- 書き込めるのは`--cwd`で指定した作業場所と、報告用の一時ディレクトリだけです。
- ネットワークなどの制限は既存のCodex設定に従います。
- ReviewerとAdvisorがプロジェクトを編集しないことは、役割の指示として守らせます。

権限が足りずに作業できない場合、担当は操作・対象・エラー・理由・完了した作業を
`blocked`として報告し、ペインを残します。Mainが自分の権限で対応してから、
同じ担当に続きを依頼します。担当の権限を自動で広げることはありません。

`default_permissions`の指定は、上記の環境で従来のsandbox指定だけでは担当が
フルアクセスになったことへの対処です。Codexを更新したときは、担当の実際の権限を確認し直してください。

## 報告の待ち方

Mainは`wait --until-event`を1本だけ動かし、どれかの担当から報告や対応が必要な状態が
届くまで待ちます。

- 同時に動ける待機はrunごとに1つです。
- 状態が変わらない間は、herdrへの確認間隔を2秒から最大10秒まで伸ばします。この間MainのLLMは動きません。
- 同じ通知は繰り返し返しません。新しい報告や状態の変化は通知します。

CIやビルドのように状態確認を繰り返す処理は、Workerに完了まで監視させます。
Mainは途中経過を確認し直さず、別の作業を進めるか報告を待ちます。

## 共有app-serverでの利用

共有app-serverで動くCodexでは、Mainのコマンドに`HERDR_PANE_ID`が渡らないことがあります。
その場合は、Mainの会話とherdrの端末を最初に明示的に結び付けます。

```bash
python3 "$helper" init --cwd "$project_dir" \
  --main-pane "$pane_id" --main-terminal-id "$terminal_id" \
  --socket "$herdr_socket_path"
python3 "$helper" doctor --run "$run_dir"
```

`helper`は同梱の`visible_team.py`の絶対パス、`run_dir`は`init`の出力です。
フォーカスや作業ディレクトリだけでMainの端末を決めず、会話IDか端末の内容で確認してから登録します。
登録後は、端末が移動しても同じ端末を追い、閉じられた場合は操作を止めます。

## 注意事項

- 担当のペインは、herdrが`done`を示しただけでは閉じません。報告の回収と担当の状態を確認してから閉じます。
- ユーザーが担当に直接指示した場合、担当は古い報告を無効にしてから作業し、改めて報告します。
  入力とペインの終了を完全に同期する仕組みではありません。
- 報告と起動記録は実行ごとの一時ディレクトリに残ります。OSの一時ファイル削除の対象になるため、
  重要な結果はプロジェクトの文書などに残してください。
- 委譲の判断では、通常のLuna MAXの利用コストを「ほぼ無料」とみなし、Mainのコンテキストの節約を優先します。
  これは設計上の仮定で、実際の料金を保証するものではありません。Workerを別モデルにした場合は適用しません。

## 開発

```bash
python3 -m unittest discover -s tests -v
python3 plugins/visible-team/skills/visible-team/scripts/visible_team.py --help
```

テストはherdrを模擬したもので、実際の画面や担当の動作は確認しません。
実機での確認手順は[docs/MANUAL_VALIDATION.md](docs/MANUAL_VALIDATION.md)にあります。

Mainのrollout JSONLから、待機回数・終了理由・`status`や`read`の回数を集計できます。

```bash
python3 plugins/visible-team/skills/visible-team/scripts/audit_wait_history.py \
  /path/to/main-rollout.jsonl
```

### 構成

| パス | 内容 |
|---|---|
| `.agents/plugins/marketplace.json` | Codex用のカタログ |
| `.claude-plugin/marketplace.json` | Claude Code用のマーケットプレイス定義 |
| `plugins/visible-team/.codex-plugin/plugin.json` | Codexプラグイン定義 |
| `plugins/visible-team/.claude-plugin/plugin.json` | Claude Codeプラグイン定義 |
| `plugins/visible-team/skills/visible-team/SKILL.md` | Mainと担当の指針 |
| `plugins/visible-team/skills/visible-team/references/` | 操作手順・Advisor・レビュー・Claude Code Mainの詳細 |
| `plugins/visible-team/skills/visible-team/scripts/` | herdr操作と報告の受け渡し、待機履歴の集計 |

## 参考

- [herdr agent automation](https://herdr.dev/docs/agent-automation/)
- [herdr CLI reference](https://herdr.dev/docs/cli-reference/)
- [Codex 設定リファレンス](https://learn.chatgpt.com/docs/config-file/config-reference)
- [Codex 権限設定](https://learn.chatgpt.com/docs/permissions)

## ライセンス

MIT License。herdrとCodex本体のコードは含みません。
