# mahjong-experiments

この repo は、麻雀強化学習プロジェクトの実験計画 (`runbook.md`) と結果 (`report.md`) を管理する場所である。

本体コードは `../mahjong`、旧 Stage01 / Stage02 の参照実装は `../majong-rl` にある。
ここには、実験の意図・条件・結果・解釈だけを残す。

## ディレクトリ構成

実験は stage ごとに分け、その下に個別実験を置く。

```text
mahjong-experiments/
  README.md
  Stage01_DiscardOnly/
    exp_001/
      runbook.md
      report.md
    ...
  Stage02_CallUnlock/
    exp_001/
      runbook.md
      report.md
    ...
  Stage03_RiichiEnvSelfPlay/
    exp_001/
      runbook.md
      report.md
```

各 stage 直下に `README.md` は原則置かない。
stage 全体の現状はこの root README にまとめ、詳細は個別 `exp_XXX` の `runbook.md` / `report.md` に閉じる。

## 運用ルール

- `report.md` を作る前に、必ず対応する `runbook.md` を作成する。
- `runbook.md` がない実験の `report.md` は作成しない。
- 実験条件・比較軸・成功/失敗基準は `runbook.md` に書く。
- 実行結果・解釈・次アクションは `report.md` に書く。
- 生ログ・checkpoint・大量の中間生成物は Git 管理しない。
- 必要な場合のみ `run_map.json` を置き、削除済み run の対応関係を追跡する。
- bugfix 系の実験では、必要に応じて `bug_report.md` を置く。

Git 管理するもの:

- `README.md`
- `StageXX_*/exp_XXX/runbook.md`
- `StageXX_*/exp_XXX/report.md`
- `StageXX_*/exp_XXX/run_map.json`（必要時のみ）
- `StageXX_*/exp_XXX/bug_report.md`（必要時のみ）

Git 管理しないもの:

- `runs/`
- `driver_logs/`
- checkpoint / shard / raw metrics などの大量成果物
- 一時集計ファイル

## Stage 一覧

### Stage01_DiscardOnly

旧 engine 上で、まず自摸直後の打牌だけを学習対象にした段階。

主目的:

- imitation / PPO の基礎パイプライン構築
- reward / advantage / worker / evaluator の不具合洗い出し
- discard-only 条件で PPO が imitation を超えられるか確認
- 防御特徴量の headroom 確認

主な結論:

- PPO 自体は壊れていなかった。
- corrected semantics 後は、特徴量を与えれば PPO は imitation を上回れる。
- `danger_mask` などの防御特徴量は非常に効いた。
- Stage01 は現在の主戦場ではなく、regression harness / 上限比較用の基準系として残す。

代表的な参照先:

- `Stage01_DiscardOnly/exp_065_bugfix/report.md`
- `Stage01_DiscardOnly/exp_068/report.md`
- `Stage01_DiscardOnly/exp_069/report.md`
- `Stage01_DiscardOnly/exp_070/report.md`

### Stage02_CallUnlock

旧 engine 上で、discard-only から副露・応答行動を解禁した段階。

主目的:

- Chi / Pon / Daiminkan / Skip など optional action を学習対象に入れる
- discard policy と call / candidate policy の分離を検証する
- Stage01 で得た PPO stabilizer を、より広い行動空間で検証する
- semantic auxiliary / target KL / family diagnostics などを整える

主な結論:

- 行動空間を広げても、diagnostics と stabilizer を入れれば学習基盤は維持できる。
- Stage02 で有効だった要素は、Stage03 に移植する価値がある。
- ただし旧 engine 固有の設計も多く、新しい RiichiEnv にはそのまま持ち込まない。

Stage03 に引き継いだ主な要素:

- shanten / ukeire teacher
- tie-aware imitation
- rule-based call policy heuristics
- terminal / yaku auxiliary target
- semantic summary injection
- target KL early stop
- entropy bonus
- decision family diagnostics
- actor type exclusion
- lr groups
- per-player-round weighting
- post-riichi exclusion
- shard-level audit

### Stage03_RiichiEnvSelfPlay

PyPI 版 `riichienv` を engine として使う新しい段階。

主目的は、**RiichiEnv 上で public-only な self-play 学習ループを成立させること**。
強さの最大化より先に、model rollout / imitation / PPO / evaluation / diagnostics が end-to-end で破綻しないことを確認する。

現時点のコード状態:

- `riichienv` adapter 実装済み
- model-facing action abstraction / resolver 実装済み
- `RIICHI_DISCARD` の 2-step action 解決済み
- public-only observation encoder 実装済み
- encoder v2 hints 実装済み
  - shanten
  - discard-after shanten delta
  - ukeire
  - remaining draws / turn progress
  - tile presence flags
- Stage02 系統 rule-based baseline 実装済み
  - shanten 最小化
  - ukeire 最大化
  - call policy heuristics
- `teacher_best_mask` schema v2 実装済み
- tie-aware imitation loss / accuracy 実装済み
- Stage03 multi-head model 実装済み
- `ModelPolicyAgent` 実装済み
- rollout-time `old_log_prob` / `value` collection 実装済み
- self-play / evaluation loop 実装済み
- all-round yaku / han / fu backfill via `MjaiReplay` 実装済み
- PPO learner 実装済み
- PPO combined log-prob formulation 実装済み
- target KL / entropy / gradient diagnostics 実装済み
- lr groups 実装済み
- per-player-round weighting 実装済み
- post-riichi exclusion 実装済み
- semantic summary injection opt-in 実装済み
- shard / sample audit diagnostics 実装済み

現時点では、**初回 smoke experiment はまだ未実行**。
したがって Stage03 の current best はまだ無い。

#### Stage03 の重要な設計差分

Stage03 は Stage02 の単純な移植ではない。
`riichienv` の action structure に合わせて、いくつかの設計を変えている。

1. **Combined policy distribution**

   Stage02 は decision type が engine 側で分かれていたため、discard と optional action を branch-local softmax として扱えた。

   Stage03 では、1 decision point で通常打牌と candidate action が同時に legal になり得る。
   そのため `ModelPolicyAgent` と PPO は、次の combined softmax 上で action probability / PPO ratio / KL を計算する。

   ```text
   [discard logits 34, candidate scores C]
   ```

   Stage02 の separated PPO update はそのまま移植していない。
   現状は combined distribution のまま、family 別 diagnostics で観測する。

2. **Riichi discard handling**

   `riichienv` では立直宣言と打牌が 2-step になっている。
   Stage03 では resolver 側で `RIICHI_DISCARD` candidate として model-facing action にまとめる。

   旧 Stage02 のような `riichi_discard_mask` feature は、RiichiEnv では dead feature になったため削除済み。

3. **Yaku target capture**

   `riichienv` の `env.win_results` は mid-game round では Python 側から読めない。
   Stage03 では game 完走後に `MjaiReplay` で `mjai_log` を replay し、全 round の winner yaku / han / fu を backfill する。

   `mjai_log` には初手配など hidden payload も含まれるが、Stage03 では和了後に公開される label の復元にのみ使い、model input / sample metadata には入れない。

#### 現時点の Stage03 schema / dim

- default encoder observation dim: `440`
- legacy encoder observation dim: `363` (`enable_hints=False`)
- `DecisionSample` schema version: `2`
- `teacher_best_mask`: `(34,) float32`

既存 checkpoint / shard との互換性は重視しない段階。
不一致は fail-fast で検出する。

#### 既知の制約

- shanten / ukeire hint は Python 実装で重い。
  - encoder hints on で 1 encode あたり 26〜35ms 程度かかることがある。
  - 短期は初回 smoke を優先し、必要なら LRU cache / C++ / Rust extension を検討する。
- `YON_IKKYOKU` では deterministic wall による strict determinism が成立する。
  - `YON_TONPUSEN` 以上では engine 制約により round 2 以降の strict determinism は限定的。
- opponent threat features は未実装。
  - `danger_mask` / opponent tenpai 推定は、初回 smoke 後の deal-in 系 metrics を見て判断する。
- riichi.dev connector は未実装。
  - offline self-play で一定の強さと安定性を確認してから扱う。

#### 次にやること

最初の実験は `Stage03_RiichiEnvSelfPlay/exp_001` とする。

目的は強さの評価ではなく、以下の pipeline validation。

- baseline / model rollout が crash しない
- imitation warm start が走る
- PPO update が走る
- `old_log_prob` / new log-prob の整合が崩れない
- yaku / terminal target が入っている
- family counts / actor counts / teacher diagnostics が読める
- shard audit が動く
- hidden info leak の兆候がない
- 実行時間が現実的な範囲に収まる

## Current Best

### Stage01

Stage01 の current best は `Stage01_DiscardOnly/exp_070` の `context_plus_danger` 系。
ただしこれは旧 engine / discard-only 条件の基準系であり、Stage03 と直接比較するものではない。

### Stage02

Stage02 は Stage03 へ移行するための知見抽出段階として扱う。
現時点では Stage03 へ引き継ぐべき stabilizer / diagnostics の参照元。

### Stage03

まだ無し。

`Stage03_RiichiEnvSelfPlay/exp_001` 実行後に、まずは current best ではなく
「初回 smoke が成立したか」を記録する。

## 読む順番

Stage03 の現状を把握したい場合:

1. この `README.md` の `Stage03_RiichiEnvSelfPlay` 節
2. `Stage03_RiichiEnvSelfPlay/exp_001/runbook.md`（作成後）
3. `Stage03_RiichiEnvSelfPlay/exp_001/report.md`（実行後）

過去 stage の流れを確認したい場合:

1. Stage01 の代表 report
2. Stage02 の関連 report
3. 必要なら個別 `runbook.md` / `run_map.json`
