# AdServerSDK（Android）

当社の広告配信サーバーからバナー広告を取得・表示し、インプレッション / クリックを自動計測する Android SDK です。バイナリ（AAR）は **Maven Central** で配布されます。ソースコードは含まれません。

このリポジトリは配布ドキュメントとライセンスの公開置き場です（バイナリはありません）。

---

## インストール

`settings.gradle(.kts)` のリポジトリに `mavenCentral()` が含まれていることを確認し、依存を追加します。

```kotlin
dependencies {
    // 直接配信のみ使う場合
    implementation("jp.co.agift.adserver:sdk:<version>")

    // AdMob メディエーション（カスタムイベント）を併用する場合はこちら（コアも推移的に入ります）
    implementation("jp.co.agift.adserver:sdk-admob:<version>")
}
```

> ⚠️ `<version>` は当社が案内するバージョンを**完全一致で固定**してください。当社は版ごとに動作確認したバージョンを案内します（動的バージョン `+` は使用しないでください）。

## クイックスタート（直接配信）

```kotlin
// アプリ起動時に一度
AdServerSDK.configure()

// バナーの表示
val banner = AdServerBannerView(
    context,
    adUnitId = "<当社から共有する AdUnit の UUID>",
    adSize = AdBannerSize.BANNER, // 320x50
)
container.addView(banner)
banner.load()
```

表示時の VIEW（インプレッション）とタップ時の CLICK は SDK が自動で計測します。在庫が無い場合（no-fill）も枠は維持され、自動でリトライされます。

## ドキュメント

- [ベンダー向け導入ガイド](docs/VENDOR_GUIDE.md) — 組み込み手順・PPID/同意連携・イベント通知
- [AdMob 併用ポリシー](docs/ADMOB_POLICY.md) — カスタムイベントの設定・注意事項

## ライセンス

本 SDK はプロプライエタリライセンスです。利用には当社との契約が必要です。詳細は [LICENSE](LICENSE) を参照してください。

お問い合わせ: https://agift.co.jp
