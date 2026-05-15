# Skill `allowed-tools` 検証セット

`allowed-tools` の挙動を最小構成で検証するためのリポジトリ。
**同じセッション内で2つのスキルを呼び分けて挙動を比較できる**。

## 構成

```
skill-permission-test/
└── .claude/
    ├── skills/
    │   ├── echo-allowed/
    │   │   ├── SKILL.md
    │   │   └── scripts/run-test          # 独自スクリプト
    │   └── echo-manual/
    │       ├── SKILL.md                  # echo-allowed と同じ構造
    │       └── scripts/run-test
    └── settings.local.json               # "Skill(echo-allowed)" のみ許可
```

両スキルとも、自前のシェルスクリプト `scripts/run-test` を実行する。
`allowed-tools` にはそのスクリプトパスが完全一致で登録されている。

`echo` などのユーザー設定で既に許可されているコマンドを避けるため、独自スクリプト方式を採用。

## 使い方

セッション内で以下を順番に試す:

### Pattern 1: テキスト言及 + allowlist 登録あり → Bash プロンプトあり

```
echo-allowedでテストして
```

**期待**: Skill 起動はプロンプトなし（allowlist で自動許可）。
続く `scripts/run-test` 実行で**許可プロンプトあり**（`allowed-tools` が無効化）。

### Pattern 2: テキスト言及 + allowlist 未登録 → Bash プロンプトなし

```
echo-manualでテストして
```

**期待**: Skill 起動の**許可プロンプトあり** → 手動で Yes。
続く `scripts/run-test` 実行は**プロンプトなし**（`allowed-tools` 適用）。

### Pattern 3: スラッシュコマンド → Bash プロンプトなし

```
/echo-allowed
```

```
/echo-manual
```

**期待**: どちらも Bash プロンプトなし。
※ `/echo-manual` は Skill 自体の起動プロンプトが出る可能性あり（出ても allowed-tools は適用される）。

## 期待される挙動表

| スキル       | 入力方法     | Skill プロンプト      | Bash プロンプト |
| ------------ | ------------ | --------------------- | --------------- |
| echo-allowed | テキスト言及 | ✅ なし（allowlist）  | ❌ あり         |
| echo-manual  | テキスト言及 | ❌ あり（手動承認）   | ✅ なし         |
| echo-allowed | スラッシュ   | ─（slash で明示起動） | ✅ なし         |
| echo-manual  | スラッシュ   | ─（slash で明示起動） | ✅ なし         |

この差異から「allowlist 自動許可だと `allowed-tools` が無効化される」ルールが再現される。

## テキスト言及で自動実行したい場合

`Skill(name)` を allowlist に入れて auto-invoke 運用したい場合、`allowed-tools` だけでは Bash プロンプトが出てしまう（上表 1 行目）。

完全自動化するには `settings.local.json` に **Bash 等の許可も併記**する必要がある:

```json
{
  "permissions": {
    "allow": [
      "Skill(echo-allowed)",
      "Bash(.claude/skills/echo-allowed/scripts/run-test)"
    ]
  }
}
```

`allowed-tools` はあくまでスラッシュコマンドや手動承認経由でのみ効くと割り切り、自動運用は `settings.local.json` に明示的に書くのが確実。
