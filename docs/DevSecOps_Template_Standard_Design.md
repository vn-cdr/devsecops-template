# DevSecOpsテンプレート標準設計書

## GitHub Enterprise Cloud／プライベートリポジトリ／5,000人・5,000リポジトリ想定

- 作成日：2026-11-07
- 対象：全社GitHub Enterprise Cloud（GHEC）環境
- 対象規模：約5,000ユーザー、約5,000リポジトリ
- リポジトリの前提：プライベートリポジトリ
- 基本方針：GitHub Template Repositoryを用いて、リポジトリ単位の開発パイプラインとセキュリティ設定ファイルを配布する
- 設計上の前提：Reusable Workflowは標準構成では利用しない。各リポジトリにワークフローをコピーし、中央統制はGHECの標準機能で実施する

------------------------------------------------------------------------

# 1. エグゼクティブサマリー

本設計では、DevSecOpsテンプレートを単なるGitHub ActionsのYAML集ではなく、全社共通のリポジトリ標準を実装・配布・維持するための仕組みとして定義する。

5,000人・5,000リポジトリ規模では、テンプレートにすべての設定を持たせる方式は適切ではない。テンプレートから作成されたリポジトリはテンプレート元と独立した履歴を持ち、テンプレートの更新が既存リポジトリへ自動反映されない。また、テンプレートリポジトリのファイル展開と、OrganizationのRulesetsやSecurity ConfigurationsなどのGitHub設定は別の仕組みである。

したがって、次の3層で設計する。

1.  **全社共通統制（GitHub標準機能）**：Organization Rulesets、Security Configurations、Actionsポリシー、権限設定など。全リポジトリに適用し、可能な限り強制する。
2.  **DevSecOpsテンプレート（GitHubリポジトリ）**：CI/CD、テスト、スキャン、SBOM、アーティファクト証明、README、CODEOWNERSなどをファイルとして配布する。
3.  **公開後の運用（管理プロセス・自動化）**：バージョン管理、利用状況の可視化、更新PR、準拠状況の監査、例外管理を行う。

## 1.1 推奨結論

- テンプレートリポジトリはプライベートで公開し、対象開発者・チームにRead権限を付与する。
- まず共通テンプレートを定義し、コンテナ、サーバレス、仮想サーバ、IaCなどは必要に応じてワークロード別テンプレートとして分ける。
- Branch Protection相当のルールはOrganization Rulesetsで管理し、テンプレートに依存させない。
- Secret Scanning、Push Protection、Code ScanningなどはSecurity Configurationsで一括管理する。
- GitHub Actionsの利用可能なAction、SHA固定、GITHUB_TOKENの既定権限はOrganization／Enterprise側で統制する。
- 新規リポジトリ作成時には、テンプレート展開だけでなく、チーム権限、Custom Properties、Security Configuration、必要なEnvironment等を設定する作成フローを用意する。
- テンプレート更新は自動同期ではなく、更新PRを各リポジトリに配布する方式とする。
- 5,000リポジトリへの一斉展開は避け、パイロット、段階展開、全社展開の順で進める。

------------------------------------------------------------------------

# 2. 前提・用語・設計原則

## 2.1 DevSecOpsテンプレートの定義

本設計におけるDevSecOpsテンプレートは、以下を満たすGitHubリポジトリである。

- GitHubリポジトリ単位の生産性向上を目的とした、ベストプラクティスなパイプラインテンプレート。
- GitHubリポジトリ単位のセキュリティ対策を実装するための、ベストプラクティスな設定ファイル群。
- 新規リポジトリ作成時に、開発チームが共通の初期構成を利用できる。
- テンプレートのバージョン、変更内容、利用方法、運用ルールが明確である。
- 特定のアプリケーションの実装や秘密情報を含まない。

## 2.2 想定環境

| 項目               | 前提                                                                            |
|:-------------------|:--------------------------------------------------------------------------------|
| GitHub             | GitHub Enterprise Cloud                                                         |
| リポジトリ         | Organization配下のプライベートリポジトリ                                        |
| ユーザー数         | 約5,000人                                                                       |
| リポジトリ数       | 約5,000                                                                         |
| CI/CD              | GitHub Actionsを基本とする                                                      |
| テンプレート配布   | GitHub Template Repository                                                      |
| 共通統制           | Organization／Enterprise設定                                                    |
| ワークフロー再利用 | Reusable Workflowは標準構成では使用しない                                       |
| リポジトリ構成     | アプリケーション、コンテナ、サーバレス、仮想サーバ、IaC等が混在する可能性を想定 |

## 2.3 設計原則

1.  **強制すべきものはGitHub標準機能で強制する。** テンプレート内の設定ファイルだけで統制しない。
2.  **テンプレートは生産性を高める。** 開発者が毎回同じパイプラインや設定を作る負担を減らす。
3.  **共通と個別を分ける。** 全リポジトリ共通の設定と、ワークロード固有の設定を混在させない。
4.  **最小権限を基本とする。** Actions、GitHub App、Runner、Environmentの権限を必要最小限にする。
5.  **導入後の更新を前提とする。** テンプレートの初期展開だけでなく、既存リポジトリへの更新配布を設計する。
6.  **段階的に強制する。** 既存リポジトリの状況を把握し、Evaluate等で影響を確認してからActiveにする。
7.  **例外を管理する。** 例外は期限、責任者、リスク、代替統制を記録する。

------------------------------------------------------------------------

# 3. 全体アーキテクチャ

``` mermaid
flowchart TB
    subgraph GOV["全社共通統制：GitHub Organization / Enterprise"]
      RS["Organization Rulesets"]
      SC["Security Configurations"]
      AP["Actions Policies"]
      RP["Repository Permissions / Custom Properties"]
    end

    subgraph TPL["DevSecOpsテンプレート"]
      BASE["共通テンプレート"]
      APP["アプリケーション向け"]
      CON["コンテナ向け"]
      SRV["サーバレス向け"]
      VM["仮想サーバ向け"]
      IAC["IaC向け"]
    end

    subgraph CREATE["リポジトリ作成フロー"]
      PORTAL["申請・作成ポータル / CLI / GitHub App"]
      META["所有者・分類・テンプレート版の登録"]
      ACCESS["チーム権限・Environment等の設定"]
    end

    subgraph REPOS["各開発リポジトリ：約5,000"]
      FILES["Workflow / CODEOWNERS / Dependabot / Docs"]
      PIPE["CI/CD・テスト・セキュリティ検査"]
      ART["SBOM / Artifact Attestation / Release"]
    end

    subgraph OPS["公開後の運用"]
      RELEASE["テンプレートのリリース・変更管理"]
      UPDATE["更新PRの自動配布"]
      AUDIT["準拠状況・利用状況の監査"]
      EX["例外申請・期限管理"]
    end

    TPL --> PORTAL
    PORTAL --> META
    META --> ACCESS
    ACCESS --> REPOS
    GOV --> REPOS
    FILES --> PIPE --> ART
    RELEASE --> UPDATE --> REPOS
    REPOS --> AUDIT
    AUDIT --> EX
```

## 3.1 各構成要素の役割

| 構成要素                | 役割                                             | 主な実装                                  |
|:------------------------|:-------------------------------------------------|:------------------------------------------|
| DevSecOps Template      | 新規リポジトリの初期ファイル・パイプラインを配布 | Template Repository                       |
| Organization Rulesets   | ブランチ・タグ・マージ条件を統制                 | Organization Rulesets                     |
| Security Configurations | セキュリティ機能の有効化と適用範囲を管理         | Security Configurations                   |
| Actions Governance      | 実行可能なAction、権限、トークンを制御           | Organization／Enterprise Actions Settings |
| Repository Metadata     | 所有者、分類、テンプレート版、リスク区分を管理   | Custom Properties                         |
| Repository Creation     | リポジトリ作成、チーム付与、メタデータ設定       | GitHub App、API、CLI、社内ポータル        |
| Update Management       | テンプレート更新を既存リポジトリへ配布           | GitHub App等による更新PR                  |
| Compliance Management   | 導入率、設定準拠率、脆弱性対応状況を確認         | Security Overview、API、監査レポート      |

------------------------------------------------------------------------

# 4. テンプレートとGitHub標準設定の責任分界

## 4.1 基本的な責任分界

| 対象                       |  テンプレートで配布   |   GitHub標準機能で設定   | 運用で管理 |
|:---------------------------|:---------------------:|:------------------------:|:----------:|
| GitHub Actionsワークフロー |           ○           |                          |     ○      |
| テスト・Lint・Build処理    |           ○           |                          |     ○      |
| CODEOWNERS                 |           ○           |                          |     ○      |
| Pull Requestテンプレート   |           ○           |                          |            |
| Dependabot設定ファイル     |           ○           |     ○（機能有効化）      |     ○      |
| CodeQL                     |   △（高度な構成時）   | ○（Default setupを優先） |     ○      |
| Secret Scanning            |                       |            ○             |     ○      |
| Push Protection            |                       |            ○             |     ○      |
| Dependency Graph           |                       |            ○             |     ○      |
| Dependency Review          |      ○（Action）      | ○（Required Workflow等） |     ○      |
| Branch Protection          |                       |      ○（Rulesets）       |     ○      |
| PRレビュー必須             |                       |      ○（Rulesets）       |     ○      |
| Actions利用制限            |                       |            ○             |     ○      |
| GITHUB_TOKEN既定権限       |                       |            ○             |     ○      |
| Repository権限             |                       | ○（Organization設定等）  |     ○      |
| Environment保護            |                       |      ○（API／設定）      |     ○      |
| SBOM生成                   |           ○           |                          |     ○      |
| Artifact Attestations      |           ○           |   ○（利用条件・権限）    |     ○      |
| コンテナスキャン           | ○（対象テンプレート） |                          |     ○      |
| IaCスキャン                | ○（対象テンプレート） |                          |     ○      |
| DAST                       | △（対象システムのみ） |                          |     ○      |
| 例外管理                   |                       |                          |     ○      |

凡例： - ○：主な実装・管理先 - △：対象リポジトリや技術スタックに応じて採用

## 4.2 Template Repositoryの制約

GitHub Template Repositoryは、テンプレートのディレクトリ構成やファイルを新しいリポジトリに展開する機能である。テンプレートから作成されたリポジトリは独立した履歴を持つ。

したがって、以下はテンプレートファイルのコピーだけでは実現できないものとして扱う。

- Organization Rulesetsの適用
- リポジトリのチーム権限・Collaborator設定
- Repository Secrets、Environment Secrets
- Environmentの保護ルール
- ActionsのRepository／Organizationポリシー
- Security Configurationの適用
- WebhookやGitHub Appのインストール・設定
- Custom Propertiesの値の登録
- リポジトリの説明、Topics等のメタデータ設定

これらはOrganization／Enterprise側の標準機能、またはリポジトリ作成フローで設定する。

------------------------------------------------------------------------

# 5. 全社共通のGitHub設定

## 5.1 Repository作成・権限設定

| 設定項目                     | 推奨設定                                     | 理由・補足                          |
|:-----------------------------|:---------------------------------------------|:------------------------------------|
| Repository Visibility        | Private                                      | 本設計の前提                        |
| Repository Creation          | 権限を持つ担当者・自動化に限定               | 野良リポジトリの発生を防止          |
| Organization Base Permission | NoneまたはReadを基本に検討                   | 全メンバーへの過剰なWrite権限を防止 |
| Team Access                  | GitHub Team単位で付与                        | 個人単位の権限付与を減らす          |
| Outside Collaborator         | 原則禁止または申請制                         | 外部アクセスの統制                  |
| Repository Admin             | 最小限の担当者に限定                         | 設定変更・削除リスクの低減          |
| Repository Deletion          | 管理者限定、削除時の承認手順                 | 誤削除防止                          |
| Private Repository Fork      | Enterprise／Organizationポリシーに沿って制限 | コードの複製・持ち出しリスクを低減  |
| Default Branch               | `main`を標準                                 | ルール・ワークフローの共通化        |

## 5.2 Custom Properties

5,000リポジトリを管理するため、Custom Propertiesを導入する。テンプレート利用状況、対象ワークロード、重要度、所有者を機械的に判定できるようにする。

推奨プロパティ：

| Property                     | 値の例                                             | 用途                   |
|:-----------------------------|:---------------------------------------------------|:-----------------------|
| `devsecops_profile`          | `base` / `container` / `serverless` / `vm` / `iac` | 適用テンプレートの識別 |
| `devsecops_template_version` | `1.2.0`                                            | 導入済みテンプレート版 |
| `system_owner`               | チーム名または管理ID                               | 責任者の識別           |
| `business_criticality`       | `low` / `medium` / `high` / `critical`             | リスク別統制           |
| `data_classification`        | 組織の分類体系                                     | データ保護レベル       |
| `environment_type`           | `development` / `staging` / `production`           | 環境区分               |
| `repository_lifecycle`       | `active` / `maintenance` / `archived`              | ライフサイクル管理     |
| `exception_status`           | `none` / `approved` / `expired`                    | 例外管理               |

推奨設定：

- `devsecops_profile`、`system_owner`、`business_criticality`、`data_classification`は必須化を検討する。
- `devsecops_template_version`は作成フローで自動設定する。
- 開発者が自由に変更できるプロパティと、管理者・自動化だけが変更できるプロパティを分ける。
- Custom PropertiesをRulesetsの対象リポジトリ選択に活用する。
- 既存リポジトリにも一括で値を設定し、未分類リポジトリを可視化する。

## 5.3 Organization Rulesets

Branch Protectionはリポジトリ単位の個別設定にせず、Organization Rulesetsを基本とする。

### 推奨ルール

**対象ブランチ** - `main` - `release/*`（リリースブランチを運用する場合） - 必要に応じて保護対象の長期ブランチ

**マージ条件** - Pull Request経由の変更を必須化する。 - 通常リポジトリは1名以上のApproveを必須とする。 - 高重要度リポジトリは2名以上、またはセキュリティ・運用担当者のレビューを追加する。 - CODEOWNERSによるレビューを必須化する対象を定義する。 - 新しいコミットがPushされた場合、古いApproveを無効化する設定を検討する。 - 最後のPushを行ったユーザー以外のApproveを要求する。 - Conversation Resolutionを必須化する。 - 必須Status Checkが成功しない場合はマージ不可とする。

**ブランチ保護** - Force Pushを禁止する。 - Branch Deletionを禁止する。 - 管理者によるBypassを原則禁止する。 - 緊急時のBypassは限定されたチームに付与し、理由・実施者を監査する。 - RulesetのEnforcement Statusは、既存環境への影響確認後にActiveにする。

**タグ保護** - `v*`等のリリースタグを保護する。 - リリースタグの削除・更新を制限する。 - リリース権限を限定する。

### Rulesetsの設計単位

- 全リポジトリ共通の基本Rulesetを作る。
- `business_criticality`や`devsecops_profile`に応じた追加Rulesetを作る。
- リポジトリごとに大量のRulesetを作成しない。
- 例外はBypass権限の乱用ではなく、申請・期限・代替統制で管理する。

## 5.4 GitHub ActionsのOrganization設定

| 設定項目                        | 推奨                                              |
|:--------------------------------|:--------------------------------------------------|
| Actionsの利用                   | 原則有効。ただし承認済みActionに制限              |
| Allow Actions                   | GitHub公式Actionと社内承認済みActionをAllowlist化 |
| Third-party Actions             | 原則Allowlist方式                                 |
| Action Version                  | Full-length Commit SHAへの固定を必須化            |
| Branch／Tag参照                 | `@main`等の可変参照を禁止                         |
| GITHUB_TOKEN Default Permission | Read-onlyを基本                                   |
| Job単位の権限                   | `permissions`で必要な権限だけを明示               |
| PR作成・Approve権限             | 原則無効                                          |
| Fork Pull Request               | 外部コードにSecretsを渡さない                     |
| Organization Secrets            | 共通用途に限定し、Repository単位のSecretsを最小化 |
| Self-hosted Runner              | Runner Group、対象Repository、ネットワークを分離  |
| Runnerの永続化                  | 可能な限りEphemeral Runnerを採用                  |
| Workflowの変更                  | CODEOWNERSでレビュー対象にする                    |

補足： - `GITHUB_TOKEN`の既定値をRead-onlyにしても、Workflow内の`permissions`で権限を追加できる場合がある。より上位のポリシーで制限できる範囲を確認し、Workflowのレビューも組み合わせる。 - Full-length SHA固定をOrganizationポリシーで要求する場合、GitHub公式Actionも含めてSHA固定が必要になる。 - ActionのAllowlistを社内Actionだけにすると、`actions/checkout`等のGitHub公式Actionも実行できなくなるため、必要なActionを明示的に許可する。 - Workflow dependency lockfileはActionの推移的依存関係も固定できるが、利用可否・Preview状態を確認して導入する。標準化の基本はFull-length SHA固定とする。

## 5.5 Security Configurations

Security Configurationsは、全リポジトリのセキュリティ機能を一括で有効化・管理するために利用する。

### 推奨するセキュリティ設定

| 機能                            | 推奨             | 実装・運用             |
|:--------------------------------|:-----------------|:-----------------------|
| Dependency Graph                | 有効             | 依存関係の可視化       |
| Dependabot Alerts               | 有効             | 脆弱な依存関係の検知   |
| Dependabot Security Updates     | 有効             | 脆弱性修正PRの作成     |
| Dependabot Version Updates      | 必要に応じて有効 | 定期的な依存関係更新   |
| Secret Scanning                 | 有効             | 漏えいしたSecretの検知 |
| Push Protection                 | 有効             | SecretのPushをブロック |
| Code Scanning / CodeQL          | 対応言語で有効   | SAST                   |
| Dependency Review               | 有効             | PRで依存関係変更を検査 |
| Private Vulnerability Reporting | 必要に応じて有効 | 脆弱性の報告経路       |

### CodeQLの方式

- 基本はCodeQL Default Setupを推奨する。
- Default Setupで対応できないビルド方式・言語・カスタムクエリがある場合のみAdvanced Setupを検討する。
- Default SetupはOrganization単位で対象リポジトリへ適用し、可能な限りWorkflowファイルの個別管理を減らす。
- CodeQLの対象言語、Query Suite、失敗条件、アラート対応基準を定義する。
- CodeQL Default SetupとAdvanced Setupを同じリポジトリで不用意に併用しない。

### Dependabotの方式

- Dependency Graphを有効化する。
- Dependabot Alertsを有効化する。
- Dependabot Security Updatesを有効化する。
- `dependabot.yml`で対象エコシステム、更新頻度、グルーピング、PR数を制御する。
- 依存関係更新PRのレビュー担当をCODEOWNERS等で定義する。
- 大量のPRが発生しないよう、更新グループとスケジュールを調整する。

### ライセンス上の注意

プライベートリポジトリでの利用条件は、契約中のGitHubプラン・追加ライセンスに依存する。少なくとも次を契約・管理画面で確認する。

| 機能                   | プライベートリポジトリでの主な条件 |
|:-----------------------|:-----------------------------------|
| CodeQL / Code Scanning | GitHub Code Securityが必要         |
| Dependency Review      | GitHub Code Securityが必要         |
| Secret Scanning        | GitHub Secret Protectionが必要     |
| Push Protection        | GitHub Secret Protectionが必要     |
| Artifact Attestations  | GHECで利用可能                     |

ライセンス対象外の機能を必須パイプラインに組み込まない。ライセンスが未整備の場合は、代替ツールと費用・運用負荷を比較して決定する。

------------------------------------------------------------------------

# 6. DevSecOpsテンプレートに含める内容

## 6.1 共通テンプレートのファイル構成

``` text
devsecops-template-base/
├── README.md
├── CONTRIBUTING.md
├── SECURITY.md
├── CHANGELOG.md
├── TEMPLATE_VERSION
├── .gitignore
├── .editorconfig
├── .github/
│   ├── CODEOWNERS
│   ├── pull_request_template.md
│   ├── ISSUE_TEMPLATE/
│   │   ├── bug_report.yml
│   │   └── feature_request.yml
│   ├── dependabot.yml
│   └── workflows/
│       ├── ci.yml
│       ├── dependency-review.yml
│       ├── build.yml
│       └── release.yml
├── docs/
│   ├── development.md
│   ├── pipeline.md
│   ├── security.md
│   └── operations.md
├── scripts/
│   └── validate.sh
└── src/
    └── .gitkeep
```

実際のアプリケーションコード、Dockerfile、Terraform等はワークロード別テンプレートに配置する。

## 6.2 共通で含めるファイル

| ファイル                                  | 内容・目的                               | 優先度                       |
|:------------------------------------------|:-----------------------------------------|:-----------------------------|
| `README.md`                               | リポジトリ概要、利用方法、構成、実行方法 | 必須                         |
| `CONTRIBUTING.md`                         | 開発・PR・レビューのルール               | 推奨                         |
| `SECURITY.md`                             | 脆弱性報告・セキュリティ連絡先           | 必須                         |
| `CHANGELOG.md`                            | 変更履歴                                 | 必須                         |
| `TEMPLATE_VERSION`                        | 適用したテンプレート版                   | 必須                         |
| `.gitignore`                              | 不要ファイルの除外                       | 必須                         |
| `.editorconfig`                           | エディタ設定の統一                       | 推奨                         |
| `.github/CODEOWNERS`                      | コード・設定ファイルのレビュー責任者     | 必須                         |
| `.github/pull_request_template.md`        | PR記載事項の標準化                       | 推奨                         |
| `.github/ISSUE_TEMPLATE/`                 | Issueの入力標準化                        | 推奨                         |
| `.github/dependabot.yml`                  | 依存関係更新の設定                       | 対象に応じて必須             |
| `.github/workflows/ci.yml`                | CIの基本処理                             | 必須                         |
| `.github/workflows/dependency-review.yml` | PRの依存関係検査                         | ライセンス・方式に応じて必須 |
| `docs/`                                   | 開発・パイプライン・運用手順             | 必須                         |

## 6.3 CODEOWNERS

CODEOWNERSは、レビュー責任を明確にするためにテンプレートに含める。

例：

``` text
# リポジトリ全体の責任者
* @organization/application-team

# GitHub Actionsやセキュリティ設定
/.github/ @organization/devsecops-team

# TerraformなどのIaC
/terraform/ @organization/platform-team
```

注意点： - 実際のTeam名に置き換える。 - テンプレートのプレースホルダーを残したまま本番利用させない。 - CODEOWNERS自身の変更にもOwnerを定義する。 - RulesetsでCODEOWNERSレビューを必須化する場合、Ownerが存在しない状態でマージできないようにする。

## 6.4 CI/CDパイプラインの基本構成

### Pull Request時

| 処理              | 目的                             | 推奨                 |
|:------------------|:---------------------------------|:---------------------|
| Checkout          | ソース取得                       | 必須                 |
| Format Check      | コード整形確認                   | 対象言語で必須       |
| Lint              | 静的品質確認                     | 必須                 |
| Unit Test         | 単体テスト                       | 必須                 |
| Build             | ビルド確認                       | 必須                 |
| Dependency Review | 依存関係の脆弱性・ライセンス確認 | 対象リポジトリで必須 |
| CodeQL            | SAST                             | 対応言語で必須       |
| IaC Scan          | IaCの設定不備検査                | IaC利用時に必須      |
| Container Scan    | コンテナイメージ検査             | コンテナ利用時に必須 |
| Secret Scan       | Secret漏えい検査                 | GitHub標準機能を優先 |

### Mainブランチへのマージ後

| 処理                    | 目的                     | 推奨               |
|:------------------------|:-------------------------|:-------------------|
| Build                   | リリース候補の生成       | 必須               |
| Unit / Integration Test | 品質確認                 | 必須               |
| Container Build         | コンテナ作成             | コンテナ利用時     |
| Container Scan          | イメージ脆弱性検査       | コンテナ利用時     |
| SBOM Generation         | ソフトウェア構成の可視化 | リリース対象で必須 |
| Artifact Attestation    | ビルド由来・完全性の証明 | リリース対象で必須 |
| Artifact Publish        | 成果物の保管             | 必要に応じて必須   |

### デプロイ時

| 処理                     | 目的                         | 推奨                    |
|:-------------------------|:-----------------------------|:------------------------|
| Environment Protection   | 本番デプロイの承認・制限     | 本番環境で必須          |
| OIDC Authentication      | クラウド認証                 | 対応クラウドで必須      |
| Deploy                   | デプロイ                     | 必須                    |
| DAST                     | 稼働中アプリケーションの検査 | Web/API等で必要に応じて |
| Post-deploy Test         | デプロイ後の動作確認         | 必須                    |
| Attestation Verification | 成果物の由来・完全性確認     | 本番リリースで推奨      |

## 6.5 ワークロード別テンプレート

共通テンプレートにすべての処理を詰め込まず、対象技術に応じて分ける。

| テンプレート | 主な対象                 | 含める機能                                        |
|:-------------|:-------------------------|:--------------------------------------------------|
| Base         | 一般的なアプリケーション | CI、Lint、Test、Build、CODEOWNERS                 |
| Application  | Java、Node.js、Python等  | 言語別Build、Test、依存関係管理                   |
| Container    | Docker／OCIコンテナ      | Docker Build、イメージスキャン、SBOM、Attestation |
| Serverless   | AWS Lambda等             | パッケージ作成、IaC検査、デプロイ                 |
| VM           | 仮想サーバ向けアプリ     | Build、パッケージ、デプロイ、構成検査             |
| IaC          | Terraform等              | Format、Validate、Lint、IaC Scan、Planレビュー    |

### テンプレート設計上の注意

- 共通テンプレートには、ほぼすべてのリポジトリに必要な最低限の処理だけを含める。
- コンテナ、サーバレス、仮想サーバ、IaCの処理は対象テンプレートに分ける。
- DASTは稼働環境やテストデータが必要になるため、すべてのリポジトリに一律で組み込まない。
- コンテナスキャンはイメージのビルド後、リリース・デプロイ前を基本とする。
- CodeQL Default Setupを使う場合、同じ目的のCodeQL Workflowをテンプレートに重複して含めない。
- テンプレートに固定の本番アカウントID、Secret、環境固有値を含めない。
- 使わないWorkflowや空の設定ファイルを大量に配布しない。

------------------------------------------------------------------------

# 7. GitHub Actionsのセキュリティ設計

## 7.1 Actionの利用統制

- GitHub公式Actionを含む利用可能ActionをAllowlistで管理する。
- 第三者Actionは利用目的、メンテナンス状況、権限、依存関係を確認して承認する。
- Actionの参照はFull-length Commit SHAに固定する。
- `@main`、`@master`等の可変ブランチ参照を禁止する。
- Actionの更新は定期的にレビューし、セキュリティ修正を取り込む。
- 社内独自Actionを作る場合、所有者、変更管理、脆弱性対応、廃止手順を定義する。

## 7.2 GITHUB_TOKEN

- Organizationの既定権限はRead-onlyを基本とする。
- WorkflowまたはJob単位で`permissions`を明示する。
- `contents: write`、`packages: write`等は必要なJobだけに付与する。
- `pull-requests: write`等の権限は必要な場合だけに限定する。
- Workflow全体に過剰なWrite権限を付与しない。
- ActionsによるPR作成・Approveは原則禁止とする。

## 7.3 Secret管理

- Secretをソースコード、Workflow、ログ、Dockerfile等に直接記載しない。
- GitHub Secretsまたは外部Secret Managerを利用する。
- 長期固定Credentialを避け、可能な限りOIDCを利用する。
- Organization Secretsは共通用途に限定し、Repository単位のSecretを最小化する。
- Environment Secretsはデプロイ先の環境ごとに分離する。
- Secretの棚卸し・ローテーション・漏えい時の失効手順を整備する。

## 7.4 Runner

- GitHub-hosted Runnerを基本とする。
- Self-hosted Runnerが必要な場合、Runner Groupでアクセス可能なリポジトリを制限する。
- 本番デプロイ用Runnerと一般CI用Runnerを分離する。
- 可能な限りEphemeral Runnerを利用する。
- 外部コードを実行するJobと、重要なSecretを扱うJobを分離する。
- Runnerに残るWorkspace、Credential、Cache等の残存リスクを管理する。

------------------------------------------------------------------------

# 8. ソフトウェアサプライチェーン対策

## 8.1 Dependency Management

- Dependency Graphを有効化する。
- Dependabot Alertsを有効化する。
- Dependabot Security Updatesを有効化する。
- Dependabot Version Updatesは更新頻度とPR量を調整して利用する。
- Dependency ReviewをPR時に実施する。
- 依存関係の許可ライセンスを定義する。
- 未使用・EOL・既知脆弱性のある依存関係を定期的に棚卸しする。

## 8.2 コンテナ

コンテナを利用するリポジトリでは、以下を標準化する。

- 信頼できるベースイメージを使用する。
- ベースイメージを定期更新する。
- 不要なパッケージ・ツールを含めない。
- root実行を避ける。
- イメージの脆弱性スキャンを実施する。
- Critical／High等の脆弱性に対するFail条件を定義する。
- SBOMを生成する。
- リリース対象イメージに署名またはAttestationを付与する。
- デプロイ時に許可されたRegistry・イメージであることを検証する。

## 8.3 IaC

Terraform等を利用するリポジトリでは、以下を標準化する。

- Format／Validateを実施する。
- Lint・静的解析を実施する。
- Hard-coded Secretを禁止する。
- Public exposure、過剰権限、暗号化未設定等を検査する。
- Plan結果をPRでレビューする。
- 本番環境へのApplyはEnvironment保護と承認を組み合わせる。
- Stateファイルや機密情報をリポジトリに含めない。

## 8.4 SBOM・Artifact Attestations

- リリース対象の成果物にSBOMを付与する。
- 成果物のビルド元リポジトリ、Commit SHA、Workflow、実行環境を追跡できるようにする。
- GitHub Artifact Attestationsを利用し、成果物の由来を証明する。
- 生成だけで終わらず、成果物を利用・デプロイする側でAttestationを検証する。
- すべてのテストビルドに無条件でAttestationを付けるのではなく、配布・実行される成果物を中心に適用する。

注意： - プライベートリポジトリのArtifact AttestationsはGHECで利用できる。 - プライベートリポジトリではGitHubのSigstoreインスタンスが使われるが、公開透明性ログは利用されない。 - Attestationは成果物の由来を示すものであり、成果物自体が安全であることを保証するものではない。

------------------------------------------------------------------------

# 9. テンプレートの公開方法

## 9.1 公開リポジトリの構成

推奨例：

``` text
organization/
├── devsecops-template-base
├── devsecops-template-application
├── devsecops-template-container
├── devsecops-template-serverless
├── devsecops-template-vm
├── devsecops-template-iac
└── .github
    └── workflow-templates/
```

- 各テンプレートリポジトリはPrivateにする。
- 対象開発者・利用チームにはRead権限を付与する。
- テンプレートの変更権限はDevSecOps／Platform担当に限定する。
- 各テンプレートリポジトリの`main`を保護する。
- Template Repository設定を有効にする。
- リリースタグとCHANGELOGを管理する。

## 9.2 GitHub Template Repositoryの利用

新規リポジトリ作成時に、開発者が「Use this template」を利用して作成する。

標準の作成手順：

1.  利用するテンプレートを選択する。
2.  リポジトリ名、Descriptionを入力する。
3.  Organization配下にPrivateで作成する。
4.  作成フローでCustom Propertiesを登録する。
5.  必要なGitHub Team権限を付与する。
6.  Security Configurationが適用されていることを確認する。
7.  必要なEnvironment、Secrets、Variablesを設定する。
8.  `TEMPLATE_VERSION`とCustom Propertyのバージョンを確認する。
9.  初回CIを実行し、正常終了を確認する。

## 9.3 作成フローの自動化

5,000リポジトリ規模では、手作業だけで作成後設定を行わない。

推奨方式：

- 社内ポータル、GitHub App、API、CLI等を利用する。
- テンプレートからリポジトリを生成する。
- Private Visibilityを確認する。
- Custom Propertiesを設定する。
- 所有チームを付与する。
- 必要なEnvironmentを作成する。
- テンプレート版を登録する。
- Security ConfigurationとRulesetsの適用状況を確認する。
- 作成結果を台帳・監査データに記録する。

GitHub Template Repositoryはファイルを展開する仕組みであり、権限・Rulesets・Secrets・Environment等をすべて自動コピーする仕組みではない。作成フロー側で補完する。

## 9.4 `.github` Workflow Templatesとの違い

Organizationの`.github`リポジトリに配置する`workflow-templates`は、GitHub ActionsのWorkflowファイルを既存リポジトリに追加するための別機能である。

| 項目           | Template Repository              | Organization Workflow Templates |
|:---------------|:---------------------------------|:--------------------------------|
| 主な目的       | リポジトリ全体の初期構成         | Workflowファイルの追加          |
| 配布対象       | ファイル・ディレクトリ・ブランチ | Workflowテンプレート            |
| 主な利用場面   | 新規リポジトリ作成               | 既存リポジトリへのWorkflow追加  |
| 更新の自動反映 | されない                         | されない                        |
| 推奨用途       | 本設計の主方式                   | 追加Workflowを配布する補助方式  |

本設計ではTemplate Repositoryを主方式とする。`.github`のWorkflow Templatesは、既存リポジトリに後から特定のWorkflowを追加する用途に限定する。

## 9.5 Required Workflowsの活用

Organization Rulesetsの「Require workflows to pass before merging」は、共通の必須Workflowをマージ条件として適用する選択肢である。

- 例えばDependency Review等、全リポジトリで必須にしたいチェックを中央管理する用途で検討する。
- Reusable Workflowとは異なる仕組みである。
- 標準パイプライン全体を中央化する目的ではなく、全社で必須とする統制チェックに限定する。
- Workflowの配置先リポジトリ、アクセス権、対象ブランチ、対応イベントを事前に検証する。
- 対象リポジトリで実際にRequired Workflowが実行され、Rulesetでマージを制御できることをパイロットで確認する。

------------------------------------------------------------------------

# 10. テンプレートのバージョン管理と更新運用

## 10.1 バージョン管理

SemVerを基本とする。

| バージョン | 意味                         | 例      |
|:-----------|:-----------------------------|:--------|
| Major      | 互換性のない変更・大規模変更 | `2.0.0` |
| Minor      | 後方互換性のある機能追加     | `1.3.0` |
| Patch      | バグ修正・セキュリティ修正   | `1.2.1` |

管理する情報：

- Git Tag
- GitHub Release
- CHANGELOG
- `TEMPLATE_VERSION`
- 各リポジトリのCustom Property `devsecops_template_version`
- 対象ワークロード・対応言語
- 必要なライセンス・Runner・外部サービス
- 変更の影響範囲と移行手順

## 10.2 更新が自動反映されないことへの対応

テンプレートから作成されたリポジトリはテンプレート元と独立しているため、テンプレート側の更新は既存リポジトリへ自動反映されない。

したがって、次の更新方式を推奨する。

1.  DevSecOpsチームがテンプレートを更新する。
2.  テスト用リポジトリでCI・セキュリティ検査・デプロイを確認する。
3.  GitHub Releaseを作成する。
4.  更新対象リポジトリをCustom Propertiesで抽出する。
5.  更新用GitHub App等で各リポジトリにPull Requestを作成する。
6.  開発チームが差分・影響を確認してマージする。
7.  マージ後にテンプレート版を更新する。
8.  未更新リポジトリをレポートする。

更新PRは、原則として開発チームがレビュー・マージする。中央管理者が各リポジトリのファイルを直接上書きする方式は避ける。

## 10.3 更新PRの配布対象

- `devsecops_profile`で対象テンプレートを特定する。
- `devsecops_template_version`で更新対象のバージョンを特定する。
- 既に最新版のリポジトリは対象外にする。
- 例外承認中のリポジトリは、例外条件に従って扱う。
- Archived／Maintenanceのリポジトリは別ルールで管理する。

## 10.4 更新頻度

| 更新種別             | 推奨運用                        |
|:---------------------|:--------------------------------|
| 緊急セキュリティ修正 | 速やかに修正・影響調査・更新PR  |
| 重要な脆弱性対策     | リスクに応じて期限を設定        |
| 通常の改善           | 月次または四半期                |
| 大規模変更           | Major Releaseとして計画的に展開 |
| GitHub仕様変更対応   | 影響確認後、必要に応じて更新    |

## 10.5 5,000リポジトリへの更新展開

一度に5,000リポジトリへ更新PRを作成するのではなく、段階的に実施する。

推奨例：

| フェーズ |       対象        | 確認内容                             |
|:---------|:-----------------:|:-------------------------------------|
| Canary   | 10〜20リポジトリ  | 主要言語・主要ワークロードで動作確認 |
| Pilot    | 50〜100リポジトリ | チームのレビュー負荷・CI成功率       |
| Wave 1   |  全体の約10〜20%  | API制限、更新PR量、障害確認          |
| Wave 2   |  全体の約30〜50%  | 互換性・例外・未更新状況             |
| General  |    残りの対象     | 全社展開                             |

- API Rate Limitを考慮して処理を分割する。
- 失敗時のRetry、再実行、対象除外を用意する。
- 変更内容に応じて、緊急更新と通常更新の優先順位を分ける。
- 更新PRの作成数、マージ率、失敗率を記録する。

------------------------------------------------------------------------

# 11. 公開後の運用・ガバナンス

## 11.1 運用体制

| 役割                      | 主な責任                                          |
|:--------------------------|:--------------------------------------------------|
| DevSecOps／Platformチーム | テンプレート開発、公開、バージョン管理、更新PR    |
| GitHub管理者              | Organization設定、Rulesets、Actionsポリシー、権限 |
| Securityチーム            | セキュリティ標準、リスク評価、例外承認            |
| アプリケーションチーム    | テンプレート利用、Workflow保守、脆弱性対応        |
| システムオーナー          | リポジトリの所有、重要度・分類、例外の説明責任    |
| 監査・ガバナンス担当      | 準拠状況、例外、証跡の確認                        |

## 11.2 日常・定期運用

| 頻度   | 運用内容                                                   |
|:-------|:-----------------------------------------------------------|
| 日次   | Critical／High脆弱性、Secret漏えい、CI失敗の確認           |
| 週次   | Dependabot PR、Code Scanningアラート、更新滞留の確認       |
| 月次   | テンプレート更新、Actions・Runner・依存Actionの棚卸し      |
| 四半期 | テンプレート利用率、Rulesets準拠率、例外期限、権限の棚卸し |
| 半期   | セキュリティ標準・パイプライン標準の見直し                 |
| 年次   | 全社標準、リスク評価、ライセンス・運用コストの見直し       |

## 11.3 利用状況の可視化

次の項目を継続的に収集する。

- 全リポジトリ数
- テンプレート利用リポジトリ数
- テンプレート利用率
- テンプレートのバージョン別リポジトリ数
- Rulesets適用率
- Security Configuration適用率
- Secret Scanning有効率
- Push Protection有効率
- CodeQL有効率
- Dependabot Alerts有効率
- Dependency Review実行率
- 必須Status Checkの準拠率
- CODEOWNERS設定率
- 未対応脆弱性数・重大度別件数
- 更新PRの作成数・マージ率・滞留日数
- 例外申請数・期限切れ数

可視化にはGitHub Security Overview、Custom Properties、RulesetsのInsights、REST／GraphQL API、必要に応じて社内BIを組み合わせる。

## 11.4 例外管理

例外を認める場合は、最低限以下を記録する。

- 対象リポジトリ
- 対象ルール・機能
- 例外理由
- リスク評価
- 代替統制
- 申請者
- 承認者
- 有効期限
- 再評価日
- 解消計画

例外は恒久化しない。期限切れを自動検出し、再申請または解消を要求する。

## 11.5 脆弱性対応基準

組織のリスク基準に合わせてSLAを決定する。以下は初期案であり、事業影響や公開範囲を考慮して調整する。

| 重大度   |  初期対応目安  | 修正・緩和目安 |
|:---------|:--------------:|:--------------:|
| Critical |  1営業日以内   |    7日以内     |
| High     |  3営業日以内   |    30日以内    |
| Medium   |  10営業日以内  |    90日以内    |
| Low      | 定期トリアージ |    計画対応    |

- CVSSだけでなく、悪用状況、公開範囲、到達可能性、事業影響を考慮する。
- 期限内に修正できない場合は例外管理に移す。
- Secret漏えいは重大度にかかわらず、Credential失効・ローテーションを優先する。

------------------------------------------------------------------------

# 12. 準拠状況のKPI

初期目標値の例：

| KPI                                         |                初期目標                |
|:--------------------------------------------|:--------------------------------------:|
| 新規リポジトリのテンプレート利用率          |                95%以上                 |
| 新規リポジトリの必須Custom Properties設定率 |                  100%                  |
| Organization Rulesetsの適用率               |                  100%                  |
| Security Configurationの適用率              | 100%（対象ライセンス・対象リポジトリ） |
| GITHUB_TOKEN既定Read-only                   |                  100%                  |
| CODEOWNERS設定率                            |                95%以上                 |
| 必須CIの成功率                              |                95%以上                 |
| テンプレート最新版から1世代以内の割合       |                95%以上                 |
| 期限切れ例外                                |                  0件                   |
| Critical脆弱性の期限超過                    |               0件を目標                |

注意： - KPIは単純な有効／無効だけでなく、実際に検査が成功しているか、アラートが対応されているかも見る。 - Archivedリポジトリ、サンプル、実験用リポジトリ等は母数から除外するか、別区分にする。 - ライセンス対象外の機能を分母に含めない。

------------------------------------------------------------------------

# 13. 導入ロードマップ

## Phase 0：現状調査・標準化

- 全リポジトリを棚卸しする。
- 言語、フレームワーク、デプロイ方式、重要度、Ownerを整理する。
- 現行Rulesets、Branch Protection、Actions設定を調査する。
- GitHub Code Security／Secret Protectionのライセンス状況を確認する。
- 現行パイプラインと利用Actionを調査する。
- 共通化対象とワークロード固有対象を分類する。

## Phase 1：標準設計・テンプレート開発

- 共通テンプレートを作成する。
- ワークロード別テンプレートを作成する。
- README、CODEOWNERS、SECURITY.md等を整備する。
- CI、テスト、Build、Dependency Review等を実装する。
- Security Configurationsを定義する。
- Organization Rulesetsを定義する。
- Actions Allowlist、SHA固定、GITHUB_TOKEN権限を定義する。
- テンプレートのバージョン管理方式を決定する。

## Phase 2：パイロット

- 主要言語・主要ワークロードを含む10〜20リポジトリを選ぶ。
- テンプレートから新規リポジトリを作成する。
- Rulesets・Security Configurationsの適用を確認する。
- CIの実行時間、失敗率、開発者の負荷を測定する。
- セキュリティ検査の誤検知・ノイズを確認する。
- テンプレートの改善を行う。

## Phase 3：公開・段階展開

- テンプレートを正式公開する。
- 利用ガイド・FAQを公開する。
- リポジトリ作成フローを自動化する。
- 新規リポジトリへの適用を標準化する。
- 既存リポジトリへ段階的に適用する。
- 未準拠リポジトリを可視化する。

## Phase 4：継続運用

- 月次でテンプレートを更新する。
- 更新PRを自動配布する。
- Security Overview等で準拠状況を確認する。
- 例外の期限を管理する。
- KPIを定期的に確認する。
- GitHubの新機能・仕様変更を標準に反映する。

------------------------------------------------------------------------

# 14. 最終的に設定する内容チェックリスト

以下を実装タスクとして利用する。

## 14.1 Organization／Enterprise設定

- [ ] Repository VisibilityはPrivateを基本とする。
- [ ] Repository Creation権限を限定する。
- [ ] Organization Base PermissionをNoneまたはReadにする。
- [ ] GitHub Team単位でRepository権限を付与する。
- [ ] Outside Collaboratorを原則禁止または申請制にする。
- [ ] Repository Admin権限を限定する。
- [ ] Private Repository Forkのポリシーを定義する。
- [ ] Organization Rulesetsを作成する。
- [ ] `main`ブランチにPR必須ルールを適用する。
- [ ] Approveを必須化する。
- [ ] CODEOWNERSレビューを必須化する。
- [ ] Conversation Resolutionを必須化する。
- [ ] 必須Status Checkを設定する。
- [ ] Force Pushを禁止する。
- [ ] Branch Deletionを禁止する。
- [ ] RulesetsのBypass権限を限定する。
- [ ] リリースタグの保護ルールを設定する。
- [ ] GitHub Actionsの利用可能ActionをAllowlist化する。
- [ ] ActionをFull-length Commit SHAに固定する。
- [ ] GITHUB_TOKENの既定権限をRead-onlyにする。
- [ ] Workflow／Job単位で必要な権限だけを明示する。
- [ ] ActionsによるPR作成・Approveを原則禁止する。
- [ ] Self-hosted Runnerの利用条件を定義する。
- [ ] Repository Custom Propertiesを定義する。
- [ ] 必須Custom Propertiesを設定する。

## 14.2 Security Configurations

- [ ] Dependency Graphを有効化する。
- [ ] Dependabot Alertsを有効化する。
- [ ] Dependabot Security Updatesを有効化する。
- [ ] Secret Scanningを有効化する。
- [ ] Push Protectionを有効化する。
- [ ] CodeQL Default Setupを有効化する。
- [ ] 必要に応じてCodeQL Advanced Setupを定義する。
- [ ] Dependency Reviewを有効化する。
- [ ] Security Configurationを新規リポジトリの既定値に設定する。
- [ ] 既存リポジトリにもSecurity Configurationを適用する。
- [ ] Security ConfigurationのEnforcementを設定する。
- [ ] ライセンス対象リポジトリと対象外リポジトリを整理する。

## 14.3 テンプレートリポジトリ

- [ ] 共通テンプレートを作成する。
- [ ] 必要なワークロード別テンプレートを作成する。
- [ ] Template Repository設定を有効にする。
- [ ] テンプレートリポジトリをPrivateにする。
- [ ] 利用者にRead権限を付与する。
- [ ] テンプレート変更権限をDevSecOpsチームに限定する。
- [ ] READMEを作成する。
- [ ] CONTRIBUTING.mdを作成する。
- [ ] SECURITY.mdを作成する。
- [ ] CHANGELOGを作成する。
- [ ] TEMPLATE_VERSIONを作成する。
- [ ] CODEOWNERSを作成する。
- [ ] Pull Request Templateを作成する。
- [ ] Dependabot設定を作成する。
- [ ] CI Workflowを作成する。
- [ ] Dependency Review Workflowを作成する。
- [ ] Build／Test／Lintを実装する。
- [ ] 必要に応じてContainer Scanを実装する。
- [ ] 必要に応じてIaC Scanを実装する。
- [ ] リリース対象にSBOM生成を実装する。
- [ ] リリース対象にArtifact Attestationsを実装する。
- [ ] 本番デプロイにEnvironment保護とOIDCを実装する。

## 14.4 公開・利用手順

- [ ] テンプレート利用ガイドを作成する。
- [ ] 新規リポジトリ作成手順を作成する。
- [ ] 作成時のCustom Properties登録を必須化する。
- [ ] チーム権限付与を自動化する。
- [ ] Security Configuration適用を確認する。
- [ ] Environment作成を自動化または標準化する。
- [ ] 初回CIの成功を確認する。
- [ ] テンプレートバージョンを記録する。

## 14.5 公開後の運用

- [ ] SemVerでテンプレートをバージョン管理する。
- [ ] GitHub ReleaseとCHANGELOGを管理する。
- [ ] 更新対象リポジトリをCustom Propertiesで特定する。
- [ ] GitHub App等で更新PRを自動作成する。
- [ ] 更新PRを段階的に配布する。
- [ ] テンプレート利用率を可視化する。
- [ ] テンプレートバージョン別の利用数を可視化する。
- [ ] Rulesets準拠率を可視化する。
- [ ] Security Configuration適用率を可視化する。
- [ ] 脆弱性対応状況を可視化する。
- [ ] 例外申請・承認・期限管理を整備する。
- [ ] 月次・四半期の運用レビューを実施する。

------------------------------------------------------------------------

# 15. 重要な設計判断・未確定事項

以下は実装開始前に確定する必要がある。

| 論点                 | 決定事項                                    |
|:---------------------|:--------------------------------------------|
| GitHub契約           | GHECの契約プラン・追加ライセンス            |
| Code Security        | 5,000リポジトリのうち対象範囲・ライセンス数 |
| Secret Protection    | Secret Scanning／Push Protectionの対象範囲  |
| 言語・フレームワーク | Java、Node.js、Python、Go等の利用状況       |
| ワークロード         | コンテナ、サーバレス、VM、IaCの対象範囲     |
| Runner               | GitHub-hosted／Self-hostedの使い分け        |
| Container Registry   | GHCR、ECR等の利用先                         |
| Cloud認証            | OIDCの対象クラウド・Trust Policy            |
| SBOM                 | SPDX／CycloneDX等の標準                     |
| Artifact Attestation | 署名・検証の対象成果物と検証ポリシー        |
| DAST                 | 対象アプリケーション、実行環境、利用ツール  |
| 脆弱性SLA            | 重大度別の対応期限                          |
| テンプレート更新     | 更新PRの自動配布対象・承認者・期限          |
| 例外管理             | 承認者、代替統制、期限、再評価方法          |

------------------------------------------------------------------------

# 16. 標準・フレームワークとの対応

## 16.1 NIST SSDF（SP 800-218）

| SSDF Practice Group               | 本設計との対応                                             |
|:----------------------------------|:-----------------------------------------------------------|
| PO：Prepare the Organization      | 全社標準、役割・責任、テンプレート管理、教育               |
| PS：Protect the Software          | Repository権限、Secret管理、Actions制御、署名・Attestation |
| PW：Produce Well-Secured Software | CI/CD、SAST、SCA、IaC Scan、Container Scan、テスト         |
| RV：Respond to Vulnerabilities    | Dependabot、Code Scanning、脆弱性SLA、例外・修正管理       |

## 16.2 OWASP Top 10 CI/CD Security Risks

| OWASPリスク                                       | 主な対策                                               |
|:--------------------------------------------------|:-------------------------------------------------------|
| CICD-SEC-1 Insufficient Flow Control              | Rulesets、PR必須、承認、必須Status Check               |
| CICD-SEC-2 Inadequate IAM                         | Team権限、最小権限、OIDC、GITHUB_TOKEN制限             |
| CICD-SEC-3 Dependency Chain Abuse                 | Action SHA固定、Dependabot、Dependency Review          |
| CICD-SEC-4 Poisoned Pipeline Execution            | PRイベントの権限制限、Secret分離、Runner分離           |
| CICD-SEC-5 Insufficient PBAC                      | Job単位の権限、Environment保護                         |
| CICD-SEC-6 Credential Hygiene                     | Secret Scanning、Push Protection、OIDC、ローテーション |
| CICD-SEC-7 Insecure System Configuration          | Rulesets、Security Configurations、Runner標準          |
| CICD-SEC-8 Ungoverned Third-party Services        | Action Allowlist、承認・棚卸し                         |
| CICD-SEC-9 Improper Artifact Integrity Validation | SBOM、Artifact Attestations、検証                      |
| CICD-SEC-10 Insufficient Logging and Visibility   | Security Overview、監査ログ、KPI                       |

## 16.3 SLSA

- Build Provenanceを生成する。
- ビルド環境・Workflow・ソースCommitを追跡可能にする。
- 成果物の由来を検証する。
- ビルド環境の分離・改ざん耐性を段階的に高める。
- SLSA Levelを一律に宣言するのではなく、実際の要件充足状況に基づいて評価する。

------------------------------------------------------------------------

# 17. 参考資料

以下は設計時に参照したGitHub公式ドキュメントおよび標準資料である。GitHubの機能・ライセンス・Preview状態は変更されるため、実装時に最新情報を再確認する。

## GitHub Template Repository／Workflow Templates

- [Creating a template repository](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-template-repository)
- [Creating a repository from a template](https://docs.github.com/en/repositories/creating-and-managing-repositories/creating-a-repository-from-a-template)
- [Creating workflow templates for your organization](https://docs.github.com/en/actions/how-tos/reuse-automations/create-workflow-templates)
- [Reusing workflow configurations](https://docs.github.com/en/actions/reference/workflows-and-actions/reusing-workflow-configurations)

## GitHub Rulesets／Repository Governance

- [About rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
- [Creating rulesets for repositories in your organization](https://docs.github.com/en/organizations/managing-organization-settings/creating-rulesets-for-repositories-in-your-organization)
- [Available rules for rulesets](https://docs.github.com/en/enterprise-cloud/latest/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/available-rules-for-rulesets)
- [Managing custom properties for repositories in your organization](https://docs.github.com/en/organizations/managing-organization-settings/managing-custom-properties-for-repositories-in-your-organization)
- [Repository roles for an organization](https://docs.github.com/en/organizations/managing-user-access-to-your-organizations-repositories/managing-repository-roles/repository-roles-for-an-organization)

## GitHub Actions Security

- [Disabling or limiting GitHub Actions for your organization](https://docs.github.com/en/organizations/managing-organization-settings/disabling-or-limiting-github-actions-for-your-organization)
- [Security for GitHub Actions](https://docs.github.com/en/actions/how-tos/secure-your-work)
- [Workflow dependency lockfiles](https://docs.github.com/en/actions/concepts/security/workflow-dependency-lockfiles)
- [Configuring OpenID Connect in cloud providers](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-cloud-providers)
- [Artifact attestations](https://docs.github.com/en/actions/concepts/security/artifact-attestations)

## GitHub Security

- [Enabling security features at scale](https://docs.github.com/en/code-security/concepts/security-at-scale/organization-security)
- [Creating a custom security configuration](https://docs.github.com/en/code-security/how-tos/secure-at-scale/configure-organization-security/establish-complete-coverage/create-custom-configuration)
- [Configuring default setup for code scanning](https://docs.github.com/en/code-security/how-tos/find-and-fix-code-vulnerabilities/configure-code-scanning/configure-code-scanning)
- [Configuring default setup for code scanning at scale](https://docs.github.com/en/code-security/how-tos/secure-at-scale/configure-organization-security/configure-specific-tools/code-scanning-at-scale)
- [Enabling secret scanning](https://docs.github.com/en/code-security/how-tos/secure-your-secrets/detect-secret-leaks/enable-secret-scanning)
- [Configuring Dependabot alerts](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/configure-dependabot-alerts)
- [Configuring Dependabot security updates](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/configure-security-updates)
- [Enforcing dependency review across an organization](https://docs.github.com/en/enterprise-cloud/latest/code-security/how-tos/secure-at-scale/configure-organization-security/configure-specific-tools/enforce-dependency-review)

## Security Standards

- [NIST SP 800-218: Secure Software Development Framework](https://csrc.nist.gov/pubs/sp/800/218/final)
- [NIST Secure Software Development Framework](https://csrc.nist.gov/projects/ssdf)
- [OWASP Top 10 CI/CD Security Risks](https://owasp.org/projects/top-10-cicd-security-risks)
- [SLSA Build Track](https://slsa.dev/spec/v1.0/levels)
- [SLSA Build Provenance](https://slsa.dev/spec/v1.0/provenance)

------------------------------------------------------------------------

# 18. 最終提言

5,000人・5,000リポジトリ規模でDevSecOpsテンプレートを運用する場合、最も重要なのはテンプレートそのものよりも、テンプレートを中心とした標準化・統制・更新の仕組みである。

推奨する実装方針は以下のとおり。

1.  **テンプレートはGitHub Template Repositoryとして公開する。**
2.  **Organization Rulesetsでブランチ・PR・レビューを統制する。**
3.  **Security Configurationsでセキュリティ機能を一括適用する。**
4.  **ActionsのAllowlist、SHA固定、GITHUB_TOKEN Read-onlyを全社標準にする。**
5.  **共通テンプレートとワークロード別テンプレートを分ける。**
6.  **テンプレートにはCI/CD、テスト、スキャン、SBOM、Attestation等のファイルを含める。**
7.  **新規リポジトリ作成時にCustom Properties、権限、Environment等を自動設定する。**
8.  **テンプレートの更新はGitHub App等でPull Requestとして配布する。**
9.  **利用状況・準拠状況・脆弱性対応状況を定期的に可視化する。**
10. **例外管理とバージョン管理を運用プロセスに組み込む。**

この構成により、テンプレートを開発者の生産性向上に活用しながら、全社共通のセキュリティ統制をGitHub標準機能で維持できる。
