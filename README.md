# xbrl-align

EDINET XBRLタクソノミの勘定科目を、会計基準(J-GAAP/IFRS)・業種(全23業種)・年度をまたいで名寄せし、`標準キー → [対応するQName]` の辞書を生成するツール。

2つのパイプラインを提供:

| | Original (`main.py`) | LLM-first (`main_llm.py`) |
|---|---|---|
| マッチング | ラベルマッチ + LLM補完 | **LLM全件マッチング** |
| ガードレール | 5種 | **8種** |
| jppfs起点 | CK.yamlのqnameで固定 | LLMが選択 + qnameで検証 |
| 定義ファイル | `CK.yaml` | `CK_llm.yaml`（注記付き） |
| 出力先 | `output/` | `output_llm/` |

## セットアップ

```bash
uv sync
cp .env.example .env  # GEMINI_API_KEY を設定
```

## Original パイプライン（ラベルマッチ + LLM補完）

```bash
# 全年度
uv run python main.py align

# LLMなし（ラベルマッチのみ）
uv run python main.py align --no-llm

# 特定年度
uv run python main.py align --years 2026 2025
```

出力: `output/{taxonomy_date}_alignment.json`

## LLM-first パイプライン

```bash
# 全年度
uv run python main_llm.py

# 特定年度
uv run python main_llm.py --years 2026
```

出力: `output_llm/{taxonomy_date}_alignment.json`

## 辞書のフォーマット（共通）

```json
{
  "NetSales": {
    "ja_name": "売上高",
    "period_type": "duration",
    "representative": ["jppfs_cor:NetSales", "jpigp_cor:NetSalesIFRS", "jpigp_cor:RevenueIFRS"],
    "variants": ["jppfs_cor:IncidentalBusinessesSalesGAS", ...]
  },
  "CashAndCashEquivalents:periodStartLabel": {
    "ja_name": "現金及び現金同等物の期首残高",
    "period_type": "instant",
    "label_role": "periodStartLabel",
    "representative": ["jppfs_cor:CashAndCashEquivalents"]
  }
}
```

| フィールド | 意味 |
|-----------|------|
| `representative` | 合計値を示す代表タグ。優先順（JGAAP→IFRS）。まずここから探す |
| `variants` | 業種固有の同義タグ（GS chain由来）。representativeで見つからない時のフォールバック |
| `period_type` | `"instant"`(BS) / `"duration"`(PL/CF)。contextRefの期間フィルタに使う |
| `label_role` | 期首/期末の区別。`"periodStartLabel"` or `"periodEndLabel"` |
| `ja_name` | 日本語名 |

### キー命名規則

- 通常: qnameそのまま（例: `NetSales`, `Assets`）
- 期首/期末: `qname:label_role`（例: `CashAndCashEquivalents:periodStartLabel`）

## XBRLデータからの値抽出

```python
import json

with open("output_llm/2025-11-01_alignment.json") as f:
    alignment = json.load(f)["standard_keys"]

def extract_value(facts, key, context_filter):
    """
    facts: XBRLパース済みのfactリスト
    key: 標準キー名（例: "NetSales", "CashAndCashEquivalents:periodEndLabel"）
    context_filter: contextRefの判定関数（当期・連結かどうか等）
    """
    entry = alignment[key]

    # Step 1: representative を順番に探す（合計値が取れる）
    for qname in entry["representative"]:
        for fact in facts:
            if fact.qname == qname and context_filter(fact.context):
                return fact.value

    # Step 2: variants にフォールバック（業種固有の場合）
    variant_set = set(entry["variants"])
    for fact in facts:
        if fact.qname in variant_set and context_filter(fact.context):
            return fact.value

    return None

# PL科目: 当期・連結のduration contextで取る
revenue = extract_value(facts, "NetSales", is_current_consolidated)

# BS科目: 期末日のinstant contextで取る
total_assets = extract_value(facts, "Assets", is_current_period_end)

# CF期首/期末: label_roleで区別
cash_end = extract_value(facts, "CashAndCashEquivalents:periodEndLabel", is_current_period_end)
cash_start = extract_value(facts, "CashAndCashEquivalents:periodStartLabel", is_prior_period_end)
```

**注意:**
- `context_filter` で「当期・連結」を絞ること（XBRLには前期・単体の値も混在する）
- BS科目(`period_type: "instant"`)は期末日、PL/CF科目(`"duration"`)は通期のcontextで取る
- `label_role` がある科目は期首/期末で異なるcontextを使い分けること
- `representative` を先に探すことで、業種固有の内訳タグを誤取得する事故を防げる

## 年度間比較

```bash
# Originalパイプラインの出力を比較
uv run python -m src.xbrl_align.compare

# LLM-firstパイプラインの出力を比較
uv run python -m src.xbrl_align.compare --output-dir output_llm

# 特定キーの年度推移
uv run python -m src.xbrl_align.compare --keys NetSales Inventories

# 差分のみ
uv run python -m src.xbrl_align.compare --diff-only
```

## IFRS同義語の検証

```bash
uv run python scripts/validate_ifrs_synonyms.py output_llm/2025-11-01_alignment.json
```

## 標準キーの追加・変更

- Original: `CK.yaml` を編集 → `uv run python main.py align`
- LLM-first: `CK_llm.yaml` を編集（注記を追加可） → `uv run python main_llm.py`

## パイプラインの仕組み

### Original (`main.py`)
```
CK.yaml の qname を起点に:
  ├─ ラベルマッチ（日本語/英語完全一致）でIFRS概念を確定
  ├─ 未マッチ分をLLMで解決（追加同義語も収集）
  ├─ ガードレール（5種）で検証
  └─ GS foldでjppfs業種バリアントを展開（5段階フィルタ）
```

### LLM-first (`main_llm.py`)
```
CK_llm.yaml の全キー + jppfs/jpigp全概念リストをLLMに1-shot:
  ├─ LLMが各キーの jppfs代表概念 + jpigp候補(複数) を返す
  ├─ ガードレール（8種）で全結果を検証・棄却
  └─ jppfs代表概念からGS foldでvariantsを展開（5段階フィルタ）
```

ガードレール一覧（LLM-first）:

| # | 検証 |
|---|------|
| 1 | 概念がタクソノミに実在するか |
| 2 | abstract概念でないか |
| 3 | periodType一致 |
| 4 | balance一致 |
| 5 | calc子の排他（別キーに割当済みの概念のcalc子でないか） |
| 6 | jppfs qname検証（CK.yamlのqnameと一致するか） |
| 7 | 重複排他（同一jpigp概念がexactで複数キーに入らないか） |
| 8 | confidence閾値 (< 0.7 を棄却) |

## モデル変更

`src/xbrl_align/config.py` または `src/xbrl_align_llm/config.py` の `GEMINI_MODEL` を変更:

```python
GEMINI_MODEL = "gemini-3.1-pro-preview"
```

## プロジェクト構成

```
CK.yaml                        # 標準キー定義（Originalパイプライン用）
CK_llm.yaml                    # 標準キー定義（LLM-first用、注記付き）
main.py                        # Original CLIエントリポイント
main_llm.py                    # LLM-first CLIエントリポイント
output/                        # Original 出力
output_llm/                    # LLM-first 出力
scripts/
  validate_ifrs_synonyms.py    # IFRS同義語検証
  compare_linkbases.py         # リンクベース比較（開発用）
  hub_analysis.py              # ハブ分析（開発用）
src/xbrl_align/                # Original パイプライン
  config.py
  models.py
  ck_loader.py
  compare.py                   # 年度間比較ツール
  parse/
    taxonomy_loader.py         # XSD+ラベル+リンクベース読込
    gs_loader.py               # GS アーク収集（全23業種）
    deprecated_loader.py       # 廃止概念読込
  fold/
    gs_folder.py               # GS chain折り畳み
    concept_index.py           # 概念索引構築
  match/
    pipeline.py                # ラベルマッチ + LLM補完
    label_seed.py              # ラベル完全一致
    hard_constraint.py         # periodType/balance フィルタ
    calc_propagation.py        # 計算リンクベース伝播
    llm_resolver.py            # Gemini LLM解決
  output/writer.py
src/xbrl_align_llm/            # LLM-first パイプライン
  config.py
  ck_loader.py                 # 注記フィールド対応
  llm_matcher.py               # LLM全件マッチング
  guardrails.py                # 8種のガードレール
  pipeline.py                  # 統合（LLM→ガードレール→GS fold）
  writer.py
```
