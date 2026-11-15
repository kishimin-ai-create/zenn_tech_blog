---
title: "xUnitのテストケースをコードではなくコメントだけで設計するSkill"
emoji: "📝"
type: "tech"
topics: ["xunit", "aspnetcore", "csharp", "codex", "test"]
published: true
---

## 結論

テストケースを洗い出したいだけなのに、AIが`[Fact]`や空のテストメソッドまで作ってしまうことがある。そうなると、どこまでがレビュー対象で、どこまでが実装済みなのかが曖昧になる。

この記事では、ASP.NET Core向けのxUnitテスト設計Skillを、「コメントだけを書き、テストコードは書かない」という契約に変えた方法を書く。

対象読者は、CodexのユーザーSkillを管理していて、テスト実装前の計画を既存の`.cs`ファイルに残したい開発者である。

## 受け入れ条件を先に固定する

求められたのは、「テストコードは絶対に書かず、コメントだけを書く」ことだった。これを曖昧なおすすめではなく、Skillの制約として定義した。

書いてよいのは、`//`で始まるコメントだけ。次のものは作らないし、変えない。

- `[Fact]`と`[Theory]`
- テストメソッドとテストクラス
- fixture、fake、stub
- assertionとデータプロバイダー
- `using`とプロジェクト設定
- プロダクションコード

既存のテストコードは、設計の根拠を確かめるためと、実行するためだけに読む。コメントを足しただけのケースは、実装済みのテストともSkippedのテストとも数えない。

## コメントに実装可能な情報を残す

コードを書かないことと、曖昧なメモで済ませることは別物である。あとで実装する人が判断をやり直さなくて済むように、テスト名、前提、操作、期待結果、テスト層、優先度をコメントに残す。

```csharp
// TODO(test): PostImages_WhenRequestIsValid_ReturnsPng
// Given: a valid image request
// When: the client posts the request
// Then: the API returns PNG content with the documented metadata
// Level: ASP.NET Core integration
// Priority: High
```

仕様が決まっていないなら、期待結果は作らない。代わりにブロッカーを書く。

```csharp
// TODO(test): PostImages_WhenUpstreamTimesOut_ReturnsGatewayTimeout
// Blocked by: define the upstream timeout contract
// Expected: return the documented timeout response without internal details
// Priority: High
```

この形なら、テスト対象のWhatは残せる。実装のHowを先回りすることもない。

## Skillの発火条件にも制約を書く

Skillは、`SKILL.md`の本文を読む前に、YAML frontmatterの`description`で使うかどうかを判断する。なので、コメント専用であることを、本文だけでなくdescriptionにも書いた。

```yaml
---
name: xunit-aspnet-core-test-case-design
description: Design behavior-focused, comment-only xUnit test plans for ASP.NET Core applications without writing test code.
---
```

UI向けの`agents/openai.yaml`も、同じ契約に揃える。既定のプロンプトがただ「xUnitテストを設計する」となっていると、利用者の意図より広い生成を誘ってしまうからである。

## テスト層はコードを書かなくても決められる

コメントだけの計画でも、最小のテスト層は選べる。

| 層 | コメントで記録する境界 | 代表的な対象 |
| --- | --- | --- |
| Unit | ホストやI/Oを使わない振る舞い | Value Object、ドメイン不変条件、純粋な変換 |
| Integration | ASP.NET CoreホストとHTTP契約 | Routing、model binding、認証、ProblemDetails |
| Contract | 外部HTTPとの互換性 | status、header、JSON schema、timeout mapping |
| E2E | 主要な利用者フロー | 認証から重要操作までの横断経路 |

Unitで足りる分岐を、IntegrationやE2Eで重ねて書かない。逆に、HTTP status、serialization、DI、middleware、認証の境界は、Unitのmockだけで証明したことにしない。

## 検証方法

変更したあと、Skillの形式と、禁止したテストの骨組みが無いことを確かめた。

```powershell
$env:PYTHONUTF8 = '1'
python "$HOME\.codex\skills\.system\skill-creator\scripts\quick_validate.py" `
  "$HOME\.agents\skills\xunit-aspnet-core-test-case-design"
```

結果は`Skill is valid!`だった。さらに、属性の直後にテストメソッドが続くような、実行できるxUnitの骨組みを正規表現で探し、見つからないことを確かめた。

変更は次のコミットに残した。

```text
f85d929 fix: require comment-only xUnit plans
```

## 制約と再考条件

コメントはテストランナーに認識されない。なので、失敗もSkippedも報告されない。「未実装テストの一覧」として、別に管理する必要がある。

将来、利用者が実行できるテストコードをはっきり求めたら、コメント専用Skillの制約をその場で無視するのではなく、別のSkillに責務を分けるほうが安全である。発火条件と出力の契約を分けておけば、「設計だけ」と「実装まで」の境目をレビューしやすくなる。

確認日: 2026年8月11日

## まとめ

確認日: 2026年8月11日
