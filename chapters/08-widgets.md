# 第8章：ウィジェット

執筆者：卞宜璇
最終更新：2026-06-03

## この章で学ぶこと

この章では、アプリを開かなくてもiPhoneのホーム画面に情報を表示できる機能 **「WidgetKit（ウィジェット）」** の作り方を学びます。

具体的には、毎日異なる名言（モチベーションが上がる言葉）を日替わりで表示するアプリを題材にします。ウィジェットが自動で更新されるスケジュールを作る `TimelineProvider` の仕組み、小さいサイズ（Small）と中くらいのサイズ（Medium）で画面レイアウトを切り替える方法、そしてアプリとウィジェットを連携させるための基礎知識を学びます。

---

## 模範コードの全体像

```swift
// ============================================
// 第8章：ウィジェットを作る
// ============================================
// 今日の名言をホーム画面に表示するウィジェットです。
// メインアプリとウィジェットの両方のコードを含みます。
//
// 【セットアップ手順】
// 1. Xcodeで File → New → Target → Widget Extension を選択
// 2. 「Include Configuration App Intent」のチェックを外す
// 3. Widget Extensionの名前を「QuoteWidget」にする
// 4. メインアプリとウィジェットで App Group を設定する
//    （Signing & Capabilities → App Groups）
// ============================================

// ============================================
// ■ メインアプリ側のコード（ContentView.swift）
// ============================================

import SwiftUI

// MARK: - 名言データ（アプリとウィジェットで共有）

struct Quote: Identifiable, Codable {
    let id: Int
    let text: String
    let author: String
}

struct QuoteStore {
    static let quotes: [Quote] = [
        Quote(id: 1, text: "為せば成る、為さねば成らぬ何事も", author: "上杉鷹山"),
        Quote(id: 2, text: "千里の道も一歩から", author: "老子"),
        Quote(id: 3, text: "継続は力なり", author: "ことわざ"),
        Quote(id: 4, text: "失敗は成功のもと", author: "ことわざ"),
        Quote(id: 5, text: "知ることは愛することの始まりである", author: "ことわざ"),
        Quote(id: 6, text: "学びて思わざれば則ち罔し", author: "孔子"),
        Quote(id: 7, text: "過ちて改めざる、死を過ちと謂う", author: "孔子"),
    ]

    static func todaysQuote() -> Quote {
        let dayOfYear = Calendar.current.ordinality(of: .day, in: .year, for: Date()) ?? 0
        let index = dayOfYear % quotes.count
        return quotes[index]
    }
}

// MARK: - メインアプリのContentView

struct ContentView: View {
    let todaysQuote = QuoteStore.todaysQuote()
    @State private var allQuotes = QuoteStore.quotes

    var body: some View {
        NavigationStack {
            VStack(spacing: 24) {
                // 今日の名言（ハイライト）
                VStack(spacing: 16) {
                    Text("今日の名言")
                        .font(.caption)
                        .foregroundStyle(.secondary)

                    Text("「\(todaysQuote.text)」")
                        .font(.title2)
                        .bold()
                        .multilineTextAlignment(.center)

                    Text("— \(todaysQuote.author)")
                        .font(.subheadline)
                        .foregroundStyle(.secondary)
                }
                .padding(24)
                .frame(maxWidth: .infinity)
                .background(
                    RoundedRectangle(cornerRadius: 16)
                        .fill(.blue.opacity(0.08))
                )
                .padding(.horizontal)

                // 全名言リスト
                List(allQuotes) { quote in
                    VStack(alignment: .leading, spacing: 4) {
                        Text(quote.text)
                            .font(.body)
                        Text("— \(quote.author)")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                    .padding(.vertical, 4)
                }
            }
            .navigationTitle("名言集")
        }
    }
}

#Preview {
    ContentView()
}


// ============================================
// ■ ウィジェット側のコード（QuoteWidget.swift）
// ============================================
// ※ Widget Extension ターゲット内のファイルに記述します。
// ============================================

import WidgetKit
import SwiftUI

// MARK: - タイムラインエントリ

struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

// MARK: - タイムラインプロバイダ

struct QuoteProvider: TimelineProvider {
    // プレースホルダー（読み込み中の仮表示）
    func placeholder(in context: Context) -> QuoteEntry {
        QuoteEntry(
            date: Date(),
            quote: Quote(id: 0, text: "読み込み中...", author: "")
        )
    }

    // スナップショット（ウィジェットギャラリーでのプレビュー）
    func getSnapshot(in context: Context, completion: @escaping (QuoteEntry) -> Void) {
        let entry = QuoteEntry(
            date: Date(),
            quote: QuoteStore.todaysQuote()
        )
        completion(entry)
    }

    // タイムライン（実際のウィジェット更新スケジュール）
    func getTimeline(in context: Context, completion: @escaping (Timeline<QuoteEntry>) -> Void) {
        let currentDate = Date()
        let quote = QuoteStore.todaysQuote()
        let entry = QuoteEntry(date: currentDate, quote: quote)

        // 次の日の0時にウィジェットを更新する設定
        let tomorrow = Calendar.current.startOfDay(
            for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
        )

        let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
        completion(timeline)
    }
}

// MARK: - ウィジェットのビュー

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    @Environment(\.widgetFamily) var family

    var body: some View {
        switch family {
        case .systemSmall:
            smallWidget
        case .systemMedium:
            mediumWidget
        default:
            mediumWidget
        }
    }

    // 小サイズ（正方形）
    var smallWidget: some View {
        VStack(spacing: 4) {
            Image(systemName: "quote.opening")
                .font(.caption)
                .foregroundStyle(.blue)

            Text(entry.quote.text)
                .font(.caption)
                .bold()
                .multilineTextAlignment(.center)
                .lineLimit(3)

            Text(entry.quote.author)
                .font(.caption2)
                .foregroundStyle(.secondary)
        }
        .padding(12)
    }

    // 中サイズ（横長）
    var mediumWidget: some View {
        HStack(spacing: 16) {
            Image(systemName: "quote.opening")
                .font(.title)
                .foregroundStyle(.blue)

            VStack(alignment: .leading, spacing: 4) {
                Text("今日の名言")
                    .font(.caption2)
                    .foregroundStyle(.secondary)

                Text(entry.quote.text)
                    .font(.subheadline)
                    .bold()

                Text("— \(entry.quote.author)")
                    .font(.caption)
                    .foregroundStyle(.secondary)
            }

            Spacer()
        }
        .padding()
    }
}

// MARK: - ウィジェット定義

@main
struct QuoteWidget: Widget {
    let kind: String = "QuoteWidget"

    var body: some WidgetConfiguration {
        StaticConfiguration(kind: kind, provider: QuoteProvider()) { entry in
            QuoteWidgetEntryView(entry: entry)
                .containerBackground(.fill.tertiary, for: .widget)
        }
        .configurationDisplayName("今日の名言")
        .description("日替わりで名言を表示します")
        .supportedFamilies([.systemSmall, .systemMedium])
    }
}

// MARK: - プレビュー

#Preview(as: .systemMedium) {
    QuoteWidget()
} timeline: {
    QuoteEntry(date: .now, quote: QuoteStore.todaysQuote())
}
```

📌 **このアプリは何をするものか：**

メインアプリを開くと、登録されている名言のリストと「今日の名言」が1つ大きく表示されます。
そして、このアプリの最大の機能は「ホーム画面のウィジェット」です。ユーザーがホーム画面にウィジェットを追加すると、アプリを開かなくても、今日の名言がいつでも自動的に表示されます。この表示は毎日夜の0時（明日の始まり）になると自動的に切り替わる仕組みになっています。また、ウィジェットのサイズが小さいとき（Small）と横長のとき（Medium）で、文字がはみ出さないように自動でレイアウトが変わるようになっています。

---

## コードの詳細解説

### TimelineProviderの仕組み

```swift
let currentDate = Date()
let quote = QuoteStore.todaysQuote()
let entry = QuoteEntry(date: currentDate, quote: quote)

let tomorrow = Calendar.current.startOfDay(
    for: Calendar.current.date(byAdding: .day, value: 1, to: currentDate)!
)
let timeline = Timeline(entries: [entry], policy: .after(tomorrow))
completion(timeline)
```

* **何をしているか：** ウィジェットが次にいつ新しくなるかという「スケジュール表（タイムライン）」を作っています。現在の時間（ `currentDate` ）の名言をセットしたあと、 `tomorrow` （次の日の夜0時）を計算して、その時間になったら新しく更新するようにOSへ伝えています。
* **なぜこう書くのか：** 今回のアプリは「日替わり（1日に1回）」で名言が変わる仕様だからです。1時間ごとに何度も新しくする必要がないため、明日の0時を指定するのが一番効率がよく、iPhoneのバッテリーにも優しいためです。
* **もしこう書かなかったら：** 更新するタイミングを設定しないと、最初に表示された名言からずっと変わらなくなってしまい、次の日になっても古い名言が表示されたままになってしまいます。

---

### TimelineEntryとウィジェットビュー

```swift
struct QuoteEntry: TimelineEntry {
    let date: Date
    let quote: Quote
}

struct QuoteWidgetEntryView: View {
    var entry: QuoteProvider.Entry
    // ...
    Text(entry.quote.text)
}
```

* **何をしているか：** `QuoteEntry` は、ウィジェットが表示される「時間（ `date` ）」と、その時に表示する「データ（ `quote` ）」をセットにするための構造体です。そして、画面を作る `QuoteWidgetEntryView` がそのデータを受け取って、文字（ `Text` ）として画面に表示しています。
* **なぜこう書くのか：** WidgetKitのルールで、ウィジェットに送るデータは必ず `TimelineEntry` というプロトコルに従って、 `date` という名前のプロパティを持たなければいけないと決まっているからです。
* **もしこう書かなかったら：** ルール通りに書かないとエラーになり、ウィジェットのシステムにデータを送ることができなくなります。

---

### ウィジェットサイズごとのレイアウト

```swift
switch family {
case .systemSmall:
    smallWidget
case .systemMedium:
    mediumWidget
default:
    mediumWidget
}
```

* **何をしているか：** ユーザーがホーム画面に置いたウィジェットのサイズ（小さい正方形か、横長の長方形か）をチェックして、表示するデザイン（ビュー）を切り替えています。
* **なぜこう書くのか：** 小さいサイズ（Small）は画面がとても狭いため、文字を縦に並べてコンパクトに収める必要があります。中サイズ（Medium）は横に広いので、 `HStack` を使ってアイコンと文章を横並びにするなど、サイズに合わせて一番綺麗に見えるデザインにするためです。
* **もしこう書かなかったら：** どのサイズでも同じデザインになってしまい、小さい画面で文字がはみ出したり、大きい画面で不自然な隙間ができたりしてしまいます。

---

### メインアプリとの連携（セットアップ）

```swift
// 【セットアップ手順】より
// 4. メインアプリとウィジェットで App Group を設定する
```

* **何をしているか：** メインアプリのターゲットと、ウィジェットのターゲットの「両方」で同じ名言のデータ（ `QuoteStore` ）を共通して使えるように、特別なデータ共有の設定（App Groups）をしています。
* **なぜこう書くのか：** iOSのセキュリティルールにより、別々のターゲット同士は通常データの共有ができないようになっています。お互いが同じデータを共有して連動するために、この設定が必要になります。
* **もしこう書かなかったら：** メインアプリ側で用意している名言リストのデータをウィジェット側が読み込むことができなくなり、ウィジェットが正しく表示されなくなります。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
| :--- | :--- | :--- |
| **`TimelineProvider`** | ウィジェットをいつ、どんなデータで更新するかというスケジュールを作るプロトコル。 | `struct QuoteProvider: TimelineProvider { ... }` |
| **`@main (Widget)`** | ウィジェットのプログラムがスタートする一番最初の入り口（エントリポイント）を示す注釈。 | `@main struct QuoteWidget: Widget { ... }` |
| **`@Environment(\.widgetFamily)`** | ユーザーが選択したウィジェットの現在のサイズ（Small, Mediumなど）を取得するための環境変数。 | `@Environment(\.widgetFamily) var family` |
| **`containerBackground`** | iOS 17以降のウィジェットで背景の色や形を安全に設定するための新しいAPI。 | `.containerBackground(.fill.tertiary, for: .widget)` |
| **`Calendar.current.ordinality`** | 「今年の通算何日目か」を数値で計算できるメソッド。日替わりのインデックス計算に使用。 | `Calendar.current.ordinality(of: .day, in: .year, for: Date())` |

---

## 自分の実験メモ

### 🧪 実験1：名言のリストに新しい言葉を追加してみた
* **やったこと：** `QuoteStore.quotes` の配列の中に、自分が好きなオリジナルの言葉 `Quote(id: 8, text: "一歩ずつ進めば、必ずゴールに着く", author: "自分")` を追加してみた。
* **結果：** アプリのリスト画面に新しく追加され、日付が変わったときの計算（％）にも正しく組み込まれることを確認した。
* **わかったこと：** データの配列（モデル）を増やすだけで、メインアプリもウィジェットも自動的に対応してくれるので、プログラムの設計がとても綺麗にできていると分かった。

### 🧪 実験2：ウィジェットのサイズから「小さいサイズ」を消してみた
* **やったこと：** `QuoteWidget` の一番下にある `.supportedFamilies([.systemSmall, .systemMedium])` の中から、 `.systemSmall` を削除してビルドしてみた。
* **結果：** ホーム画面でウィジェットを追加する画面（ギャラリー）のとき、小さい正方形の選択肢が消えて、横長サイズ（Medium）しか選べなくなった。
* **わかったこと：** アプリのデザインに合わせて、「このサイズだけで使ってほしい」というときは、コード側で制限をかけることでユーザーの表示崩れを防げると分かった。

---

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `policy: .after(tomorrow)` の `tomorrow` はどうやって計算しているのですか？
   * **得られた理解：** `Calendar.current.startOfDay` を使うことで、ただの24時間後ではなく「次の日の朝の0時0分0秒」という一日のスタート地点をぴったり計算しているのだと分かりました。

2. **質問：** `QuoteStore` のコードは、アプリ側とウィジェット側のどちらに書けばいいですか？
   * **得られた理解：** 両方のプログラムで同じデータを使うため、ファイルの所属設定（Target Membership）で両方にチェックを入れるか、同じコードをコピーして共有する必要があると分かり、ターゲットの概念がクリアになりました。

3. **質問：** `lineLimit(3)` は何のために使っているのですか？
   * **得られた理解：** 小さいウィジェット（Small）のときに、名言の文章が長すぎると画面を突き抜けてしまうため、最大3行までで自動で止めて、見た目が壊れないように守るための大切な設定だと分かりました。

---

## この章のまとめ

この章では、アプリの画面を飛び出して、iPhoneのホーム画面にデータを表示する **「WidgetKit」** の基本を学びました。

* **タイムラインという新しい考え方：** アプリのようにボタンが押されたら動くのではなく、「未来の更新スケジュール表をあらかじめ作ってOSに渡しておく」という独自の仕組みは、スマートフォンのバッテリーを長持ちさせるためにとても大切な技術なのだと分かり、勉強になりました。
* **学んだことの応用：** 前の章で学んだ「歩数計アプリ」などのデータも、今回の仕組み（App Groups）を使えば「今日の歩数をホーム画面にずっと出すウィジェット」に進化させることができます。作れるアプリの幅が一気にプロのレベルに近づいたと感じて、とても嬉しいです！
