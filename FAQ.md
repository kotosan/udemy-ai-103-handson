# よくあるつまずきと対処（FAQ）

ハンズオン全体で共通して起きやすい問題をまとめました。**まずここを確認してください。** レッスン固有の内容は各レッスンフォルダの `README.md` にあります。

---

## 1. モデルがデプロイできない：`isn't available due to insufficient quota`

**症状**：Foundry ポータルのモデルカタログでモデルは見えるのに、デプロイしようとすると「クォータが足りないため利用できません」と出る。デプロイ画面のどのリージョンを選んでも同じ。

**原因**：**クォータティア**です。設定ミスではありません。Foundry はサブスクリプションごとに等級（ティア）を割り当てており、**そのティアの表に載っているモデルにしか既定のクォータが付きません**。作りたてのサブスクリプションは最下位ティア（Free Tier ／ Tier 0）になりやすく、そこでクォータが付くのは次の4つだけです。

| モデル | 既定クォータ | 新しいサブスクリプションで使えるか |
|---|---|---|
| `gpt-4.1-mini` | 200,000 TPM | 使えない（非推奨＝Deprecated。退役は 2027-04-14） |
| `gpt-5-mini` | 500,000 TPM | **使える** |
| `o4-mini` | 100,000 TPM | 使えない（非推奨＝Deprecated。2026-11-19 に退役） |
| `text-embedding-3-small` | 1,000,000 TPM | 使える（埋め込みモデル） |

表には4つ載っていますが、`gpt-4.1-mini` と `o4-mini` は**非推奨（Deprecated）**です。Microsoft Learn のモデルのライフサイクルでは、Deprecated のモデルは**既存の顧客だけ**が新しいデプロイを作れます。既存の顧客かどうかは**サブスクリプション単位**（そのサブスクリプションで、そのモデルのバージョンをデプロイしたことがあるか）で決まり、同じテナントの新しいサブスクリプションには引き継がれません。つまり、**作りたてのサブスクリプションで使えるチャットモデルは `gpt-5-mini` だけ**です。

`gpt-5.4` や `gpt-5.4-nano` のような最新世代は、最下位ティアでは**表に行が無い＝割り当てゼロ**です。ポータルの **Manage → Quota** を開いても、そのモデルの行自体が存在しません（「残り0」ではありません）。

**現在のティアを確認する**（ポータルに表示が無いため API で確認します）：

```powershell
az rest --method get `
  --url "https://management.azure.com/subscriptions/$(az account show --query id -o tsv)/providers/Microsoft.CognitiveServices/quotaTiers?api-version=2025-10-01-preview"
```

`properties.currentTierName` に `Tier 1` などが返ります。

**対処（上から順に試してください）**

1. **使えるモデルに読み替える（推奨）**。各レッスンはモデル名を `.env` の環境変数に出しているので、値を変えるだけで動きます。読み替え先は `gpt-5-mini` です（大きいモデル役も軽量モデル役も同じ）。関数ツールを使うレッスンなど、`gpt-5-mini` では動かないレッスンもあります（次の 1-2 を参照）。
2. **クォータ増加を申請する**。[申請フォーム](https://aka.ms/oai/stuquotarequest) から、サブスクリプション・リージョン・モデル名・希望 TPM を明記します（サブスクリプションの **Owner** か **Contributor** が必要）。申請は「既存クォータを実際に使っている顧客」が優先されるので、**1 で動かしてから出すと通りやすい**です。
3. **待つ**。ティアは消費実績と支払い履歴に応じて自動昇格します。

**効かない回避策**

| やりがちなこと | 実際 |
|---|---|
| 別のリージョンでデプロイする | **効きません。** Global Standard のクォータは 2026年5月以降、サブスクリプション単位で全リージョン共通の1プールです（Quota ページの **Scope 列**が `Global` のとき） |
| 容量（TPM）を最小にして再試行 | 割り当てがゼロなら不可 |
| デプロイタイプを変える | プールは別建てになりますが、最下位ティアには対象モデルの行がありません |

> ℹ️ `gpt-4.1` / `gpt-4.1-mini` / `gpt-4.1-nano` / `gpt-4o` / `gpt-4o-mini` も非推奨（Deprecated）です。`gpt-4.1` 系の退役は **2027-04-14** ですが、退役の前でも、そのサブスクリプションで以前にデプロイしたことが無ければ新しくデプロイできません。最新の予定は [Model retirement schedule](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirement-schedule)、ライフサイクルの意味は [Foundry Models lifecycle and support policy](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirements) で確認できます。

---

## 1-2. レッスン別：最下位ティアで動くか一覧

**モデルを使わないレッスン（読み替え不要）**

`L1-2` / `L3-4` / `L5-2`

**`.env` のモデル名を `gpt-5-mini` に読み替えれば動くレッスン**

`.env.sample` にフォールバックをコメントで書いてあるレッスンは、コメントを外すだけです。書いていないレッスンは、値を `gpt-5-mini` に書き換えます。`gpt-5-mini` のデプロイは、L1-1 の README「クォータ不足でモデルをデプロイできないとき」の対処1のコマンドで作ります。「講座で確認」の列は、2026-10-23 に `gpt-5-mini` で実行して確かめたレッスンです。

| レッスン | 既定 | 読み替え先 | 講座で確認 | 補足 |
|---|---|---|---|---|
| `00_intro/L0-3_hello_foundry` | `gpt-5.4-nano` | `gpt-5-mini` | 済 | 推論してから答えるので、応答まで十数秒かかる |
| `01_plan_manage/L1-1_model_selection` | `gpt-5.4` / `gpt-5.4-nano` | `gpt-5-mini` / `gpt-5-mini` | 済 | 2つとも同じモデルになるので、大小の差は見られない（動画で確認） |
| `01_plan_manage/L1-3_deploy_cicd` | `gpt-5.4-nano`（`deploy_model.azcli`・`main.bicep` でデプロイ） | `gpt-5-mini`（バージョン `2025-08-07`） | − | モデル名とバージョンの両方を書き換える |
| `01_plan_manage/L1-4_quota_cost` | `gpt-5.4` | `gpt-5-mini` | − | |
| `01_plan_manage/L1-5_keyless_security` | `gpt-5.4` | `gpt-5-mini` | − | |
| `01_plan_manage/L1-6_observability_tracing` | `gpt-5.4` | `gpt-5-mini` | − | |
| `01_plan_manage/L1-7_responsible_ai` | `gpt-5.4-nano` | `gpt-5-mini` | − | |
| `02_genai_agents/L2-1_chat_app` | `gpt-5.4` | `gpt-5-mini` | − | |
| `02_genai_agents/L2-2_file_search_rag` | `gpt-5.4` | `gpt-5-mini` | 済 | |
| `02_genai_agents/L2-3_foundry_iq` | `gpt-5.4` | `gpt-5-mini` | − | |
| `02_genai_agents/L2-4_multistep_tools` | `gpt-5.4` | `gpt-5-mini` | 済 | |
| `02_genai_agents/L2-5_evaluate_rag` | `gpt-5.1`（採点役） | `gpt-5-mini` | 済 | `JUDGE_IS_REASONING_MODEL=true` のまま |
| `02_genai_agents/L2-6_single_agent` | `gpt-5.4` | `gpt-5-mini` | 済 | |
| `02_genai_agents/L2-8_mcp_tool` | `gpt-5.4` | `gpt-5-mini` | − | |
| `02_genai_agents/L2-9_multi_agent` | `gpt-5.4` | `gpt-5-mini` | 済 | 実行時間は約2倍 |
| `03_vision/L3-3_multimodal_understanding` | `gpt-5.4` | `gpt-5-mini` | − | 画像の入力に対応 |
| `04_text/L4-1_text_analysis` | `gpt-5.4` | `gpt-5-mini` | 済 | |
| `04_text/L4-3_speech_pipeline` | `gpt-5.4` | `gpt-5-mini` | − | |
| `05_info_extraction/L5-1_search_grounding` | `gpt-5.4` ＋ `text-embedding-3-large` | `gpt-5-mini` ＋ `text-embedding-3-small`（**次元を 1536 に**） | − | |

> ⚠️ **`L5-1` は次元数もセットで変えてください。** `3072` は `-large` 専用で、`-small` は最大 `1536` です。合わせないとインデックス作成が失敗します。既にインデックスを作っている場合は作り直しが必要です。
>
> ⚠️ `L4-4`（Voice Live）と `L5-3`（Content Understanding）は、使えるモデルをサービス側が決めています。`gpt-5-mini` で動くかは講座では確かめていません（`L5-3` は `set_defaults.py` が対応モデルの一覧を表示するので、そこに `gpt-5-mini` があるかを見てください。埋め込みの `text-embedding-3-large` も最下位ティアにはありません）。

**Tier 1 以上（`gpt-5.1`）が必要なレッスン**

| レッスン | 理由 |
|---|---|
| `02_genai_agents/L2-7_function_tool_agent` | エージェントに関数ツールを持たせる。Foundry Agent Service のツール対応表で、`gpt-5-mini` の Functions は No |
| `02_genai_agents/L2-10_hitl` | 同上 |
| `02_genai_agents/L2-11_tracing` | 同上 |
| `02_genai_agents/L2-12_optimize` | `gpt-5.1` の推論の強さ `none`（推論しない）と `medium` を比べる。`gpt-5-mini` は `none` を選べず、`temperature` も 400 になる |

`gpt-5.1` は Tier 1 以上の表にあります。ティアが上がるのを待つか、クォータ増加を申請してください（上の「対処」の 2・3）。

**読み替え先が無いレッスン（クォータ申請が必要）**

| レッスン | モデル | 理由 |
|---|---|---|
| `03_vision/L3-1_image_generation` | `gpt-image-2` | 最下位ティアに**画像生成モデルが1つも無い** |
| `03_vision/L3-2_video_generation` | `sora-2` | 最下位ティアに**動画生成モデルが無い**。加えて `sora-2`（2025-12-08）は **2026-10-15 に退役済み（後継なし）**で、ティアに関係なく実行できない |
| `04_text/L4-2_translation` の LLM 翻訳（②） | `gpt-5.4` | LLM 翻訳に使えるモデルは Translator が決めており、`gpt-5-mini` は `not supported` の 400 になる（講座の検証）。`.env` の `LLM_DEPLOYMENT=` を空にすると、① の NMT 翻訳だけ実行して ② はスキップする |

`L3-1` と `L4-2` の②は、差し替えでは回避できません。→ 次の項目へ

---

## 1-3. 画像生成・動画生成が使えないとき（L3-1 / L3-2）

上記のレッスンは、最下位ティアだと**モデルを差し替えても実行できません**（`L3-2` の `sora-2` は退役済みなので、ティアに関係なく実行できません。動画で学んでください）。取れる道は2つです。

**① 動画を見て学ぶ（すぐできる）**

実践レクチャーの動画では、パラメータの意味、リクエストの組み立て方、返ってきた結果の読み方、非同期ジョブの扱い方まで画面で追えます。**手を動かせなくても、試験対策としては十分に機能します**。試験で問われるのは「どのモデル・どのパラメータを選ぶか」「結果をどう扱うか」という判断であって、実際にデプロイできたかどうかではありません。まずは動画で流れを掴んでください。

**② クォータ増加を申請する（継続して使う予定があるなら）**

今後も画像生成や動画生成を使う見込みがあるなら、申請しておく価値があります。

- フォーム： https://aka.ms/oai/stuquotarequest
- 記入するもの：サブスクリプションID・リージョン・モデル名（`gpt-image-2` など）・希望 TPM
- 必要な権限：サブスクリプションの **Owner** または **Contributor**
- コツ：申請は「**既存のクォータを実際に使っている顧客**」が優先されます。先に他のレッスンで実際に推論を回してから申請すると通りやすくなります
- 動画生成の `sora-2`（2025-12-08）は **2026-10-15 に退役済み（後継なし）**です。申請の前に、カタログと [モデルの廃止スケジュール](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirement-schedule) で、使えるモデルが残っているかを確認してください

なお、ティアは消費実績と支払い履歴で自動的に上がります。講座を順に進めていれば、後半のレッスンに着手する頃には使えるようになっている可能性もあります。

---

## 2. `Quota exceeded`（割り当てはあるが空きが無い）

1 と違い、**Quota ページに行はあって残りが 0** の状態です。**Manage → Quota** で他のデプロイに割り当てた TPM を減らして空きを作るか、増加申請をしてください。反映まで最大15分かかります。

---

## 3. `429 Too Many Requests`

1分あたりのトークン数（TPM）またはリクエスト数（RPM）の上限を超えています。**トークンに余裕があるのに 429 になる**こともあります（RPM 側の上限、または一時的な容量逼迫）。リトライを指数バックオフで実装し、`retry-after` ヘッダーの値に従ってください。詳細はクォータ管理のレクチャーで扱います。

---

## 4. `401`（認証エラー）／`DefaultAzureCredential failed to retrieve a token`

`az login` が済んでいないか、トークンが期限切れです。`az login` をやり直し、`az account show` で目的のサブスクリプションが選ばれているか確認してください。

`DefaultAzureCredential failed to retrieve a token from the included credentials` と表示され、環境変数・マネージドID・VS Code・Azure CLI・Azure PowerShell などの資格情報が**すべて失敗した**と出る場合も、原因は同じで、Azure CLI にサインインしていません。

```powershell
az version     # Azure CLI が入っているか確認（無ければインストールしてターミナルを開き直す）
az login       # ブラウザでサインイン
az account show --query "{subscription:name, state:state}" -o table
```

- **Codespaces の場合や、ブラウザが開かない場合**は `az login --use-device-code` を使います。表示された URL（https://microsoft.com/devicelogin ）を手元のブラウザで開き、表示されたコードを入力します。
- 目的のサブスクリプションが出ない場合は、`az login --tenant <テナントID>` でテナントを指定するか、`az account set --subscription "<名前またはID>"` で切り替えます。
- 詳しい手順は [Hello, Foundry の README の手順0](00_intro/L0-3_hello_foundry/README.md#0-azure-cli-にサインインする初回だけ) にあります。
- 本講座の SDK（`azure-ai-projects` 2.x）は Entra ID 認証にのみ対応しているため、API キーに切り替えて回避することはできません。

## 5. `403`（権限エラー）

プロジェクトに対するロールが足りません。必要な最小ロールは **Foundry User**（以前の名前は Azure AI User。ロールIDと権限は同じです）。自分で作ったプロジェクトなら通常は権限がありますが、他の人が作ったプロジェクトでは割り当てが必要です。

## 6. `404` / モデルが見つからない

`.env` のモデル名が**実際のデプロイ名**と一致しているか確認してください。コードの `model=` に渡すのは**カタログ上のモデル名ではなくデプロイ名**です。あわせてエンドポイントの形式も確認してください（次項）。

## 7. エンドポイントの形式を間違えている

`PROJECT_ENDPOINT` は次の形です。

```
https://<リソース名>.services.ai.azure.com/api/projects/<プロジェクト名>
```

末尾が `/api/projects/<プロジェクト名>` になっている点が重要です。モデルを直接呼ぶためのエンドポイントとは別物で、見た目が似ているので取り違えやすいところです。

## 8. `azure-ai-projects` のバージョン不一致

本講座は **2.x**（新しい Foundry プロジェクト）を前提にしています。1.x（classic / Hub）のコード例とは互換性がありません。`pip show azure-ai-projects` でバージョンを確認してください。レッスンごとに仮想環境を分けると安全です。

## 9. Foundry リソースの作成自体が `InsufficientQuota` で失敗する

これはモデルの TPM とは**別枠のクォータ**です。無料試用版や Azure for Students のサブスクリプションで起きやすく、**従量課金（Pay-As-You-Go）への切り替え**か、サポートリクエストが必要になります。本講座は従量課金のサブスクリプションを推奨します。

## 10. `.env` が読み込まれない／コミットしてしまいそう

`.env.sample` をコピーして `.env` を作り、**同じフォルダで**実行してください。`.env` は `.gitignore` で除外済みなので、コミットされません。**秘密情報をリポジトリに入れない**のは基本の作法なので、除外設定を外さないでください。

## 11. モデルIDが古い／廃止されている

このカリキュラムが扱う領域はモデルの更新が非常に速く、教材に書かれたモデルIDが古くなることがあります。デプロイの前に **Foundry のモデルカタログで現行のID**を確認してください。廃止スケジュールは [Model retirement schedule](https://learn.microsoft.com/azure/foundry/openai/concepts/model-retirement-schedule) にあります。

## 12. コストが心配

各レッスンの推論は数円程度です。**従量課金のデプロイは呼び出さなければ基本的に課金されません**ので、講座を続けて進めるならデプロイは残して再利用するほうが手間がかかりません。学習を中断するときは、リソースグループごと削除するのが確実です。

```powershell
az group delete --name <リソースグループ名> --yes --no-wait
```

---

## それでも解決しないとき

各レッスンの `README.md` の「つまずきポイント」を確認し、エラーメッセージ全文を添えて講座のQ&Aで質問してください。エラー番号（401 / 403 / 404 / 429）と、Quota ページに対象モデルの**行があるかどうか**を書き添えてもらえると、原因の切り分けが早くなります。
