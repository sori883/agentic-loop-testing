---
name: loop-engineering
description: Coding agent のループを設計・実行するための Skill。作業発見、作業単位の切り出し、maker/checker 分離、検証、linear-memory などの memory backend への永続状態委譲、次アクション判断を扱う。Codex が反復可能な agent loop を動かす、automation prompt を準備する、triage/explorer/implementer/verifier を協調させる、または memory backend に依存しない汎用 loop 挙動を保つ必要があるときに使う。
---

# Loop Engineering

## 概要

この Skill は、単発プロンプトではなく、小さく反復可能な coding-agent loop を動かすために使う。Loop 自体は汎用に保ち、作業、検証、エスカレーションを制御し、永続状態の読み書きは設定された memory backend に委譲する。

## Loop Contract

現在の依頼を、次のフィールドを持つ bounded work item として扱う。

```md
## Goal
## Context
## Constraints
## Acceptance Criteria
## Verification
## Escalation
```

不足しているフィールドがあれば、ローカル文脈から安全な最小値を推定する。欠落が実行リスクや実装方針の分岐につながる場合だけ、人間に確認する。

## Controller Workflow

1. 依頼を bounded work item に正規化する。
2. 永続状態を読む・書く前に、設定された memory backend skill を読み込む。Linear が backend の場合は `$linear-memory` を使う。
3. 非自明な作業では maker/checker を分離する。
   - `triage` は候補作業の優先度とスコープを整理する。
   - `explorer` はコード、依存関係、既存パターンを調査する。
   - `implementer` は範囲を絞って変更する。
   - `verifier` は Goal と Acceptance Criteria に照らして確認する。
4. 実装を伴う作業では `$tdd-cycle` を使い、Plan、Memory、Red、Green、Refactor、Verify の順に進める。
5. 各工程後に `$cross-review-gates` を使い、担当 agent とは別 agent で review gate を通す。
6. 各 agent が何を読み書きできるかは memory backend skill に従う。
7. 検証と review gate が通る、人間が明示的に停止する、またはエスカレーションが必要になるまで進める。

## Memory Backend

backend 固有の memory rule はこの Skill に書かない。

- Linear が設定されている場合は、Linear Document、Issue、Comment の読み取りと、実行ログ、検証、エスカレーションの記録に `$linear-memory` を使う。
- Linear の進捗状態は `$linear-memory` の State Machine に従って status / label / Comment に反映する。
- GitHub の branch、PR、CI、review comment を扱う場合は `$github-workflow` を使う。
- 実装を伴う場合は `$tdd-cycle` を使い、計画、E2E/受け入れテスト、最小実装、テスト通過、リファクタリングのサイクルを守る。
- 設計、テスト、実装、リファクタリング、最終検証の各工程後は `$cross-review-gates` を使い、担当 agent とは別 agent にクロスレビューさせる。
- 別の backend が設定されている場合は、その backend 用 Skill を使う。
- backend が設定されていない場合は、永続 memory があるふりをしない。最終応答に結果を要約し、永続化していないことを明記する。

## Verification Rules

主観的な完了宣言ではなく、検証可能な停止条件を優先する。

- 既存テスト、lint、type check、build、screenshot、対象を絞った手動確認など、状況に合う検証を使う。
- 検証手段がない場合は、最小限有用な確認を行い、その限界を記録する。
- 意味のある変更では、implementer だけを成功判定者にしない。
- verifier が `REJECT` した場合は、修正して再検証するか、明確な理由とともにエスカレーションする。
- 各工程後の reviewer が `REJECT` した場合は、次工程へ進まず、対象 phase を修正して再レビューする。
- memory backend がある場合は、検証結果をそこに記録する。

## Escalation Rules

Loop が安全に進められない場合はエスカレーションする。

- Goal がプロジェクト制約と衝突している。
- 必要な認証、承認、外部状態が不足している。
- 次アクションが破壊的または高リスクである。
- Acceptance Criteria が曖昧で、実装方針が変わり得る。
- 利用可能な tool では信頼できる検証ができない。

エスカレーションでは、具体的な質問と、人間に必要な判断を明示する。memory backend がある場合は、エスカレーションもそこに記録する。

## Automation Prompt Shape

Automation prompt は長い手順を埋め込まず、この Skill を呼ぶ短い形にする。

```text
Use $loop-engineering.
Use $linear-memory when Linear is the memory backend.
Use $github-workflow when touching GitHub branches, PRs, CI, or review comments.
Use $tdd-cycle when implementation is required.
Use $cross-review-gates after design, test, implementation, refactor, and final verification phases.
Discover or accept one bounded work item.
Use maker/checker separation when work is non-trivial.
Verify before completion.
Persist durable state through the configured memory backend.
```
