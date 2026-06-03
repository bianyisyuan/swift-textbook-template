# 第5章：機能統合の実践

> 執筆者：卞宜璇
> 最終更新：2026-06-03

## この章で学ぶこと

この章では、これまでに独立して学んできた「カメラ・アルバムからの写真選択（PhotosUI）」「MapKitによる地図表示・位置情報取得（CoreLocation）」、そして「SwiftDataによるデータの永続化」という3つの重要機能を1つのアプリに融合させる **「機能統合（インテグレーション）」** の実践手法を学びます。

具体的には、撮影・選択した写真データ（バイナリ）をデータベースへ安全に保存するモデル設計、デバイスのGPSから現在地をリアルタイムに取得する非同期マネージャーの構築、そして `TabView` を用いて「マップ画面」と「一覧画面」でデータを相互に同期・共有する本格的なアプリ全体のアーキテクチャ設計をマスターします。

---

## 模範コードの全体像

```swift
// ============================================
// 第5章：カメラ + 地図 + データ保存の統合アプリ
// ============================================
// 写真を撮影し、撮影場所を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
//   - NSCameraUsageDescription（実機の場合）
// ============================================

import SwiftUI
import SwiftData
import MapKit
import PhotosUI

// MARK: - データモデル

@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date

    init(title: String, memo: String = "", latitude: Double, longitude: Double, imageData: Data? = nil) {
        self.title = title
        self.memo = memo
        self.latitude = latitude
        self.longitude = longitude
        self.imageData = imageData
        self.createdAt = .now
    }

    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }

    var uiImage: UIImage? {
        guard let data = imageData else { return nil }
        return UIImage(data: data)
    }
}

// MARK: - 位置情報マネージャー

@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        currentLocation = locations.last?.coordinate
    }
}

// MARK: - アプリエントリポイント
// ※ App ファイルに以下を記述：
//
// @main
// struct PhotoMapApp: App {
//     var body: some Scene {
//         WindowGroup {
//             ContentView()
//         }
//         .modelContainer(for: PhotoRecord.self)
//     }
// }

// MARK: - メインビュー（タブ構成）

struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem {
                    Label("マップ", systemImage: "map")
                }

            ListTab()
                .tabItem {
                    Label("一覧", systemImage: "list.bullet")
                }
        }
    }
}

// MARK: - マップタブ

struct MapTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query private var records: [PhotoRecord]
    @State private var locationManager = LocationManager()
    @State private var cameraPosition: MapCameraPosition = .automatic
    @State private var isShowingAddSheet = false
    @State private var selectedRecord: PhotoRecord?

    var body: some View {
        NavigationStack {
            ZStack(alignment: .bottomTrailing) {
                Map(position: $cameraPosition) {
                    UserAnnotation()

                    ForEach(records) { record in
                        Annotation(record.title, coordinate: record.coordinate) {
                            Button {
                                selectedRecord = record
                            } label: {
                                if let uiImage = record.uiImage {
                                    Image(uiImage: uiImage)
                                        .resizable()
                                        .aspectRatio(contentMode: .fill)
                                        .frame(width: 40, height: 40)
                                        .clipShape(Circle())
                                        .overlay(Circle().stroke(.white, lineWidth: 2))
                                        .shadow(radius: 2)
                                } else {
                                    Image(systemName: "photo.circle.fill")
                                        .font(.title)
                                        .foregroundStyle(.blue)
                                }
                            }
                        }
                    }
                }
                .mapControls {
                    MapUserLocationButton()
                }

                // 追加ボタン
                Button {
                    isShowingAddSheet = true
                } label: {
                    Image(systemName: "plus.circle.fill")
                        .font(.system(size: 56))
                        .foregroundStyle(.blue)
                        .background(Circle().fill(.white))
                        .shadow(radius: 4)
                }
                .padding(24)
            }
            .navigationTitle("フォトマップ")
            .sheet(isPresented: $isShowingAddSheet) {
                AddRecordView(locationManager: locationManager)
            }
            .sheet(item: $selectedRecord) { record in
                RecordDetailView(record: record)
            }
        }
    }
}

// MARK: - 一覧タブ

struct ListTab: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \PhotoRecord.createdAt, order: .reverse) private var records: [PhotoRecord]

    var body: some View {
        NavigationStack {
            List {
                ForEach(records) { record in
                    HStack(spacing: 12) {
                        if let uiImage = record.uiImage {
                            Image(uiImage: uiImage)
                                .resizable()
                                .aspectRatio(contentMode: .fill)
                                .frame(width: 50, height: 50)
                                .clipShape(RoundedRectangle(cornerRadius: 8))
                        }

                        VStack(alignment: .leading, spacing: 4) {
                            Text(record.title)
                                .font(.headline)
                            Text(record.createdAt, style: .date)
                                .font(.caption)
                                .foregroundStyle(.secondary)
                        }
                    }
                }
                .onDelete { offsets in
                    for index in offsets {
                        modelContext.delete(records[index])
                    }
                }
            }
            .navigationTitle("記録一覧")
        }
    }
}

// MARK: - 記録追加画面

struct AddRecordView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    let locationManager: LocationManager

    @State private var title = ""
    @State private var memo = ""
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImageData: Data?
    @State private var previewImage: Image?

    var body: some View {
        NavigationStack {
            Form {
                Section("写真") {
                    if let image = previewImage {
                        image
                            .resizable()
                            .aspectRatio(contentMode: .fit)
                            .frame(maxHeight: 200)
                            .clipShape(RoundedRectangle(cornerRadius: 8))
                    }

                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("写真を選択", systemImage: "photo")
                    }
                }

                Section("情報") {
                    TextField("タイトル", text: $title)
                    TextField("メモ（任意）", text: $memo, axis: .vertical)
                        .lineLimit(3...6)
                }

                Section("位置情報") {
                    if let location = locationManager.currentLocation {
                        Text("緯度: \(location.latitude, specifier: "%.4f")")
                        Text("経度: \(location.longitude, specifier: "%.4f")")
                    } else {
                        Text("位置情報を取得中...")
                            .foregroundStyle(.secondary)
                    }
                }
            }
            .navigationTitle("新しい記録")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        saveRecord()
                    }
                    .disabled(title.isEmpty || locationManager.currentLocation == nil)
                }
            }
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    if let data = try? await newItem?.loadTransferable(type: Data.self) {
                        selectedImageData = data
                        if let uiImage = UIImage(data: data) {
                            previewImage = Image(uiImage: uiImage)
                        }
                    }
                }
            }
        }
    }

    func saveRecord() {
        guard let location = locationManager.currentLocation else { return }

        let record = PhotoRecord(
            title: title,
            memo: memo,
            latitude: location.latitude,
            longitude: location.longitude,
            imageData: selectedImageData
        )
        modelContext.insert(record)
        dismiss()
    }
}

// MARK: - 記録詳細画面

struct RecordDetailView: View {
    let record: PhotoRecord

    var body: some View {
        ScrollView {
            VStack(spacing: 16) {
                if let uiImage = record.uiImage {
                    Image(uiImage: uiImage)
                        .resizable()
                        .aspectRatio(contentMode: .fit)
                        .clipShape(RoundedRectangle(cornerRadius: 12))
                }

                VStack(alignment: .leading, spacing: 8) {
                    Text(record.title)
                        .font(.title2)
                        .bold()

                    if !record.memo.isEmpty {
                        Text(record.memo)
                            .foregroundStyle(.secondary)
                    }

                    Text(record.createdAt, style: .date)
                        .font(.caption)
                        .foregroundStyle(.tertiary)
                }
                .frame(maxWidth: .infinity, alignment: .leading)

                // ミニマップ
                Map {
                    Marker(record.title, coordinate: record.coordinate)
                }
                .frame(height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 12))
            }
            .padding()
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: PhotoRecord.self, inMemory: true)
}
```

📌 **このアプリは何をするものか：**

ユーザーが起動すると、画面下部のタブで「マップ」と「記録一覧」を自由に切り替えられます。「マップ」タブでは、端末の現在地周辺の地図が広がり、これまでに記録した思い出の写真がその撮影場所（ピンの代わりの丸型アイコン）に自動展開されます。
右下の青い「＋」ボタンを押すと、新しい記録を追加するシートが開きます。ここでは、アルバムから写真（PhotosPicker）を選択すると同時に、内部のGPSマネージャーが現在地の正確な緯度・経度を自動取得します。タイトルを入力して保存すると、データと写真がまとめて SwiftData に永続化されます。
保存されたデータは「一覧」タブにも即座にリアルタイム同期され、リストからスワイプで簡単に削除可能です。また、マップ上の写真アイコンをタップすると、その場所の詳細（写真、メモ、周辺のミニマップ）が美しいハーフシートで表示される、本格的な位置情報連動型アルバムアプリです。

---

## コードの詳細解説

### データモデルの設計

```swift
@Model
class PhotoRecord {
    var title: String
    var memo: String
    var latitude: Double
    var longitude: Double
    var imageData: Data?
    var createdAt: Date
    // ...
    var coordinate: CLLocationCoordinate2D {
        CLLocationCoordinate2D(latitude: latitude, longitude: longitude)
    }
}
```

* **何をしているか：** SwiftDataのモデル `@Model` クラス内に、テキスト情報のほかに位置情報を示す緯度・経度（ `Double` ）、および画像用のバイナリデータ（ `Data?` ）をまとめて定義しています。さらに、MapKitでそのままピンの配置に使えるように、緯度・経度を合体させた `CLLocationCoordinate2D` を返す便利な「計算プロパティ（ `coordinate` ）」をモデル内に内蔵させています。
* **なぜこう書くのか：** MapKitのAPIは位置情報を指定する際、バラバラの緯度・経度ではなく `CLLocationCoordinate2D` という特殊な構造体を要求します。しかし、この構造体はSwiftDataのデータベースにそのまま直接保存することができないデータ型です。そのため、データベースには基本型である `Double` で安全に2つの数値として保存し、コード内で使うときだけ計算プロパティで構造体に自動変換して取り出すのが最もクリーンでエラーの出ない設計になるからです。
* **もしこう書かなかったら：** モデルのプロパティに直接 `var coordinate: CLLocationCoordinate2D` を記述しようとすると、SwiftDataが「この型はデータベースのテーブルに変換できません」というコンパイルエラーを吐き、ビルドすら通らなくなってしまいます。

---

### タブ構成の設計

```swift
struct ContentView: View {
    var body: some View {
        TabView {
            MapTab()
                .tabItem { Label("マップ", systemImage: "map") }

            ListTab()
                .tabItem { Label("一覧", systemImage: "list.bullet") }
        }
    }
}
```

* **何をしているか：** アプリのルートビュー（ `ContentView` ）に `TabView` を配置し、その中に地図を表示するビュー（ `MapTab` ）とリストを表示するビュー（ `ListTab` ）を並列に並べています。それぞれのビューに対して `.tabItem` モディファイアを使って、アイコンとラベルを付与してタブバーを構成しています。
* **なぜこう書くのか：** 同一のSwiftDataデータベース（コンテナ）を参照する複数のビューを `TabView` 内に同居させることで、データの共通化を美しく達成するためです。 `MapTab` で新しくデータを1件追加すると、開発者が画面の更新処理を一切書かなくても、隣の `ListTab` のリストにもそのデータが「全自動で、一瞬で」反映される仕様をSwiftUIの標準機能だけで実現できます。
* **もしこう書かなかったら：** 画面遷移をすべて `NavigationLink` による階層移動だけで作ろうとした場合、マップと一覧という全く異なる視点の画面をユーザーが行ったり来たりする操作性が著しく低下します。また、画面ごとにデータを別々に管理してしまうと、片方で追加したのに、もう片方の画面に戻ってもデータが増えていないというデータの不整合（同期バグ）が発生しやすくなります。

---

### カメラと位置情報の連携

```swift
@Observable
class LocationManager: NSObject, CLLocationManagerDelegate {
    private let manager = CLLocationManager()
    var currentLocation: CLLocationCoordinate2D?

    override init() {
        super.init()
        manager.delegate = self
        manager.desiredAccuracy = kCLLocationAccuracyBest
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }
    // ...
}
```

* **何をしているか：** iOSの標準位置情報システム（CoreLocation）を司る `CLLocationManager` をラップした、独自の `LocationManager` クラスを作成しています。 `@Observable` マクロを付与し、初期化時にユーザーへの位置情報利用許可の申請（ `requestWhenInUseAuthorization` ）と、GPSによる位置追跡の開始（ `startUpdatingLocation` ）を自動でトリガーし、最新の座標を `currentLocation` にリアルタイム保存しています。
* **なぜこう書くのか：** 位置情報の取得は「いつGPSが座標を拾えるか分からない」という典型的な非同期の外部イベントです。そのため、ビューの内部に直接その処理を書くのは不可能です。UIKitの歴史的なデリゲート（委託）通知を受け取るために `NSObject` と `CLLocationManagerDelegate` を継承した専用クラスを外側に1本立て、そこで座標をキャッチしてビューへ安全に受け渡すのがiOS開発の絶対的な王道パターンだからです。
* **もしこう書かなかったら：** 位置情報の取得コードを書かなかったり、 `Info.plist` への権限記述（ `NSLocationWhenInUseUsageDescription` ）を忘れたりした場合、アプリを起動して追加画面を開いても「位置情報を取得中...」のまま画面が一切進まなくなり、保存ボタンもグレーアウトしたまま（ `.disabled` ）押せなくなってしまいます。

---

### SwiftDataでの画像保存

```swift
// モデル側
var uiImage: UIImage? {
    guard let data = imageData else { return nil }
    return UIImage(data: data)
}

// 追加画面側（データ変換・保存）
if let data = try? await newItem?.loadTransferable(type: Data.self) {
    selectedImageData = data
}
```

* **何をしているか：** アルバム（ `PhotosPicker` ）から選ばれた写真オブジェクト（ `PhotosPickerItem` ）から、非同期処理（ `loadTransferable` ）を使って生のバイナリデータ（ `Data` ）を抽出し、それをそのままSwiftDataのモデルに渡しています。表示する際は、モデル側に定義した計算プロパティ `uiImage` が、バイナリを自動的に `UIImage`（SwiftUIで使える形）に復元して画面に出力しています。
* **なぜこう書くのか：** 前述の通り、データベースには `UIImage` などのリッチな画像オブジェクトを直接叩き込むことができない仕様だからです。データを限界までシンプルな「1と0の並び（ `Data` 型）」にシリアライズ（直列化）して保存し、画面に描画する瞬間だけデシリアライズして画像に戻す、というデータ変換のパイプラインを組むのがルールだからです。
* **もしこう書かなかったら：** データの変換処理を挟まずに強引に画像を保存しようとするとビルドエラーになります。また、巨大な高画質画像をそのまま大量にデータベースに詰め込みすぎると、将来的にデータベース全体の動作が極端に重くなり、アプリの起動やスクロールがガクガクになる原因になります（実務では `Data` 型にする際、適切なサイズに圧縮するなどのケアが重要になります）。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
| :--- | :--- | :--- |
| **`TabView`** | 画面下部に配置され、複数の独立したビューをタップで切り替える標準のタブナビゲーション。 | `TabView { MapTab().tabItem { ... } }` |
| **`CLLocationManager`** | デバイスのGPSなどのハードウェアにアクセスし、現在の位置情報や移動を追跡するCoreLocationの中核クラス。 | `let manager = CLLocationManager()` |
| **`@Observable`** | iOS 17以降の最新の観察マクロ。クラスのプロパティの変更をSwiftUIが全自動で検知し、ビューを部分最適化して再描画する。 | `@Observable class LocationManager { ... }` |
| **`Map`** | SwiftUIに標準組み込みされたMapKitのビュー。ピン（Marker）やカスタムUI（Annotation）を簡単に配置できる。 | `Map(position: $cameraPosition) { ... }` |
| **`loadTransferable`** | `PhotosPickerItem` などから、安全かつ非同期に背後の実データ（画像データなど）を取り出すための最新のAPI。 | `try await item.loadTransferable(type: Data.self)` |
| **`axis: .vertical`** | `TextField` に指定することで、入力文字数に応じてテキストフィールドが縦方向に自動で伸びるようにするモディファイア。 | `TextField("メモ", text: $text, axis: .vertical)` |

---

## 自分の実験メモ

### 🧪 実験1：ミニマップの表示を「航空写真」や「3D」に切り替えてみた
* **やったこと：** 詳細画面（ `RecordDetailView` ）のミニマップ部分に、新しいMapKitのオプションである `.mapStyle(.imaging)` や `.mapStyle(.hybrid(elevation: .realistic))` を付与してみた。
* **結果：** 通常の平面の地図から、Google Earthのようなリアルな衛星写真と地形が立体的に浮き出るマップへと一瞬で表示が切り替わった。
* **わかったこと：** 新しいSwiftUIの `Map` コンポーネントは、モディファイアを1行足すだけで高度な地図の表示スタイル変更をサポートしており、特別なカスタマイズコードをUIKitから移植しなくてもリッチな表現ができると感動した。

### 🧪 実験2：位置情報が未許可（シミュレータ等の初期状態）の挙動を検証した
* **やったこと：** Macのシミュレータの「Features > Location」を「None」にする、またはアプリの位置情報許可を端末の設定から「拒否」にして挙動を見た。
* **結果：** 追加画面の緯度・経度の欄に「位置情報を取得中...」というグレーの文字が正しく表示され、さらに保存ボタンが完全にロック（無効化）されて安全にガードされた。
* **わかったこと：** 模範コードの `Button(...).disabled(title.isEmpty || locationManager.currentLocation == nil)` という条件式が、現在地が取れていない状態での不正なデータ保存（バグデータの発生）を防ぐ強力なバリデーション（防衛策）として完璧に機能していることが実証できた。

---

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `Map` の中で使う `Marker` と `Annotation` は、どちらも地図に印をつけるものですが、何が違うのですか？どう使い分ければいいですか？
   * **得られた理解：** デザインの自由度が全く違うと分かりました。 `Marker` は、地図におなじみの「バルーン（風船型）のピン」を素早く立てる標準の簡易表示。対して `Annotation` は、内部に任意のSwiftUIビュー（今回のように、丸く切り抜いた思い出の写真そのものなど）をそのままはめ込むことができるカスタム表示です。今回は「写真が地図に並ぶ」のがアプリのコアなので、 `Annotation` を選ぶのが大正解なのだと納得しました。

2. **質問：** `Info.plist` に記述する `NSLocationWhenInUseUsageDescription` などのプライバシーキーを書き忘れると、アプリはどうなりますか？
   * **得られた理解：** Appleの強力なセキュリティ機構により、アプリが位置情報にアクセスしようとした瞬間に「一発で強制終了（クラッシュ）」するか、一切の通信がOSによって遮断されます。ユーザーのプライバシー（GPSやカメラ、アルバム）に触れる機能を使う際は、コードが正しくてもこの設定ファイル（許諾メッセージの登録）が抜けていれば100%動かないという、iOSアプリ特有の厳格な開発ルールを深く刻み込むことができました。

3. **質問：** `@Query private var records: [PhotoRecord]` のように、第5章のマップタブではソート条件（ `sort:` ）が指定されていません。なぜ指定しなくても問題ないのですか？
   * **得られた理解：** 地図（ `Map` ）にピンを配置する際は、データが「どの順番で並んでいるか」は一切関係なく、単に地球上の座標（緯度・経度）に配置されれば良いため、ソートは不要だからです。逆に、時系列に上から並べる「一覧（ `ListTab` ）」では、最新順に見せる必要があるので `(sort: \PhotoRecord.createdAt, order: .reverse)` が必須になります。このように、データを「どう見せるか」によってクエリの最適化を使い分けることの重要性が理解できました。

---

## この章のまとめ

これまでに学んだ単体機能（カメラ、マップ、永続化）を組み合わせることで、ついに「実際にApp Storeに並んでいてもおかしくないレベルの本格的な成果物」を自らの手で組み上げることができました。

* **アーキテクチャの視点：** ハードウェアを制御する `LocationManager` （モデル層）、データを永続化する `PhotoRecord` （データ層）、そしてそれらを統合してタブで切り替える `ContentView` （ビュー層）という、綺麗なM-V-VMに近い役割分担の重要性が身に染みて理解できました。
* **データ変換の壁を突破：** `PhotosPicker` から得られた画像を `Data` に落とし込んで SwiftData に保存し、それを `UIImage` に戻して `Annotation` として地図に描画するという一連のデータフローを習得したことで、今後のあらゆるメディア系アプリ開発に応用できる強固な自信が手に入りました。
