---
name: linear-memory
description: Linear を agent loop の永続 memory backend として使うための Skill。Codex または subagent が Linear の work item を読む、実行ログを記録する、task state を更新する、判断を保存する、人間への確認事項を Linear にエスカレーションする必要があるときに使う。chat context に依存せず、Linear を source of truth として扱う。
---

# Linear Memory

## 概要

Linear を loop state の source of truth として使う。Chat context は一時的な実行文脈であり、memory として扱わない。

## Memory Model

- Linear Document: 運用ルール、backend 固有 policy、状態遷移ルール、長期的な loop convention。
- Linear Issue: 1 つの work item。Goal、Context、Acceptance Criteria、Current State、Verification、Escalation を持つ。
- Linear Comment: agent action、調査結果、検証、人間向け質問の append-only 実行ログ。

## Read Before Work

1. ユーザー依頼、automation prompt、利用可能な config から対象の Linear Issue、Project、または discovery scope を特定する。
2. プロジェクト固有ルールを適用する前に、関連する Linear Document を読む。
3. 対象 Issue の description、status、labels、links、relations、最近の Comments を読む。
4. chat history と Linear の durable state が矛盾する場合は、最新の Linear state を優先する。
5. 必要な Issue や rule Document が見つからない場合は、不足している identifier を確認する。対象 Issue がある場合は escalation として記録する。

## Write During Work

現在の agent run を超えて残すべき情報は Linear に記録する。

- 調査メモ、実装サマリ、検証結果、blocker、人間向け質問は Comment に追記する。
- 各工程後の cross-review 結果は Comment に追記する。
- GitHub branch、commit、PR、CI run と Linear issue の紐づけは Comment に追記する。
- Issue fields の更新は、project policy または user request が許可している場合だけ行う。
- 進捗ログを書くためだけに Issue description を上書きしない。run history は Comment に残す。
- 他 agent の Comment を削除・改変しない。
- ログは事実ベースで簡潔にする。chain-of-thought や長い terminal output は、証拠として必要な場合を除いて保存しない。

## Comment Contents

各 agent Comment は、実際に発生した項目だけを含める。

```md
## Agent Run

Agent:
Action:
Result: DONE | PARTIAL | APPROVE | REJECT | ESCALATE
Evidence:
Next:
```

工程レビューを記録する場合は、次の形式を使う。

```md
## Phase Review

Phase:
Actor:
Reviewer:
Verdict: APPROVE | REJECT | ESCALATE
Evidence:
Required Next Action:
Residual Risk:
```

GitHub 連携を記録する場合は、次の形式を使う。

```md
## GitHub Linkage

Linear Issue:
Branch:
Commits:
Pull Request:
CI:
Linking Method:
Notes:
```

関連 work、branch、commit、pull request、log、artifact を参照する場合は、link または issue identifier を使う。

## Role Behavior

- Triage: 候補 Issue、priority、readiness、人間向け clarification を作成または更新する。
- Explorer: findings、関連ファイル、constraints、risks、recommended next action を記録する。
- Implementer: changed files、behavior changes、実行した verification、remaining risks を記録する。
- Verifier: Acceptance Criteria に対する evidence とともに `APPROVE`、`REJECT`、`ESCALATE` を記録する。
- Phase Reviewer: 設計、テスト、実装、リファクタリング、最終検証の各工程後に、担当 agent とは別 agent として `APPROVE`、`REJECT`、`ESCALATE` を記録する。
- GitHub Operator: branch、commit、PR、CI run、review comment を Linear issue に紐づけ、URL または identifier を記録する。

## State Changes

Linear Document または user request で状態遷移ルールが明示されていない限り、Comment を優先する。

安全な default:

- Agents は Comments を追記してよい。
- Agents は自分の作業を説明する links を追加してよい。
- project policy がその role に許可していない限り、Agents は status 移動、issue close、owner 変更をしない。
- Human escalation は、labels や statuses が未設定でも、最低限 Comment として見える状態にする。

## Final Response

Linear memory に書き込んだ後の最終応答には、次だけを含める。

- 短い結果。
- 更新した Linear Issue または Comment。
- 残る risk または人間の判断が必要な事項。
