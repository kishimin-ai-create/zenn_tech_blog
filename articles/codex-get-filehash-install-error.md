---
title: "Codex CLIのインストールでGet-FileHashが見つからないときの切り分け"
emoji: "🔍"
type: "tech"
topics: ["codex", "powershell", "windows"]
published: true
---

## 結論

結論から言うと、2026年8月の調査では直接原因を再現できなかった。

2026年9月23日に、同じ症状が再発した。インストーラー実行中のPowerShellが`Get-FileHash`を解決できず、ラッパーコマンドが終了コード1になった。ここまでは確認できた。

ただ、PowerShellのバージョンやモジュール探索状態はまだ確認できていない。なので、根本原因と解決方法はまだ確定していない。

## 2026年9月23日の再発記録

ユーザーから、次のコマンドが終了コード1で失敗したと報告された。

```powershell
powershell -ExecutionPolicy Bypass -c '$env:CODEX_NON_INTERACTIVE=1; irm https://chatgpt.com/codex/install.ps1 | iex'
```

ログを見ると、インストーラーの217行目22文字目付近で止まっていた。アーカイブのハッシュ計算に使う`Get-FileHash`が解決されていなかった。

```text
+     $actualDigest = (Get-FileHash -LiteralPath $ArchivePath -Algorith ...
+                      ~~~~~~~~~~~~
+ CategoryInfo          : ObjectNotFound: (Get-FileHash:String) [], ParentContainsErrorRecordException
+ FullyQualifiedErrorId : CommandNotFoundException
```

この出力から、失敗したのはPowerShellのコマンド解決だと判断した。`$ArchivePath`の存在確認より前の段階である。ただし、実行環境の`$PSVersionTable`、`Get-Command Get-FileHash`、モジュール探索パスはまだ取れていない。

次に確認すべき情報はこの2つ。

```powershell
$PSVersionTable
Get-Command Get-FileHash -All
```

`pwsh`か`powershell`を明示して再実行する手もある。ただ、今回のログだけでは復旧したかどうかは分からない。再実行したら、インストーラーの出力全文、終了コード、`codex --version`を記録する。

## はじめに

WindowsでCodex CLIをインストールしたら、チェックサム検証で`Get-FileHash`が見つからずに終了した。この事例を整理する。対象は、PowerShellの基本操作ができて、公式インストールスクリプトを使っている開発者である。

2026年8月11日に調べたときは、Windows PowerShell 5.1とPowerShell 7.6.4のどちらでも`Get-FileHash`を使えた。元のエラーは再現できなかった。

2026年9月23日の再発では、コマンドが解決できないことと終了コード1は確認できた。ただ、実行環境の診断情報が足りない。なので、根本原因と解決方法はまだ確定していない。

## 発生した症状

Codex CLI 0.147.0のインストール中、次の順で処理が止まった。

1. `releases.openai.com`からチェックサムを取得または検証できなかった。
2. インストーラーがGitHub Releasesへフォールバックした。
3. `Get-FileHash`を解決できず、終了コード1になった。

主なエラーはこれ。

```text
The term 'Get-FileHash' is not recognized as the name of a cmdlet,
function, script file, or operable program.
```

公式リポジトリのWindowsインストーラーは、アーカイブのSHA-256を`Get-FileHash`で計算し、期待値と比べている。また、`releases.openai.com`を優先し、取得や検証に失敗したらGitHub Releasesへ切り替える作りになっている。

## 最初に確認すること

PowerShellで次を実行する。

```powershell
$PSVersionTable.PSVersion
Get-Command Get-FileHash
Get-Module -ListAvailable Microsoft.PowerShell.Utility
```

PowerShellのバージョンと、`Get-FileHash`のコマンド情報が表示されれば期待どおりである。`Get-FileHash`が見つからなければ、インストーラーを再実行する前に、PowerShell本体と`Microsoft.PowerShell.Utility`がどう解決されているかを確認する。

PowerShellが複数入っているなら、実体も確認する。

```powershell
where.exe powershell
where.exe pwsh
```

`powershell.exe`はWindows PowerShell、`pwsh.exe`はPowerShell 7以降を指す。ターミナルで確認した環境と、別のプロセスから起動された環境が同じとは限らない。

## PowerShell 7から再実行する

`pwsh`で`Get-FileHash`を解決できるなら、PowerShell 7のプロンプトから公式コマンドを実行する。

```powershell
$env:CODEX_NON_INTERACTIVE = '1'
irm https://chatgpt.com/codex/install.ps1 | iex
```

別のシェルからPowerShell 7へ渡すなら、この形が使える。

```powershell
$env:CODEX_NON_INTERACTIVE = '1'
irm https://chatgpt.com/codex/install.ps1 | pwsh -NoProfile -Command -
```

インターネットから取ってきたスクリプトをそのまま実行することになる。実行前に、URLと取得した中身がOpenAI公式のものかを確認する。監査が必要な環境なら、スクリプトを保存してレビューしてから実行する。

## インストール結果を確認する

再実行したら、別のPowerShellウィンドウを開いて確認する。

```powershell
Get-Command codex
codex --version
```

今回の調査では、再インストールはしていない。なので、この手順で元の環境が直るかは未検証である。また失敗したら、実行したPowerShellのバージョン、`Get-Command Get-FileHash`の結果、インストーラーが最初に失敗した箇所を保存して見比べる。

## 分かったことと分からなかったこと

確認できたことは次のとおり。

- 2026年8月11日時点の端末には、PowerShell 7.6.4とWindows PowerShell 5.1があった。
- 確認したときは、どちらでも`Get-FileHash`を解決できた。
- 公式インストーラーは、チェックサム検証に`Get-FileHash`を使っていた。
- GitHub Releasesへのフォールバックは、インストーラーが持っている通常の代替経路だった。

元の失敗のときだけコマンドを解決できなかった理由は、分かっていない。PowerShellの版だけを原因と決めつけない。呼び出された実体、プロファイルの有無、モジュール探索パス、親プロセスから渡された環境を、分けて確認する必要がある。

## 参考資料

- [OpenAI Codex公式リポジトリのインストール手順](https://github.com/openai/codex#installing-and-running-codex-cli)
- [OpenAI Codex公式Windowsインストーラー](https://github.com/openai/codex/blob/main/scripts/install/install.ps1)

確認日: 2026年8月11日、追記確認日: 2026年9月23日

## まとめ

確認日: 2026年8月11日、追記確認日: 2026年9月23日
