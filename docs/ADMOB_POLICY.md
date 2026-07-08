# AdMob ポリシー準拠ガイド

> ⚠️ **AdMob メディエーションアダプタ（`jp.co.agift.adserver:sdk-admob`）は現在 Maven Central では配布していません。**
> 本ドキュメントはアダプタ提供開始後（AdMob 経路を利用する場合）に適用されます。直接配信のみの利用には関係しません。

## 概要

本ドキュメントは、AdServer Android SDKがGoogle AdMobの利用規約および開発者ポリシーに準拠していることを説明します。

## カスタムイベントの利用について

本SDKはAdMobの**カスタムイベント（Custom Event Adapter）**として実装されています。
カスタムイベントはAdMobが公式に提供する仕組みであり、パブリッシャーが独自の広告ネットワークをAdMobのウォーターフォールに組み込むことができます。

### WebViewとの違い

| 比較項目 | WebView方式 | カスタムイベント方式（本SDK） |
|---|---|---|
| AdMob規約準拠 | グレーゾーン | ✅ 公式サポートの仕組み |
| フォールバック | 手動実装が必要 | ✅ AdMobが自動処理 |
| インプレッション計測 | 独自実装 | ✅ AdMobの標準計測 |
| バージョン報告 | なし | ✅ 標準API経由で報告 |

## メディエーション（ウォーターフォール）

AdMobはカスタムイベントが失敗した場合（広告取得エラー、タイムアウト等）、自動的に次の広告ソースにフォールバックします。
本SDKの `BannerCustomEventAdapter` は、AdFetcherの失敗を `callback.onFailure()` で通知するため、フォールバックが正常に動作します。

## SDKの配布について

### バイナリ配布（AARのみ）

本SDKはAARバイナリのみで配布しています。ソースコードは非公開です。
これはGoogle AdMobの規約に定める「SDKのコードをGoogleと共有しない」要件を満たすためです。

### コード難読化・エンドポイントの扱い

配布する AAR ライブラリ自体は難読化していません（`sdk/build.gradle.kts` と `sdk-admob/build.gradle.kts` の release で `isMinifyEnabled = false`）。
これはライブラリ配布のベストプラクティスに沿った意図的な設定で、縮小・難読化は SDK を組み込むアプリ側のリリースビルドの R8 で行われます（keep ルールは `consumer-rules.pro` として AAR に同梱）。

なお、配信 / ログ API のエンドポイント URL は秘匿情報ではありません。クライアント SDK は必ず自分の通信先を含むため URL はネットワーク経由でも観測可能であり、R8 は文字列リテラルを暗号化しません。エンドポイントの保護はクライアント側の難読化ではなく、サーバ側のアクセス制御（オリジン検証・レート制限等）で担保します。

## アダプターバージョン報告

`BannerCustomEventAdapter` は以下のメソッドでバージョン情報をAdMobに報告します：

```kotlin
override fun getVersionInfo(): VersionInfo  // アダプターバージョン
override fun getSDKVersionInfo(): VersionInfo  // SDKバージョン
```

これはAdMobの要件を満たすための実装です。

## パブリッシャー（アプリ開発者）の要件

本SDKを利用するアプリ開発者は以下を遵守してください：

1. **Googleとの直接契約**: AdMobカスタムイベントを使用するには、GoogleとAdMobの利用規約に同意が必要です
2. **ユーザー同意**: PPID（ユーザーメンバーID）を使用する場合は、個人情報保護法に基づきユーザーの同意を取得してください
3. **プライバシーポリシー**: アプリのプライバシーポリシーに、広告SDKによるデータ収集について記載してください

## PPID とプライバシー

PPID（Publisher Provided Identifier）はアプリが設定するオプション機能です。

- 設定された場合、広告リクエストおよびイベントログに含まれます
- PPIDの取得・管理はアプリ側の責任です
- SDKはPPIDの内容を検証しません
- ユーザーがアカウントを削除した場合は `AdServerSDK.setPPID(null)` を呼び出してください

## コンプライアンスチェックリスト

- [x] AdMobカスタムイベントAPIを使用（公式サポートの仕組み）
- [x] フォールバック動作を実装（callback.onFailure() を適切に呼び出す）
- [x] アダプターバージョンとSDKバージョンを報告
- [x] バイナリのみ配布（ソースコード非公開）
- [x] インプレッション・クリックの重複計測防止（hasReportedImpressionフラグ）
