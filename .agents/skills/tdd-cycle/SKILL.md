---
name: tdd-cycle
description: t-wada 流の TDD サイクルを coding-agent loop に適用するための Skill。Codex または subagent が計画、テストリスト作成、E2E/受け入れテストの追加、失敗確認、最小実装、テスト通過、リファクタリング、検証記録を順に行う必要があるときに使う。Linear を memory backend とする場合は、計画、テスト意図、検証結果を linear-memory に記録する。
---

# TDD Cycle

## 概要

t-wada 流の TDD を loop execution の実装サイクルとして使う。基本は小さなステップで、`Red -> Green -> Refactor` を崩さない。

## Core Rule

実装を始める前に、まずテストで期待する振る舞いを表現する。

標準サイクル:

1. Plan: Goal、Acceptance Criteria、テスト戦略、最小スコープを決める。
2. Memory: Linear が backend の場合、Plan と Test List を `$linear-memory` で記録する。
3. Red: まず E2E または受け入れテストを書き、期待通りに失敗することを確認する。
4. Green: 失敗しているテストを通すための最小実装を行う。
5. Refactor: テストが通った状態を保ったまま重複、命名、構造を整える。
6. Verify: E2E、関連 unit/integration、lint、type check、build など必要な検証を通す。
7. Memory: 実装結果、検証結果、残るリスクを `$linear-memory` に記録する。

## Test List

作業前に小さな Test List を作る。

```md
## TDD Plan

Goal:
Acceptance Criteria:

## Test List
- [ ] E2E/acceptance:
- [ ] Edge case:
- [ ] Regression:

## First Red Test

## Verification Command
```

Test List は完璧でなくてよい。まず一番重要な振る舞いを 1 つ選び、Red に進む。

## E2E First

ユーザーに見える振る舞いや workflow の変更では、最初に E2E または受け入れテストを書く。

- UI がある場合は、ユーザー操作と表示結果をテストする。
- API がある場合は、外部から見える request/response または contract をテストする。
- CLI がある場合は、コマンド実行と出力/終了コードをテストする。
- E2E が過剰または実行不能な場合は、最も外側に近い integration/acceptance test を選び、その理由を memory に記録する。

E2E だけで細部を駆動しづらい場合は、E2E の Red を確認した後、必要な unit/integration test を追加して小さく進める。

## Red

- production code を先に大きく書かない。
- 失敗理由が期待したものか確認する。
- test harness が存在しない場合は、最小限の harness を追加し、その追加自体を小さく検証する。
- 失敗しないテストは Red ではない。テスト条件を見直す。

## Green

- 失敗しているテストを通す最小コードを書く。
- 先回りした汎用化や大きな設計変更をしない。
- 必要なら `fake it`、`obvious implementation`、`triangulation` の順に小さく進める。
- Green になったら、実行したコマンドと結果を記録する。

## Refactor

Refactor は Green の後にだけ行う。

- 外部振る舞いを変えない。
- テストを通したまま重複、名前、責務、構造を整える。
- Refactor 後に同じ検証を再実行する。
- Refactor が大きくなりそうなら別 work item に分ける。

## Memory With Linear

Linear が memory backend の場合は `$linear-memory` を使う。

最初に記録するもの:

- TDD Plan
- Test List
- 最初に書く E2E/acceptance test
- 実行予定の verification command

最後に記録するもの:

- Red の確認結果
- Green の確認結果
- Refactor の有無
- 最終 verification
- 残る risk または follow-up

## Stop Conditions

完了条件:

- Acceptance Criteria を表す E2E/acceptance test が通っている。
- 関連する unit/integration test が通っている。
- 必要な lint/type check/build が通っている、または未実行理由が明確に記録されている。
- Refactor 後も検証が通っている。
- Linear memory がある場合は、Plan と結果が記録されている。

エスカレーション条件:

- Acceptance Criteria をテストに落とせない。
- E2E 環境、認証、データ、外部サービスが不足している。
- テスト追加が大規模な基盤変更を要求する。
- Red の失敗理由が期待と違い、仕様判断が必要である。
