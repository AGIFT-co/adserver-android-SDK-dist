# AdServer Android SDK 導入ガイド

## 1. AdServer Android SDK とは

自社アドサーバー（AdServer）からバナー広告を取得・表示し、VIEW / CLICK を計測する Android SDK です。
広告の読み込み導線は 2 経路あります。

1. **直接配信 API（`AdServerBannerView`）** — AdMob を介さず、当社サーバーから直接バナーを取得・描画・計測する（推奨）
2. **AdMob カスタムイベント（メディエーション）** — 既存の AdMob ウォーターフォールに組み込む（後方互換）

### モジュール構成（2 分割）

| モジュール | 内容 | GoogleMobileAds 依存 |
|---|---|---|
| `jp.co.agift.adserver:sdk`（コア） | 直接配信 API・計測・ログ・同意 | **なし** |
| `jp.co.agift.adserver:sdk-admob`（アダプタ・**現在未配布**） | コア + AdMob カスタムイベント | あり（`play-services-ads`） |

- **直接配信だけ使う**なら、コア（`sdk`）のみ依存すれば `play-services-ads` を引き込みません（AD_ID を要求しない）。
- **AdMob メディエーション経由**（`sdk-admob`）は**現在配布していません**。提供を開始する際は当社から案内します（当面の提供は直接配信のみです）。

### 主な機能

- AdMob 非依存の直接バナー描画（`AdServerBannerView`）
- 自動リフレッシュ・指数バックオフ・画面外/バックグラウンドでの停止（広告ライフサイクル）
- MRC 準拠のビューアビリティ計測（広告面の 50% 以上が連続 1 秒で VIEW）
- ログ送信の堅牢化（再送キュー・冪等キー・オフライン耐性・再起動後再送）
- 同意連携（同意がある間のみ PPID を送信、撤回で破棄）

## 2. 動作要件

- minSdk: 24 (Android 7.0)
- AdMob 経路を使う場合のみ Google Mobile Ads SDK (`play-services-ads`) 23.x

## 3. インストール

### 3-1. Maven リポジトリの追加

本 SDK は **Maven Central** で配布します。`mavenCentral()` は Android プロジェクトの標準設定のため、**追加のリポジトリ URL は不要**です。`settings.gradle.kts` に以下が含まれていることだけ確認してください（通常は既定で入っています）。

```kotlin
dependencyResolutionManagement {
    repositories {
        google()        // GoogleMobileAds 等（AdMob 経路で使用）
        mavenCentral()  // 当社 SDK はここから取得（専用 URL の追加は不要）
    }
}
```

### 3-2. 依存関係の追加

`app/build.gradle.kts` に、利用する経路に応じて追加します。

```kotlin
dependencies {
    // 直接配信（現在の提供形態）
    implementation("jp.co.agift.adserver:sdk:0.0.1")

    // AdMob メディエーションアダプタ（sdk-admob）は現在配布していません（提供開始時に案内します）
    // implementation("jp.co.agift.adserver:sdk-admob:<version>")
}
```

バージョンは当社が案内するものを**完全一致で固定**してください（現在の案内バージョン: `0.0.1`）。当社は版ごとに動作確認したバージョンを案内します。

> ⚠️ `0.0.+` 等の動的バージョン指定は使用しないでください。検証していない版が自動で取り込まれることを防ぐため、更新は当社からの案内に基づく手動更新をお願いします。

## 4. 初期設定

**`configure()` の呼び出しは省略可能**で、省略した場合も SDK 組み込みの本番エンドポイントに接続します（明示指定が必要な場合のみ呼び出してください）。

```kotlin
override fun onCreate() {
    super.onCreate()
    // 任意: エンドポイントを明示指定する場合のみ（例: 当社案内によるステージング接続）
    // AdServerSDK.configure(adServerBaseURL = "https://.../delivery", logBaseURL = "https://.../log")
}
```

> エンドポイント URL は SDK に組み込み済みです。引数なしの `AdServerSDK.configure()` は
> 組み込みの本番エンドポイントへの明示的なリセットで、呼んでも呼ばなくても挙動は同じです。

## 5. バナー広告の表示（直接配信 API）

### 基本実装

```kotlin
val banner = AdServerBannerView(
    context,
    adUnitId = "<AdUnit の UUID>",
    adSize = AdBannerSize.BANNER,
)
banner.listener = object : AdServerBannerViewListener {
    override fun onAdLoaded(bannerView: AdServerBannerView) {}
    override fun onAdFailedToLoad(bannerView: AdServerBannerView, error: Throwable) {}
    override fun onAdImpression(bannerView: AdServerBannerView) {}
    override fun onAdClicked(bannerView: AdServerBannerView) {}
}
container.addView(banner)
banner.load()
```

Jetpack Compose では `AndroidView { AdServerBannerView(...) }` でラップして使えます。

### 対応バナーサイズ（3 種固定）

| `AdBannerSize` | dp | 用途 |
|---|---|---|
| `BANNER` | 320×50 | 標準バナー |
| `LARGE_BANNER` | 320×100 | ラージバナー |
| `MEDIUM_RECTANGLE` | 300×250 | レクタングル |

`rawValue`（`banner` / `largeBanner` / `mediumRectangle`）は CMS / iOS と共有する枠種別の識別子です。

### 挙動

- 生成時点で指定サイズの**枠を確保**します（レイアウトのガタつき＝CLS を回避）。
- no-fill / エラー時は**枠を畳みません**。`fallbackView` を設定すると、広告が無い間その View を枠内に表示します。
- fill 成功後は既定間隔で**自動リフレッシュ**、失敗 / no-fill は**指数バックオフでリトライ**します。画面外・アプリのバックグラウンド中は停止し、復帰で再開します。
- **サーバー側の配信停止制御（遠隔制御）**: 障害・ポリシー対応のため、当社サーバーの指示で配信を一時停止することがあります。停止指示が `soft` の場合は枠を維持したまま新規取得を停止し、**表示中の広告も取り下げます**（`fallbackView` があればそれを表示）。`hard` の場合は**枠を畳みます**（高さ 0・非表示）。枠の高さを明示制約で固定しているレイアウトでは、畳んだ後も空間が残ることがあります。停止・再開はサーバー側で制御され、アプリの対応は不要です。

### クリエイティブ画像の扱い

- 応答の `width`/`height` を実寸として尊重し、枠と異なる場合はアスペクト比を保って枠内にフィットします。
- **枠より小さいクリエイティブは拡大せず原寸で中央表示**します（引き伸ばし劣化の回避）。
- 画像取得に失敗した場合は代替テキスト（`altText`）を表示します（この間は VIEW を計測しません）。

## 6. 会員 ID（PPID）と同意

### 設定と送信条件

```kotlin
AdServerSDK.setPPID("member_12345") // 会員 ID を保持するだけ（この時点では送信しない）
AdServerSDK.setConsent(true)        // 同意がある間だけ PPID が送信される
```

- **既定は同意オフ**です。`setConsent(true)` が成立している間のみ、`/delivery`・`/log` に `ppid` が付与されます。
- `setConsent(false)`（撤回）で、以降のリクエストから `ppid` を除外し、**未送信ログキュー内の `ppid` も破棄**します（イベント自体は送信）。
- 同意 UI の表示・文言・ストア申告は**アプリ側（パブリッシャー）の責務**です。

> ⚠️ **後方互換に関する重要な注意**: 旧版では `setPPID` だけで PPID が送信されていましたが、本版からは **`setConsent(true)` がない限り PPID は送信されません**。既存実装から更新する場合は同意フローの追加が必要です。

## 7. 個人情報の取り扱いと責務

| 主体 | 責務 |
|---|---|
| アプリ側（パブリッシャー） | 同意 UI の表示・撤回導線・プライバシーポリシー記載・Play Data Safety 申告 |
| SDK 提供者（当社） | 同意状態（`setConsent`）に連動した PPID 送信ゲート・撤回時のデータ破棄 |
| サーバー側（当社） | 受領した PPID・イベントログの保管と管理（同意判定は行いません） |

> 📌 同意ゲートは **SDK 側で完結**します。サーバは同意状態を受領しないため、サーバ側での「無同意 PPID の破棄」は行われません。PPID がサーバへ届くのは `setConsent(true)` が成立している間の送信分のみです。

## 8. 導入チェックリスト（直接配信）

- [ ] `settings.gradle.kts` に `mavenCentral()` と `google()` がある（通常は既定。専用 URL の追加は不要）
- [ ] `implementation("jp.co.agift.adserver:sdk:<version>")` を案内されたバージョンで追加した
- [ ] （任意）エンドポイントを明示指定する場合のみ `AdServerSDK.configure(...)` を `Application.onCreate()` で呼び出している（省略時は組み込みの本番エンドポイントに接続）
- [ ] `AdServerBannerView` を配置し `load()` を呼んでいる
- [ ] PPID を使う場合: 同意フローを実装し `setConsent(true/false)` を連動させた
- [ ] R8/ProGuard を有効化している場合: コンシューマールールが効いていることを確認した（§9）

## 9. ProGuard / R8

コアの公開 API はライブラリ同梱の **consumer ProGuard ルール**で保持されます。通常、アプリ側に追加の keep は不要です。
独自に最小化を強める場合は、以下が保持されていることを確認してください。

```proguard
-keep public class jp.agift.adserver.sdk.AdServerSDK { public *; }
-keep public class jp.agift.adserver.sdk.views.AdServerBannerView { public *; }
-keep public class jp.agift.adserver.sdk.models.** { *; }
```

## 10. よくある質問

**Q: 直接配信と AdMob、どちらを使うべき？**
A: 現在の提供は直接配信（`AdServerBannerView`）のみです。AdMob ウォーターフォールへの組み込み（付録 A）はアダプタ提供開始後に利用できます（提供時期は当社から案内します）。

**Q: 広告が表示されない**
A: `adUnitId` が正しい UUID か確認してください（`AdServerSDK.configure()` は省略可能で、省略時は SDK 組み込みの本番エンドポイントに接続します）。no-fill（在庫なし）の場合は `onAdFailedToLoad` に `AdServerError.NoFill` が渡ります。

**Q: PPID がサーバーに届かない**
A: `setConsent(true)` を呼んでいるか確認してください。同意がない間は仕様上 PPID を送信しません。

**Q: クリックで遷移しない**
A: クリック遷移は http/https のみ許可しています（カスタムスキームは安全のため遮断）。`INTERNET` 権限はコアが宣言します。

---

## 付録 A. AdMob メディエーション経由で利用する場合

> ⚠️ **AdMob アダプタ（`sdk-admob`）は現在 Maven Central では配布していません。**
> 本付録は提供開始後の手順です（提供時期・バージョンは当社から案内します）。

直接配信ではなく、AdMob のウォーターフォールに組み込む場合の手順です。

### A-1. 依存関係

```kotlin
dependencies {
    implementation("jp.co.agift.adserver:sdk-admob:<version>") // 当社案内のバージョンで固定
    implementation("com.google.android.gms:play-services-ads:23.+")
}
```

### A-2. AdMob App ID の設定

`AndroidManifest.xml` の `<application>` 内に AdMob の App ID を追加します（未設定だと起動時クラッシュ）。

```xml
<meta-data
    android:name="com.google.android.gms.ads.APPLICATION_ID"
    android:value="ca-app-pub-XXXXXXXXXXXXXXXX~XXXXXXXXXX" />
```

### A-3. カスタムイベント設定

AdMob ダッシュボード → 広告ユニット → メディエーション → カスタムイベント:

- **クラス名**: `jp.agift.adserver.sdk.adapters.BannerCustomEventAdapter`
- **パラメータ**: 当社の AdUnit ID

### A-4. AD_ID 権限

`sdk-admob` 経由では `play-services-ads` がマニフェストマージで `AD_ID` 権限を付与します。
直接配信のみ（コア `sdk` のみ依存）の場合は付与されません。純直接配信で明示的に除外したい場合:

```xml
<uses-permission android:name="com.google.android.gms.permission.AD_ID" tools:node="remove" />
```

### A-5. app-ads.txt の設置

AdMob で配信するには、デベロッパーウェブサイトのドメイン直下に `app-ads.txt` を設置します（2025年以降、新規アプリで必須）。

1. Google Play Console の「デベロッパーのウェブサイト」にドメインを登録（サブドメイン可）
2. ドメイン直下に配置（サブディレクトリは無効）:
   ```
   google.com, pub-XXXXXXXXXXXXXXXX, DIRECT, f08c47fec0942fa0
   ```
   `pub-...` は AdMob のパブリッシャー ID（AdMob → アカウント情報）
3. `https://登録ドメイン/app-ads.txt` で表示されることを確認
4. AdMob コンソール → アプリ → 「app-ads.txt」タブで認証（反映に最大 24 時間）

> メインドメインに設置できない場合はサブドメイン（`apps.example.com` 等）でも有効です。

### A-6. AdMob 経路の ProGuard

```proguard
-keep public class jp.agift.adserver.sdk.adapters.BannerCustomEventAdapter { *; }
```

（`sdk-admob` 同梱の consumer ルールで保持されます。）

### A-7. AdMob 経路チェックリスト

- [ ] `sdk-admob` を依存に追加した
- [ ] `AndroidManifest.xml` に AdMob App ID を追加した
- [ ] カスタムイベントのクラス名を**完全一致**で設定した
- [ ] `app-ads.txt` を設置し AdMob で認証された
