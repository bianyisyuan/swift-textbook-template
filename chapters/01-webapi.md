# 第1章：WebAPIの基本

> 執筆者：卞宜璇
> 最終更新：2026-05-22

## この章で学ぶこと

この章では、Swiftのアプリ開発において必須となる **WebAPIとの通信** を行うための準備として、URLの生成と、それに伴うオプショナル型の安全な取り扱いについて学びます。

具体的には、無効な文字列によってURLの生成が失敗（`nil`）し、アプリがクラッシュするのを防ぐための2つの重要文法、**`guard let`（早期退出）** と **`if let`（条件分岐）** の意味と使い分けをマスターします。

---

## 模範コードの全体像

```swift
import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let resultCount: Int
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let collectionName: String?
    let artworkUrl100: String
    let previewUrl: String?
    let trackPrice: Double?
    let currency: String?

    var id: Int { trackId }

    var priceText: String {
        guard let price = trackPrice, let currency = currency else {
            return "価格不明"
        }
        return "\(currency) \(String(format: "%.0f", price))"
    }
}

// MARK: - ViewModel

@Observable
class MusicSearchViewModel {
    var songs: [Song] = []
    var searchText: String = ""
    var isLoading: Bool = false
    var errorMessage: String?

    enum SearchError: LocalizedError {
        case invalidURL
        case networkError(Error)
        case decodingError(Error)
        case noResults

        var errorDescription: String? {
            switch self {
            case .invalidURL:
                return "検索URLの作成に失敗しました"
            case .networkError(let error):
                return "通信エラー: \(error.localizedDescription)"
            case .decodingError:
                return "データの読み取りに失敗しました"
            case .noResults:
                return "検索結果が見つかりませんでした"
            }
        }
    }

    func searchMusic() async {
        guard !searchText.trimmingCharacters(in: .whitespaces).isEmpty else { return }

        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }

        isLoading = true
        errorMessage = nil

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)

            if response.results.isEmpty {
                errorMessage = SearchError.noResults.errorDescription
                songs = []
            } else {
                songs = response.results
            }
        } catch let error as DecodingError {
            errorMessage = SearchError.decodingError(error).errorDescription
            songs = []
        } catch {
            errorMessage = SearchError.networkError(error).errorDescription
            songs = []
        }

        isLoading = false
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var viewModel = MusicSearchViewModel()

    var body: some View {
        NavigationStack {
            VStack(spacing: 0) {
                searchBar

                if let errorMessage = viewModel.errorMessage {
                    ErrorBanner(message: errorMessage)
                }

                contentArea
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - 検索バー

    private var searchBar: some View {
        HStack {
            TextField("アーティスト名を入力", text: $viewModel.searchText)
                .textFieldStyle(.roundedBorder)
                .onSubmit {
                    Task { await viewModel.searchMusic() }
                }

            Button("検索") {
                Task { await viewModel.searchMusic() }
            }
            .buttonStyle(.borderedProminent)
            .disabled(viewModel.searchText.isEmpty || viewModel.isLoading)
        }
        .padding()
    }

    // MARK: - コンテンツエリア

    @ViewBuilder
    private var contentArea: some View {
        if viewModel.isLoading {
            Spacer()
            ProgressView("検索中...")
            Spacer()
        } else if viewModel.songs.isEmpty {
            ContentUnavailableView(
                "曲を検索してみよう",
                systemImage: "music.note",
                description: Text("アーティスト名を入力して検索ボタンを押してください")
            )
        } else {
            List(viewModel.songs) { song in
                NavigationLink(destination: SongDetailView(song: song)) {
                    SongRow(song: song)
                }
            }
        }
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: {
                RoundedRectangle(cornerRadius: 8)
                    .fill(.gray.opacity(0.2))
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)
                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }

            Spacer()

            Text(song.priceText)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding(.vertical, 4)
    }
}

// MARK: - 詳細ビュー

struct SongDetailView: View {
    let song: Song

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                    image.resizable().aspectRatio(contentMode: .fit)
                } placeholder: {
                    ProgressView()
                }
                .frame(width: 200, height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 8)

                Text(song.trackName)
                    .font(.title2)
                    .bold()
                    .multilineTextAlignment(.center)

                Text(song.artistName)
                    .font(.title3)
                    .foregroundStyle(.secondary)

                if let albumName = song.collectionName {
                    Text(albumName)
                        .font(.subheadline)
                        .foregroundStyle(.tertiary)
                }

                Text(song.priceText)
                    .font(.headline)
                    .padding(.horizontal, 16)
                    .padding(.vertical, 8)
                    .background(.blue.opacity(0.1))
                    .clipShape(Capsule())
            }
            .padding()
        }
        .navigationTitle("曲の詳細")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - エラーバナー

struct ErrorBanner: View {
    let message: String

    var body: some View {
        HStack {
            Image(systemName: "exclamationmark.triangle.fill")
                .foregroundStyle(.yellow)
            Text(message)
                .font(.caption)
        }
        .padding(10)
        .frame(maxWidth: .infinity)
        .background(.red.opacity(0.1))
    }
}

#Preview {
    ContentView()
}
```

**📌 このアプリは何をするものか：**

ユーザーが入力したキーワードを元に、WebAPI（例：iTunes APIなど）にリクエストを送信してデータを取得し、リスト形式で画面に表示するアプリです。
通信を行う前提条件として、入力された文字列から安全に「URLオブジェクト」を構築する処理が必要となります。

---

## コードの詳細解説

### 1. `guard let` による安全なURL構築（門番型）

```swift
func openURL() {
    // 門番：URLの生成に失敗したら、elseの中に入って強制終了（return）する
    guard let url = URL(string: "[https://google.com](https://google.com)") else {
        print("URLが無効です")
        return // 必須：ここで処理を終了させる
    }

    // 🟢 門番を突破した場合、これ以降の行で `url` が自由に使える！
    print("URLの生成に成功しました: \(url)")
}
```

* **何をしているか：**
  文字列から `URL` オブジェクトを生成する際、中身が空（`nil`）になる可能性を考慮し、前提条件を満たしているかをチェックしています。
* **なぜこう書くのか：**
  `guard let` は**「門番（ガードマン）」**の役割を果たします。「条件を満たさない場合は今すぐここで処理を終了（`return`）させ、突破した場合のみ、それ以降の行でアンラップされた変数（`url`）を自由に使えるようにする」という設計思想に基づいているためです。
* **もしこう書かなかったら：**
  もし `nil` をチェックせずにそのまま無効なURLを使用すると、アプリがクラッシュ（強制終了）してしまいます。また、後述の `if let` で代用しすぎると、コードの階層が深くなってしまいます。

---

### 2. `if let` による局所的なアンラップ処理（VIPルーム型）

```swift
func doSomething() {
    // VIPルーム：URLが生成できた場合のみ、{ } の中に入れる
    if let url = URL(string: "[https://apple.com](https://apple.com)") {
        // 成功！この { } の中（VIPルーム）だけで url が使える
        print("URLが作れました: \(url)")
    } else {
        // 失敗した時の処理
        print("URLが作れませんでした")
    }
    // ⚠️ 注意：VIPルームの外に出たので、ここではもう `url` は使えない！
}
```

* **何をしているか：**
  `guard let` と同様に `nil` になるかもしれない変数を安全に取り出していますが、こちらは特定のスコープ（`{ }` の中）限定で処理を行っています。
* **なぜこう書くのか：**
  `if let` は**「VIPルーム」**のような役割を果たします。成功した場合の処理を特定のブロック内（`{ }`）だけで完結させたい場合や、「失敗しても関数全体を終了させず、次の処理へ進めたい」という場合に選ばれます。
* **もしこう書かなかったら：**
  すべてを `guard let` で書いてしまうと、エラーが起きた時点で必ず関数から退出（`return`）しなければならなくなるため、「失敗してもスルーして別の処理を続けたい」といった柔軟な制御ができなくなります。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
| :--- | :--- | :--- |
| **`guard let`** | 前提条件を確認し、満たさない場合は即座に関数等から退出（`return`）する構文（門番型）。 | `guard let url = URL(string: str) else { return }` |
| **`if let`** | 成功した場合のみ特定の `{ }` 内で変数を使用し、失敗しても退出しない構文（VIPルーム型）。 | `if let url = URL(string: str) { ... }` |
| **早期退出 (Early Exit)** | 不適切なデータを関数の最初で弾き、メインロジックをスッキリさせるSwiftの推奨パターン。 | `guard` 構文を用いた例外処理そのもの。 |
| **ネスト地獄** | `if let` などを何重にも重ねることで、コードの字下げ（インデント）が右にズレて読みづらくなる状態。 | `if let a = a { if let b = b { ... } }` |

---

## 自分の実験メモ

### 🧪 実験1：APIリクエストのURL構築を `guard let` で書いてみた

* **やったこと：** ユーザーの検索入力文字（`searchText`）をエンコードし、APIのベースURLと結合する処理を `guard let` で記述した。
* **結果：** 入力欄が空の場合や不正な文字が含まれる場合は、即座に安全に処理をストップ（`return`）させることができた。
* **わかったこと：** 「このデータがないと、この先の通信処理をやっても意味がない！」という前提条件の確認に、`guard let` がいかに強力で適しているかが理解できた。

### 🧪 実験2：オプション項目（価格情報など）の表示を `if let` で書き換えてみた

* **やったこと：** APIから返ってきたデータの中で、欠けているかもしれない項目（例：`collectionName` や `trackPrice`）の描画処理を `if let` で囲んだ。
* **結果：** データが存在するときだけUIが表示され、存在しない（`nil`）場合でもアプリはクラッシュせず、画面が崩れることもなかった。
* **わかったこと：** 「失敗しても処理を終わらせず、画面全体の描画を続けたい」という局所的なUIの出し分けには、`if let` のVIPルーム型が最適であると分かった。

---

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：`guard let url = URL(string:...)` という書き方はどんな意味ですか？初学者向けに教えてください。**
   > **得られた理解：** Swiftの安全対策として、無効な文字列によるアプリのクラッシュを防ぐために「中身が確実にあるか」を確認し、安全に取り出すための「門番」の役割をしているという本質が理解できました。

2. **質問：`guard let` と `if let` の変数の有効範囲（スコープ）の違いは何ですか？**
   > **得られた理解：** `guard let` は突破すれば「その後ずっと（記述した行より下）」使えるのに対し、`if let` は「成功した `{ }` の中（VIPルーム）」だけでしか使えないという、明確な有効範囲の違いをコード例を通して納得できました。

3. **質問：実際のコードの読みやすさ（可読性）において、どちらを選ぶべきですか？**
   > **得られた理解：** まずは「ダメならすぐ帰る（早期退出）」をデフォルトとして意識し、`guard let` を使うことで、コードの字下げ（インデント）が深くならず、下へスッキリ伸びる綺麗なSwiftらしいコードが書けると学びました。

---

## この章のまとめ

Swiftにおいて最重要概念の一つである**「オプショナル型の安全な取り扱い（アンラップ）」**について、実践的な使い分けの基準を確立できました。

* **`guard let` は前提条件のチェック専用：** 「このデータがないと始まらない！」という時に使い、ダメならすぐ帰る（早期退出）。これにより、メインロジックを `{ }` で囲む必要がなくなり、下へまっすぐ伸びる読みやすいコードになります。
* **`if let` は局所的な利用専用：** 「失敗しても処理を終わらせず、別の処理を続けたい」ときや、UIの一部を出し分けるときに `{ }` の中だけで一時的に変数を使います。

初心者のための鉄則として、まずは **`guard let` をデフォルトの考え方（ファーストチョイス）** としてクセをつけ、ネスト地獄を回避する綺麗でモダンなSwiftコードを書けるように、今後の章でも常に意識していきます。
