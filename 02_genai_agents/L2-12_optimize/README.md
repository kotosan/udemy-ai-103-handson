# L2-12 実践: プロンプト改善 ＋ self-critique で品質を上げる

同じお題を **①baseline（素のプロンプト）→ ②改善プロンプト（役割・対象・形式を明確化）→ ③self-critique（自己批評→改稿）** の3段階で実行し、出力の変化と、そのぶん増えるコスト（呼び出し回数・トークン・時間）を見比べるハンズオンです。

> 対応レクチャー：実践 `L2-12-3`
> 認証は**キーレス**（`az login` ＋ `DefaultAzureCredential`）。Azure のリソースは新しく作りません（使ったトークン分だけ課金されます）。

## どのデプロイで何を比べるか

| 手順 | デプロイ | 見るもの |
|---|---|---|
| 6 | `gpt-5.1`（推論の強さは既定の `none`＝**推論しない**） | プロンプトの技法（役割・対象・形式の指定、self-critique）で出力がどう変わるか。**このレッスンの本題** |
| 7 | `gpt-5.1` ＋ 推論の強さ `medium`（**推論する**） | 同じ4回の呼び出しで、**推論トークン**と時間がどれだけ増えるか |
| 8 | `gpt-5.1` | `temperature` と `reasoning`（推論の強さ）が、どの組み合わせで使えて、どこで 400 になるか |

Learn の Prompt engineering techniques は、**このページの技法（few-shot・chain-of-thought など）は gpt-5 や o-series のような推論モデルには推奨しない**と明記しています。推論モデルには、目的・制約・出力の形をはっきり伝える、という別の指針（Azure OpenAI reasoning models の Prompting guidance）があります。

`gpt-5.1` も推論モデルですが、**推論の強さ（reasoning effort）の既定が `none`** で、指定しなければ推論せずに答えます（Learn の Azure OpenAI reasoning models に記載）。そこでこのレッスンでは、プロンプトの技法の効果を**推論しない状態の `gpt-5.1`** で見て（手順6）、同じモデルで推論を有効にしたときのコスト（手順7）と比べます。

## ファイル構成
| ファイル | 役割 |
|---|---|
| `main.py` | baseline → 改善 → 批評 → 改稿の4回呼び出し。最後に段階ごとの字数・出力トークン（うち推論）・秒を一覧にする。引数でデプロイ名と推論の強さを差し替えられる |
| `param_check.py` | `temperature` と `reasoning` を付けて短い質問を送り、通るか 400 になるかを表示する |
| `.env.sample` | 環境変数の雛形 |
| `requirements.txt` | Python 依存パッケージ（`azure-ai-projects>=2.0.0` は 1.x と非互換） |
| `README.md` | このファイル（手順とコマンド） |

## 前提
- L0-3（Hello, Foundry）を終えていること（講座共通のリソースグループ `rg-ai103`・Foundry リソース・プロジェクト `ai103-project`・Foundry User ロール）。
- `az login` 済み ／ Python 3.11+
- デプロイ `gpt-5.1` があること（手順2で確かめます。無いときは L2-10 の手順2で作成）。

## 進め方（コピペで実行できます）

全部で9手順です。各手順のコードブロックを、そのままターミナルに貼り付けて実行します。手順1で決めた変数を後の手順で使うので、**同じターミナルで続けて**実行してください。

| 手順 | やること |
|---|---|
| 1 | 共通リソースの名前を変数に入れる |
| 2 | 使うデプロイがあるか確かめる |
| 3 | プロジェクトのエンドポイントを取得する |
| 4 | 仮想環境を作って依存を入れる |
| 5 | `.env` を用意する |
| 6 | 推論しない設定で、3段階を比べる |
| 7 | 推論を有効にして、同じ4回の呼び出しのコストを見る |
| 8 | 使えるパラメーターの違いを確かめる |
| 9 | 後片付け |

> コマンドは **PowerShell** 用です（Codespaces のターミナルで `pwsh` を選ぶ／Windows の PowerShell／Mac・Linux は PowerShell 7 を入れて `pwsh`）。行末の `` ` `` は行の継続です。
>
> 最初に、リポジトリのルートからこのフォルダーへ移動しておきます。
> ```powershell
> cd 02_genai_agents/L2-12_optimize
> ```

### 1. 共通リソースの名前を変数に入れる
```powershell
$RG = "rg-ai103"
$FOUNDRY = az cognitiveservices account list --resource-group $RG --query "[?kind=='AIServices'] | [0].name" -o tsv
$FOUNDRY
```
`ai103-foundry-<数字>` が表示されればOKです。

### 2. 使うデプロイがあるか確かめる
```powershell
az cognitiveservices account deployment list `
  --name $FOUNDRY `
  --resource-group $RG `
  --query "[].{name:name, model:properties.model.name, sku:sku.name}" -o table
```
一覧に `gpt-5.1` があればOKです。無いときは、L2-10 の README の手順2のコマンドで作ります（デプロイ自体は無料。課金は使ったトークン分だけ）。

### 3. プロジェクトのエンドポイントを取得する
```powershell
az cognitiveservices account project show `
  --name $FOUNDRY `
  --resource-group $RG `
  --project-name ai103-project `
  --query 'properties.endpoints."AI Foundry API"' -o tsv
```
返ってきた URL（末尾が `/api/projects/ai103-project`）を、手順5で `.env` に貼ります。

### 4. 仮想環境を作って依存を入れる
```powershell
python -m venv .venv
./.venv/bin/Activate.ps1
pip install -r requirements.txt
```
2行目は Codespaces（Linux）の PowerShell 用です。Windows の PowerShell では `.\.venv\Scripts\Activate.ps1` にします。

### 5. `.env` を用意する
```powershell
cp .env.sample .env
code .env
```
開いた `.env` の `PROJECT_ENDPOINT=` に手順3のエンドポイントを貼って保存します。`MODEL_DEPLOYMENT=gpt-5.1` は最初から入っています。

### 6. 推論しない設定で、3段階を比べる
```powershell
python main.py
```
4回呼び出すので十数秒かかります。出力は5つのブロックです（長いので、ターミナルを広げて上から読みます）。
```text
モデル（デプロイ名）: gpt-5.1

===== (1) baseline =====
 （形式も長さも指定していないので、キャッチコピー・説明・「主な特徴」の箇条書き5つ・締めの文まで付いた長い文章）

===== (2) 改善プロンプト =====
 家計簿が「続く」に変わる、新感覚アプリ
・レシート撮影だけで自動入力、3秒で記録完了
・…（見出し＋箇条書き3つ＋一言の行動喚起）

===== 批評 =====
 - 「新感覚アプリ」と言いながら、どこが「新感覚」なのか具体的に示せていない …（辛口の編集者としての弱点の指摘）

===== (3) 改善＋self-critique =====
 （批評を反映して書き直した版）

===== まとめ（gpt-5.1） =====
段階            字数  出力トークン  うち推論    秒
(1) baseline     384           293         0   5.8
(2) 改善         124           106         0   1.8
批評             642           496         0   5.1
(3) 改稿         188           158         0   1.9
合計                          1053         0  14.6  （呼び出し 4 回）
```
- **(2)** が指定した形式（見出し・箇条書き3つ・行動喚起）と分量（120字程度）に寄っているかを見ます。
- **(3)** は、批評の指摘（具体性など）が反映されたかを **(2) と見比べます**。分量が増えたり、お題に無い数字が入ったりして、悪くなることもあります。
- **まとめ**の表で、self-critique が足した「批評」「改稿」の2回ぶんのトークンと時間を見ます。品質を上げるほどコストも増えます。
- 「うち推論」は 0 です。推論の強さを指定していないので、`gpt-5.1` は既定の `none` で、推論せずに答えています。
- 出力は毎回変わります。数字は実行例です。

### 7. 推論を有効にして、同じ4回の呼び出しのコストを見る
```powershell
python main.py gpt-5.1 medium
```
1つ目の引数がデプロイ名、2つ目が推論の強さ（reasoning effort）です。出力の最後の**まとめ**だけを見ます。
```text
===== まとめ（gpt-5.1 / medium） =====
段階            字数  出力トークン  うち推論    秒
(1) baseline     347           392       115   6.4
(2) 改善          96           435       350   3.8
批評            1368          1138        77   9.4
(3) 改稿         152          2141      2010  14.2
合計                          4106      2552  33.8  （呼び出し 4 回）
```
- 同じモデル・同じプロンプトでも、「うち推論」が 0 でなくなります。**推論トークンは画面には出ませんが、出力トークンとして課金されます**。同じ4回でも、トークンも時間も増えます。
- 手順6で「うち推論」が 0 だったのは、`gpt-5.1` の推論の強さの既定が `none` だからです。**既定はモデルごとに違う**（`gpt-5-mini` は `medium` が既定で、`none` を選べない）ので、推論させたいとき・させたくないときは推論の強さを明示します。

### 8. 使えるパラメーターの違いを確かめる
```powershell
python param_check.py
```
```text
OK   gpt-5.1 {'temperature': 0.2}  → 答え 2（うち推論トークン 0）
OK   gpt-5.1 {'reasoning': {'effort': 'medium'}}  → 答え 2（うち推論トークン 33）
400  gpt-5.1 {'reasoning': {'effort': 'medium'}, 'temperature': 0.2}  → Unsupported parameter: 'temperature' is not supported with this model.
400  gpt-5.1 {'reasoning': {'effort': 'minimal'}}  → Unsupported value: 'minimal' is not supported with the 'gpt-5.1-2025-11-13' model. Supported values are: 'none', 'low', 'medium', and 'high'.
```
- 1行目：推論の強さが既定の `none`（推論しない）なので、`temperature` が通ります。
- 2〜3行目：推論を有効にする（`medium`）と推論トークンが出て、`temperature` を付けると 400 になります。Learn の Azure OpenAI reasoning models も、推論モデルは `temperature`・`top_p` などを使えないとしています。**出力を調整したいときは、推論させるならサンプリングではなく、推論の強さと指示（完了の条件・出力の形）で調整します。**
- 4行目：推論の強さに使える値は**モデルごとに違います**。`minimal` は `gpt-5`・`gpt-5-mini` などの値で、`gpt-5.1` は `none`／`low`／`medium`／`high` です。エラーメッセージに使える値が出ます。

### 9. 後片付け
このレッスンで作ったリソースはありません。デプロイ `gpt-5.1` は後続のレッスンでも使うので残します。レッスンフォルダーの `.venv` と `.env` は、不要になったら消してかまいません（`.env` はコミットされません）。
```powershell
deactivate
Remove-Item -Recurse -Force .venv, .env
```

## ポイント（試験の論点）
- **プロンプト改善**：役割・対象・出力形式・分量を明確にすると、出力がそろう（推論しないモデル・推論しない設定で効く技法）。
- **self-critique**：生成 → 批評 → 改稿のループ。呼び出しが増えるので、コストと遅延も増える。**重要な回答に絞って使う**。
- **「良くなった」は評価器で裏を取る**。このコードは採点しない。L2-5 の評価器で baseline と改稿を同じ観点で採点して比べる（測って・直して・また測る）。
- **推論モデルでは技法とパラメーターが変わる**：技法ページの技法（few-shot・CoT など）は推奨されない（目的・制約・出力の形は明確に伝える）／推論を有効にしたときは `temperature`・`top_p` を使わない／推論トークンも出力トークンとして課金される／推論の強さの既定と使える値はモデルごとに違う（`gpt-5.1` は既定 `none`）。モデルや推論の強さを替えたら、プロンプトを見直して測り直す。

## つまずき
| 症状 | 対処 |
|---|---|
| `DeploymentNotFound`（404） | 手順2の一覧にそのデプロイ名があるか。`.env` の `MODEL_DEPLOYMENT`、手順7の引数と一致しているか |
| `401`/`403` | `az login` 済みか、プロジェクトに **Foundry User** ロールがあるか |
| `PROJECT_ENDPOINT が未設定です` | `.env` の作成・値の入力を確認 |
| `Unsupported value: '…' is not supported with the '…' model` | そのモデルが受け付けない推論の強さを渡している（手順8の4行目と同じ）。メッセージに出る値（`gpt-5.1` は `none`／`low`／`medium`／`high`）から選ぶ |
| `Unsupported parameter: 'temperature'` | 推論を有効にした呼び出しに `temperature` を付けている（手順8の3行目と同じ）。推論させるときは `temperature` を外す |
| (3) が (2) より悪くなった | self-critique は必ず良くなるとは限らない。批評の観点を絞る／改稿時に元の制約も渡す／評価器で比べる |
| 実行に時間がかかる | 1回の実行で4回呼び出す。推論を有効にした手順7はさらに長い（講座の検証で約35秒） |

## 後片付け
- 新しく作ったリソースはありません（手順9）。デプロイは後続のレッスンでも使うので残します。
