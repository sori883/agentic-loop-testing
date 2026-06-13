---
name: cross-review-gates
description: 各工程後に、担当した agent とは別の agent で独立レビューを行うための Skill。設計、テスト、実装、リファクタリング、最終検証の各 phase で、成果物だけを対象に cross-check し、APPROVE / REJECT / ESCALATE を判定する必要があるときに使う。t-wada 流 TDD、Loop Engineering、Linear memory と組み合わせて使う。
---

# Cross Review Gates

## 概要

各工程の完了後に、担当 agent とは別の read-only reviewer agent を起動してクロスチェックする。レビューは改善作業ではなく、次の工程に進んでよいかを判定する gate として扱う。

## Core Rule

- 工程を実行した agent と、その工程をレビューする agent は同一にしない。
- Reviewer は新しい subagent として起動する。
- Reviewer には成果物、Goal、Acceptance Criteria、関連 diff、test output、Linear memory など、判定に必要な evidence だけを渡す。
- Reviewer に actor の chain-of-thought や隠れた意図を渡さない。
- Reviewer は artifact を変更しない。判定と根拠だけを返す。
- `APPROVE` なしに次工程へ進まない。

## Review Gates

標準 gate:

1. Design Review: Plan と Test List が Goal / Acceptance Criteria を満たすか確認する。
2. Test Review: 追加した E2E/受け入れテストが仕様を表しており、期待通り Red になっているか確認する。
3. Implementation Review: Green のための実装が最小で、テストを不正に弱めていないか確認する。
4. Refactor Review: 振る舞いを変えずに整理され、テストが通ったままか確認する。
5. Verification Review: 最終検証が十分で、残リスクが明示されているか確認する。

## Reviewer Output

Reviewer は次の形式で返す。

```md
## Phase Review

Phase:
Actor:
Reviewer:
Verdict: APPROVE | REJECT | ESCALATE

## Evidence

## Failed Criteria

## Required Next Action

## Residual Risk
```

`APPROVE` は、次工程へ進んでよい場合だけ使う。  
`REJECT` は、actor が修正して同じ gate を再実行できる場合に使う。  
`ESCALATE` は、仕様判断、権限、外部状態、人間の承認が必要な場合に使う。

## Memory

Linear が memory backend の場合は `$linear-memory` を使い、各 review gate の結果を Linear Comment に記録する。

記録する内容:

- Phase
- Actor
- Reviewer
- Verdict
- Evidence
- Required Next Action
- Residual Risk

## Stop Conditions

次工程へ進める条件:

- 対象 phase の Reviewer が `APPROVE` を返している。
- `REJECT` の場合は、actor が修正し、同じ phase を再レビューしている。
- `ESCALATE` の場合は、人間の判断が Linear memory または現在の会話に記録されている。

レビューできない場合は、レビューを省略せず `ESCALATE` として扱う。
