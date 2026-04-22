# 第1章：WebAPIの基本

> 執筆者：iOS学習者
> 最終更新：2026-04-22

## この章で学ぶこと

この章では、インターネット上のサービス（API）からデータを取得して、アプリ内に表示する方法を学ぶ。具体的にはiTunes Search APIを使って音楽を検索し、その結果をリスト表示するアプリを題材にする。JSONの解析（Codable）や非同期処理（async/await）の基礎を身につける。

## 模範コードの全体像

```swift
import SwiftUI

// 1. データモデル (Data Model)
struct ITunesResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    var id: Int { trackId } // trackIdをIdentifiableのidとして利用
    let trackId: Int
    let trackName: String
    let artistName: String
}

// 2. ビューとロジック (View & ViewModel Logic)
struct ContentView: View {
    @State private var searchText = ""
    @State private var songs: [Song] = []
    
    var body: some View {
        NavigationStack {
            VStack {
                // 検索フィールドとボタン
                HStack {
                    TextField("アーティストや曲名を入力", text: $searchText)
                        .textFieldStyle(.roundedBorder)
                    
                    Button("検索") {
                        Task {
                            await searchMusic()
                        }
                    }
                    .buttonStyle(.borderedProminent)
                }
                .padding()
                
                // 検索結果リスト
                List(songs) { song in
                    VStack(alignment: .leading) {
                        Text(song.trackName)
                            .font(.headline)
                        Text(song.artistName)
                            .font(.subheadline)
                            .foregroundColor(.gray)
                    }
                }
            }
            .navigationTitle("iTunes 音楽検索")
        }
    }
    
    // 3. API通信処理
    func searchMusic() async {
        // 日本語や空白をURLで使用可能な形式にエンコードする
        guard let encodedText = searchText.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed),
              let url = URL(string: "https://itunes.apple.com/search?term=\(encodedText)&entity=song&country=jp") else {
            print("無効なURLです")
            return
        }
        
        do {
            // ネットワークリクエストの送信
            let (data, _) = try await URLSession.shared.data(from: url)
            // 取得したJSONデータをSwiftの構造体にデコード
            let decodedResponse = try JSONDecoder().decode(ITunesResponse.self, from: data)
            // 画面を更新
            songs = decodedResponse.results
        } catch {
            print("データの取得に失敗しました：\(error.localizedDescription)")
        }
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

簡単な音楽検索アプリである。画面上部のテキストフィールドにアーティスト名（例：宇多田ヒカル）を入力して「検索」ボタンを押すと、アプリがAppleのiTunesサーバーにリクエストを送り、返ってきた楽曲データ（曲名とアーティスト名）をリスト化して画面に表示する。

## コードの詳細解説

### データモデル（Codable構造体）

```swift
struct ITunesResponse: Codable {
    let results: [Song]
}

struct Song: Codable, Identifiable {
    var id: Int { trackId }
    let trackId: Int
    let trackName: String
    let artistName: String
}
```

**何をしているか：**
iTunes APIから返ってくるJSONデータを受け止めるための「型（カタ）」を定義している。APIは全体を辞書型のオブジェクトとして返し、その中の `results` というキーに配列として楽曲データを入れているため、それに合わせて `ITunesResponse` と `Song` という階層構造を作っている。

**なぜこう書くのか：**
`Codable` プロトコルをつけることで、複雑なJSON文字列をSwiftの構造体に自動で変換（デコード）できるようになり、手作業でパースする手間が省けるため。また、`List` で表示するために `Identifiable` プロトコルを適用し、固有の `id` を持たせている。

**もしこう書かなかったら：**
プロパティ名（例：`trackName`）をタイポしたり `Codable` を付け忘れたりすると、JSONの変換に失敗してエラーとなり、いつまで経っても画面にデータが表示されなくなる。

---

### API通信の処理

```swift
func searchMusic() async {
    // ...中略 (URL生成)
    do {
        let (data, _) = try await URLSession.shared.data(from: url)
        let decodedResponse = try JSONDecoder().decode(ITunesResponse.self, from: data)
        songs = decodedResponse.results
    } catch {
        print("データの取得に失敗しました：\(error.localizedDescription)")
    }
}
```

**何をしているか：**
インターネットを通じてAPIからデータを取得し、Swiftのオブジェクトに変換している。URLSessionを使ってリクエストを送り、得られたデータをJSONDecoderで解析し、最後に@State変数である `songs` に代入して画面を更新している。

**なぜこう書くのか：**
通信には時間がかかるため、`async / await` を使って「非同期処理」として実装している。これにより、データ取得中もアプリの画面がフリーズしない。また、通信エラーや解析エラーに備えて `do-catch` 文で囲み、安全にエラー処理を行っている。

**もしこう書かなかったら：**
メインスレッドで同期的に通信を行うと、応答を待っている間アプリ全体が固まってしまい、ユーザーにアプリがクラッシュしたと誤解されてしまう。

---

### ビューの構成

```swift
Button("検索") {
    Task {
        await searchMusic()
    }
}
```

**何をしているか：**
ユーザーが「検索」ボタンをタップした際に、定義した `searchMusic()` 関数を呼び出している。

**なぜこう書くのか：**
`searchMusic()` は `async` がついた非同期関数なので、通常のボタンのアクション（同期的な環境）から直接呼び出すことはできない。そのため `Task { }` で囲むことで、非同期処理を実行するための新しいタスク（空間）を作り、そこで `await` して呼び出している。

**もしこう書かなかったら：**
`Task { }` なしで直接 `await searchMusic()` を書くと、Xcodeで「非同期関数を同期関数から呼び出すことはできない」というコンパイルエラーになり、アプリを実行すらできなくなる。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| `async/await` | 非同期処理を同期的に（直列に）読みやすく書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
| `Task { }` | 同期的な環境から非同期処理を呼び出すために、新しいタスク空間を生成するブロック | `Task { await searchMusic() }` |
| `addingPercentEncoding` | URLに使えない文字（日本語や空白など）を安全な形式（%エンコーディング）に変換するメソッド | `text.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed)` |

## 自分の実験メモ

**実験1：**
- やったこと：`Song` 構造体の `trackName` をわざと `trackNamee` とスペルミスしてみた。
- 結果：「検索」を押してもリストには何も表示されず、Xcodeのコンソール（`catch` の中身）にデコード失敗のエラーが出力された。
- わかったこと：`Codable` はJSONのキー名と完全一致していないと失敗する厳密な仕様であることがわかった。エラーが起きたときは `catch` の print を見るのがデバッグにおいて非常に大事だと痛感した。

**実験2：**
- やったこと：Mac（またはシミュレータ）のWi-Fiを切り、オフラインの状態で検索ボタンを押してみた。
- 結果：アプリはクラッシュせず、コンソールに「インターネット接続がありません」といったエラーが出力された。
- わかったこと：`do-catch` のおかげで通信に失敗してもアプリが落ちないことがわかった。適切なエラーハンドリングがないと、少しの通信障害でアプリが強制終了してしまう危険性がある。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   なぜ楽曲の配列 `[Song]` を直接取り出さずに、わざわざ `ITunesResponse` という構造体で一階層包む必要があるのですか？
   **得られた理解：**
   APIが返してくる生のJSON構造に合わせる必要があるから。iTunes APIのJSONの一番外側は `{}`（辞書）であり、その中に `results` というキーで配列が入っている。この「一番外側の包装」を剥がすための型として `ITunesResponse` が必要だということがわかり、JSONの階層構造と構造体の関係がスッキリ理解できた。

2. **質問：**
   `URLSession.shared.data(from: url)` の前にはなぜ `try` と `await` の2つが必要なのですか？1つではダメですか？
   **得られた理解：**
   AIの「通信は『待つ必要がある』し、『失敗するかもしれない』から」という説明が分かりやすかった。待つための `await` と、エラー（断線など）を投げるかもしれない処理に対する `try` は、それぞれ全く別の役割を持っているため、両方セットで書く必要があると理解した。

3. **質問：**
   `TextField` で使う `$searchText` の `$` は何ですか？変数名に直接書き込むのとどう違いますか？
   **得られた理解：**
   `$` は「Binding（バインディング）」を表し、「双方向の通信」を実現するものだと分かった。`$` なしだと単なる読み取りになるが、`$` をつけることでTextFieldにユーザーが入力した文字を、即座に変数 `searchText` に「書き戻す」ことができるという仕組みに納得した。

## この章のまとめ

WebAPIを使ってアプリにデータを取り込む際は、以下の「4つの基本ステップ」が重要になる。

1. **JSONの形を知る**：ブラウザ等でAPIのレスポンス構造を確認する。
2. **モデルを作る**：JSONの構造に完全に一致する `Codable` 構造体を作る。
3. **通信処理を書く**：URLを生成し、`URLSession` と `try await` でリクエストする。
4. **エラーに備える**：ネットワークは必ず失敗する前提で `do-catch` で守る。

この流れを身につけたことで、世の中にある様々なリアルデータを自分のアプリに組み込めるようになった。次回は、このコードをさらに管理しやすい「MVVM」という設計に進化させていく。
