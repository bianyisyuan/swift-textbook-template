第4章：データの永続化
執筆者：卞宜璇
最終更新：2026-06-03

この章で学ぶこと
この章では、アプリを終了したり端末の電源を切ったりしてもデータが消えないようにする「データの永続化」について学びます。SwiftUIにおけるモダンな手法として、アプリの設定情報などの保存に適した AppStorage と、本格的な構造化データの管理に適した SwiftData の2つの使い分けをマスターします。

具体的には、シンプルなメモアプリの作成を通じて、@Model マクロによるデータモデルの定義、データベースの操作を担う modelContext の役割、データを自動で取得して画面を更新する @Query、配置したクラス型データをビューと安全に双方向バインディングする @Bindable の実装フローを体系的に学びます。

模範コードの全体像

import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル
@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

// MARK: - メインビュー
struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        } else {
            return memos
        }
    }

    var body: some View {
        NavigationStack {
            List {
                ForEach(displayedMemos) { memo in
                    NavigationLink(destination: MemoEditView(memo: memo)) {
                        HStack {
                            VStack(alignment: .leading, spacing: 4) {
                                Text(memo.title.isEmpty ? "無題のメモ" : memo.title)
                                    .font(.headline)
                                Text(memo.createdAt, style: .date)
                                    .font(.caption)
                                    .foregroundStyle(.secondary)
                            }
                            Spacer()
                            if memo.isFavorite {
                                Image(systemName: "star.fill")
                                    .foregroundStyle(.yellow)
                            }
                        }
                    }
                }
                .onDelete(perform: deleteMemos)
            }
            .navigationTitle(userName.isEmpty ? "マイメモ" : "\(userName)のメモ")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button(action: addMemo) {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    private func addMemo() {
        let newMemo = Memo(title: "新規メモ", content: "")
        modelContext.insert(newMemo)
    }

    private func deleteMemos(offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(displayedMemos[index])
        }
    }
}

// MARK: - 編集画面（@Bindableの活用）
struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面（AppStorageの活用）
struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}

📌 このアプリは何をするものか：

ユーザーがアプリを起動すると、端末のデータベース（SwiftData）から保存済みのメモ一覧が自動で読み込まれ、作成日時の新しい順にリスト表示されます。右上の「＋」ボタンを押すと新しいメモが追加され、リストの行を左スワイプすることで簡単に削除できます。
各メモをタップすると詳細編集画面（MemoEditView）へ遷移し、タイトルや内容、お気に入りフラグを編集した瞬間にデータベースへ自動保存されます。
また、左上の「設定（歯車）」ボタンを押すとシートが開き、ユーザー名や表示順の設定を変更できます。ここで変更した設定は AppStorage によって即座に端末へ記憶され、アプリを完全に終了して再起動しても、ユーザーの名前や設定がそのまま維持されます。

コードの詳細解説
1. SwiftDataモデル（@Model）

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

何をしているか：
Swiftの標準的なクラス（Class）の先頭に @Model マクロを記述することで、このクラスがデータベース（SwiftData）にそのまま保存・管理可能な「データモデル」であることを宣言しています。

なぜこう書くのか：
@Model を付与するだけで、背後にあるSQLデータベースのテーブル構造の設定や、データ変更の監視に必要な複雑なコードをSwiftUIがすべて自動生成してくれるためです。開発者はデータ構造のデザインだけに集中できます。

もしこう書かなかったら：
@Model を書き忘れると、ただのメモリ上の通常クラスとして扱われてしまうため、後述するデータベース抽出用のプロパティラッパー（@Query）の対象に指定できず、コンパイルエラー（ビルド失敗）となります。

2. データの追加・削除（modelContext）
@Environment(\.modelContext) private var modelContext

// 追加
private func addMemo() {
    let newMemo = Memo(title: "新規メモ", content: "")
    modelContext.insert(newMemo)
}

// 削除
private func deleteMemos(offsets: IndexSet) {
    for index in offsets {
        modelContext.delete(displayedMemos[index])
    }
}

3. @Query によるデータ取得
   @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
   何をしているか：
@Query プロパティラッパーを使用することで、アプリ起動時やデータ更新時にデータベースの中身を自動スキャンし、作成日時（createdAt）の新しい順（.reverse）にソートした最新の配列をリアルタイムに取得しています。

なぜこう書くのか：
@Query はデータベースと画面（UI）を直結する「のぞき窓」として機能します。データが1件でも追加・変更されると、それを全自動で感知して SwiftUI の画面を再描画してくれるため、開発者が「再読み込みボタンの処理」を手動で実装する手間を一切省くことができるからです。

もしこう書かなかったら：
画面が表示されるタイミング（.onAppear など）で毎回手動でデータベースを検索・抽出する複雑な fetch 処理コードをガリガリ書く必要があり、さらにデータが書き換わった後に自動でリストが同期されないという致命的なバグの原因になります。

4. @AppStorage による設定保存
   @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
@AppStorage("userName") private var userName: String = ""

何をしているか：
ユーザー設定（名前や並び替えの好み）を、iOS標準の軽量な設定保存領域（UserDefaults）に、指定した「キー文字列」で自動同期・永続化する @AppStorage プロパティラッパーを配置しています。

なぜこう書くのか：
「お気に入りを上に表示するかどうか」といった、アプリ全体の挙動を決める少量のフラグや文字列データを、わざわざ重いデータベース（SwiftData）のテーブルに組み込むのは非効率だからです。「机の上のメモ帳」感覚で手軽に値を出し入れできる専用の仕組みとして使い分けるのがモダン設計の鉄則です。

もしこう書かなかったら：
通常の @State 変数として宣言した場合、ユーザーがどんなに名前を入力したり並び替えスイッチを切り替えたりしても、アプリを完全に終了させる（マルチタスクから消す）たびに、設定がすべて初期値（空文字やfalse）にリセットされてしまいます。

5. @Bindable による編集データの双方向連動
   struct MemoEditView: View {
    @Bindable var memo: Memo
    // ...
    TextField("タイトル", text: $memo.title)
}

何をしているか：
詳細編集画面において、親ビューから渡された Memo オブジェクトの各プロパティを TextField や Toggle で直接書き換えられるようにするため、@Bindable プロパティラッパーを使って「双方向バインディング（$）」のルートを開通させています。

なぜこう書くのか：
SwiftDataのモデル（@Model）は「クラス（参照型データ）」です。SwiftUIの入力系コンポーネント（TextFieldなど）でクラスのプロパティを直接バインドして書き換えるためには、構造体（値型）で使っていた @Binding ではなく、クラス専用の @Bindable を用いるルールになっているためです。

もしこう書かなかったら：
$memo.title のように $ をつけてテキストフィールドに値を渡すことができずコンパイルエラーになるか、あるいは一度編集画面側で一時的な変数にコピーし、完了ボタンが押されたときに元のクラスへ書き戻すといった、二度手間かつバグを誘発しやすい古い設計のコードを書く羽目になります。

新しく学んだSwiftの文法・API

項目,説明,使用例
@Model,Swiftの通常のクラスを、SwiftDataで管理・保存可能なデータベースモデルに変換する強力なマクロ。,@Model class Memo { ... }
@Query,データベース内を自動検索し、条件（ソート等）に合うデータをリアルタイムに取得してUIへ同期するプロパティラッパー。,@Query(sort: \.createdAt) var memos
modelContext,データベースに対するデータの追加（insert）や削除（delete）といった実際の操作指示を出すための操作コンテキスト。,modelContext.insert(newMemo)
@AppStorage,アプリ設定などの軽量なデータを、UserDefaults（端末の共有設定領域）へキー指定で手軽に永続化する仕組み。,"@AppStorage(""userName"") var name = """""
@Bindable,クラス型（参照型）であるSwiftDataのモデルオブジェクトに対して、SwiftUIビューとの双方向バインディング（$）を可能にするプロパティラッパー。,@Bindable var memo: Memo
.modelContainer,アプリ全体、または特定のビュー階層に対して、どのデータモデルを扱うかを設定・注入するビューモディファイア。,.modelContainer(for: Memo.self)

自分の実験メモ
🧪 実験1：@Query のソート条件を複数組み合わせてみた
やったこと： 計算プロパティ displayedMemos 内でソートするのではなく、@Query の引数部分を配列形式にし、@Query(sort: [SortDescriptor(\Memo.isFavorite, order: .reverse), SortDescriptor(\Memo.createdAt, order: .reverse)]) のように書き換えてみた。

結果： 設定画面のオンオフに関わらず、データベースから取得された時点で自動的に「まずお気に入りが上に来て、その中で新しい順に並ぶ」という完璧な初期ソート状態が実現できた。

わかったこと： SwiftDataの @Query は非常に高度な並び替え（SortDescriptor）を標準サポートしており、極力データベースから取り出す段階で条件を絞り込んでおいた方が、SwiftUI側のコードがスマートかつ高速になると分かった。

🧪 実験2：@AppStorage の初期値（デフォルト値）の挙動を確認した
やったこと： @AppStorage("userName") private var userName: String = "ゲストユーザー" のように、初期値を空文字（""）から具体的な文字列に変えてアプリの挙動を確認した。

結果： アプリを完全な初回起動（またはアンインストール後再起動）した際、ナビゲーションタイトルが自動的に「ゲストユーザーのメモ」と表示され、その後設定画面で自分の名前に変更すると、次回以降はその新しい名前がバッチリ上書き保存された。

わかったこと： @AppStorage の右辺で指定する初期値は、端末内にその「キー（"userName"）」のデータがまだ一度も保存されていない場合にのみ発動する、安全なデフォルト値（防衛策）として機能している仕組みがよく理解できた。

AIに聞いて特に理解が深まった質問 TOP3
質問： AppStorage と SwiftData はどちらもデータを保存（永続化）するために使われていますが、なぜこの2つを使い分ける必要があるのですか？初心者向けに教えてください。
得られた理解： 保存したいデータの「種類」と「量」によって明確に使い分けるべきだと納得しました。ユーザーの名前や並び替え設定のような少量のデータは「机の上のメモ帳」である AppStorage で手軽に扱い、メモの内容など大量の構造化データは「本格的な档案キャビネット」である SwiftData（データベース）に任せる。この分担により、アプリのパフォーマンスが最適化されコードもスッキリします。

質問： データを画面に出すときは @Query を使い、追加や削除のときは @Environment(\.modelContext) を使っています。この2つの役割の違いと関係性がよくわかりません。
得られた理解： この2つは完全に役割分担された「完璧な連携ペア」なのだと分かりました。@Query はデータベースを常に監視してリアルタイムに画面に映し出す「のぞき窓（モニター）」であり、modelContext はデータベースへ追加・削除の命令を送り込む「操作レバー」です。レバー（modelContext）でデータを書き換えると、窓（@Query）が自動でそれを検知して画面を書き換えてくれるため、開発者が同期コードを手書きする必要がありません。

質問： メモ編集画面（MemoEditView）のコードで @Bindable というキーワードが出てきましたが、これまで習った @State や @Binding とは何が違うのですか？
得られた理解： 扱うデータの正體が「構造体（値型）」か「クラス（参照型）」かによって、選ぶべきプロパティラッパーの道具が変わるのだと理解できました。@State / @Binding は構造体用ですが、SwiftDataのモデル（@Model）はクラス（Class）です。編集画面のテキストフィールド等でクラスの中身を直接ダイレクトに安全更新させるためには、クラス専用の @Bindable を使うのがSwiftUIの最新のお決まりパターン（公式ルート）なのだと納得しました。

この章のまとめ
SwiftUIにおける「データの永続化と状態管理の自動化」について、非常にモダンで強力なベストプラクティスを構築できました。

適材適所のデータ保存： AppStorage を使った手軽なユーザー設定の保存と、SwiftData を使った本格的なモデルデータの保存を組み合わせることで、実用的なアプリの土台となるデータ管理手法をマスターしました。

宣言型UIとデータベースの融合： @Query と modelContext の役割を明確に理解したことで、データの追加・削除がどのように画面へ自動反映されるのか、その裏側のサイクルが完全にクリアになりました。

モダンなデータ連携 of 編集： @Bindable を活用した参照型モデルの双方向バインディングを習得したことで、データの「一覧表示」「新規追加」「詳細編集」という、あらゆるアプリの基本となるCRUD（データ操作）のフローを美しく実装できる自信がつきました。
