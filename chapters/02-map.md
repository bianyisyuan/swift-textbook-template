# 第2章：地図アプリの基本

> 執筆者：卞宜璇
> 最終更新：2026-05-22

## この章で学ぶこと

この章では、iOS 17で刷新された **MapKit** の基本的な使い方と、SwiftUIでデータを効率よく扱うための **プロトコル（Identifiable / CaseIterable）** や **状態管理（@State）** を用いたデータ連携の仕組みについて学びます。

具体的には、ランドマークデータを構造体で定義し、地図上に動的にマーカーを表示させ、カテゴリでのフィルタリングやタップ連動による詳細カード表示を実装するアプリを題材にします。

---

## 模範コードの全体像

```swift
import SwiftUI
import MapKit

// 💡 ここに学校や教材で配布された、第2章の完全なソースコード（ContentView.swiftなど）を貼り付けてください。
```

**📌 このアプリは何をするものか：**

ユーザーが選択したカテゴリ（寺社、タワー、公園など）に応じて、地図上のマーカーが動的にフィルタリングされるアプリです。
地図上のマーカーをタップすると、その場所のIDが特定され、画面下部に詳細情報カード（`LandmarkCard`）が連動して表示されます。また、ボタン操作によって、特定の場所にスムーズなアニメーション付きでカメラを移動させる（ジャンプする）ことも可能です。

---

## コードの詳細解説

### 1. データモデル（ランドマーク構造体）

```swift
struct Landmark: Identifiable {
    let id = UUID()
    let name: String
    let coordinate: CLLocationCoordinate2D
}

enum Category: String, CaseIterable {
    case shrine = "寺社"
    case tower = "タワー"
    case park = "公園"
}
```

* **何をしているか：**
  `Identifiable` プロトコルを採用したランドマークのデータ構造と、`CaseIterable` プロトコルを採用したカテゴリの列挙型（`enum`）を定義しています。
* **なぜこう書くのか：**
  * **`Identifiable`:** データに一意の「背番号（`id`）」を持たせることで、SwiftUIの `ForEach` や地図のマーカー表示において、同名の場所があっても正確に区別・識別できるようにするためです。
  * **`CaseIterable`:** `Category.allCases` を使うことで、すべてのカテゴリを配列として一括取得できるようになります。これにより、UIのフィルターチップ（ボタン）を1つずつ手書きせず、`ForEach` で自動生成できるようになりコードがスマートになります。
* **もしこう書かなかったら：**
  * `Identifiable` がないと、どのピンがどのデータに対応しているのかSwiftUIが特定できず、描画エラーや予期せぬバグの原因になります。
  * `CaseIterable` がないと、カテゴリが増減するたびにUI側のボタンのコードを手動で書き直さなければならず、コードが肥大化し修正漏れが起きやすくなります。

---

### 2. 地図の表示とカメラ制御

```swift
@State private var cameraPosition: MapCameraPosition = .automatic

Map(position: $cameraPosition, selection: $selectedLandmarkID) {
    // マーカーの配置など
}

Button("東京タワーへジャンプ") {
    withAnimation {
        cameraPosition = .region(
            MKCoordinateRegion(
                center: CLLocationCoordinate2D(latitude: 35.6586, longitude: 139.7454),
                span: MKCoordinateSpan(latitude: 0.01, longitude: 0.01)
            )
        )
    }
}
```

* **何をしているか：**
  iOS 17の新しい `Map` ビューを使い、`MapCameraPosition` 構造体によって地図の表示領域（カメラ位置）を動的に制御しています。また、ボタンをタップした際に特定の座標（例：東京タワー）へカメラをアニメーション移動させています。
* **なぜこう書くのか：**
  iOS 17以降の MapKit では、地図の中に要素を直感的に並べる方式に進化しました。カメラ位置は `@State` で定義した `cameraPosition` を書き換えるだけで制御でき、さらに `withAnimation` ブロックの中で更新するだけで、標準のマップアプリのような滑らかでスムーズな移動エフェクト（カメラワーク）を自動で適用できるためです。
* **もしこう書かなかったら：**
  以前の方式（`coordinateRegion` をバインドする古い書き方）では記述が煩雑になり、地図内の要素配置との連動が難しくなります。また、`withAnimation` を使わない場合、カメラが滑らかに動かず、一瞬でパッと画面が切り替わるため、ユーザーが場所を見失う不親切なUIになってしまいます。

---

### 3. マーカーの表示

```swift
Map(position: $cameraPosition, selection: $selectedLandmarkID) {
    ForEach(landmarks) { landmark in
        Marker(landmark.name, coordinate: landmark.coordinate)
            .tag(landmark.id)
    }
}
```

* **何をしているか：**
  地図上の指定された座標にシステム標準のピン（`Marker`）を配置し、各マーカーに `.tag()` モディファイアを使ってランドマークの一意のIDを紐付けています。
* **なぜこう書くのか：**
  `Map` の引数にある selection（`$selectedLandmarkID`）と各マーカーの `.tag(landmark.id)` を連動させるためです。こうすることで、ユーザーが地図上のピンをタップした瞬間に、その tag の値が自動的にバインド先の変数に代入され、「どの要素がタップされたか」を瞬時に特定できるようになります。
* **もしこう書かなかったら：**
  `.tag()` を設定しないと、ピンをタップしたこと自体は検知できても「どのピンがタップされたのか」の情報を取得できません。そのため、画面下部にタップされた場所の詳細カード（`LandmarkCard`）を連動して表示させることが不可能になります。

---

### 4. フィルター機能

```swift
@State private var selectedCategories: Set<Category> = []

var filteredLandmarks: [Landmark] {
    if selectedCategories.isEmpty {
        return landmarks
    }
    return landmarks.filter { selectedCategories.contains($0.category) }
}
```

* **何をしているか：**
  ユーザーが選択しているカテゴリを状態変数（`@State`）で監視し、その値に基づいて表示すべきデータを動的に絞り込む「計算プロパティ（`filteredLandmarks`）」を作成しています。
* **なぜこう書くのか：**
  `@State`（センサーの役割）が更新されるたびに、SwiftUIが自動的に計算プロパティ（加工機）を再計算して画面を書き直すため、常に最新のフィルター結果が地図に反映されます。フィルタリングの複雑なロジックをUI（`body`）の中に書かずに分離できるため、コードの見通しが非常に良くなります。
* **もしこう書かなかったら：**
  フィルターをタップして変数の中身が変わっても、`@State` を使っていなければUIが更新されず、地図上のマーカーは古いままになってしまいます。また、計算プロパティを使わずに処理しようとすると、`body` の中に複雑な条件分岐やループ処理が入り込み、「ネスト地獄」を招いてコードの可読性が著しく低下します。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
| :--- | :--- | :--- |
| **`Identifiable`** | データの「背番号（一意のID）」を保証し、特定の要素を識別可能にするプロトコル。 | `struct Landmark: Identifiable { let id = UUID() }` |
| **`CaseIterable`** | 列挙型（enum）のすべてのケースを配列として一覧化（.allCases）できるようにするプロトコル。 | `ForEach(Category.allCases, id: \.self)` |
| **`@State`** | データの変化を監視する「センサー」。値が変わるとUIの自動再描画をトリガーする。 | `@State private var selectedLandmarkID: UUID?` |
| **計算プロパティ** | 値を保持せず、呼ばれるたびに処理を実行して結果を返す「加工機」のようなプロパティ。 | `var filteredLandmarks: [Landmark] { ... }` |
| **`Map (iOS 17)`** | 宣言的な「キャンバス」。中に Marker や Annotation を直感的に配置できるビュー。 | `Map(position: $cameraPosition) { ... }` |
| **`MapCameraPosition`** | 地図の表示領域（.automatic や .region など）を制御するための構造体。 | `cameraPosition = .region(MKCoordinateRegion(...))` |
| **`Marker`** | 地図上に配置するシステム標準の「ピン」を生成するコンポーネント。 | `Marker(landmark.name, coordinate: coordinate)` |

---

## 自分の実験メモ

### 🧪 実験1：マーカーの代わりに Annotation を使ってみた

* **やったこと：** `Marker` の記述部分を `Annotation` に書き換え、カスタムビューを配置した。
* **結果：** シンプルなピンの形ではなく、丸型アイコンやテキスト、カスタム画像（`Image`）を地図上に自由なレイアウトで表示できた。
* **わかったこと：** 標準的な見た目で素早く実装したい時は `Marker`、独自のデザインや複雑な情報を地図上に表現したい時は `Annotation` を使うという使い分けの基準が理解できた。

### 🧪 実験2：選択されたランドマークIDを保持する変数を「オプショナル型（UUID?）」にしてみた

* **やったこと：** `@State private var selectedLandmarkID: UUID? = nil` として定義した。
* **結果：** アプリ起動直後や、地図の何もない場所をタップした際に正常に `nil`（未選択状態）を保持できた。
* **わかったこと：** `if let` 構文と組み合わせることで、「データがある時（非nil）だけカードを表示する」というスマートなUI条件分岐が作れると分かった。

---

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：iOS 17以降での Map の新しい書き方と、Marker の配置方法のメリットは何ですか？**
   > **得られた理解：** 以前の「表示範囲をバインドする方式」から「地図の中に要素を並べる方式」に進化したことで、まるで `VStack` や `List` のように直感的にピンを配置できる柔軟性を獲得したことが分かりました。

2. **質問：地図上のマーカーをタップした時に、画面下部のカードと連動させる仕組み（tag の役割）はどうなっていますか？**
   > **得られた理解：** 各マーカーに `.tag()` で一意のIDを紐付け、それを `Map` の `selection` にバインドすることで、タップ時に自動でIDが代入される仕組みを学びました。未選択状態を表現するために「オプショナル型（?）」を使う重要性も理解できました。

3. **質問：ユーザーの操作に応じて、地図のカメラを特定の場所に移動させたり全体を表示したりするにはどうすればいいですか？**
   > **得られた理解：** `MapCameraPosition` の `.automatic` や `.region` などの設定を使い、さらに `withAnimation` ブロック内で変数を書き換えるだけで、標準マップのような滑らかな移動エフェクトが簡単に実装できることを学びました。

---

## この章のまとめ

SwiftUIの開発における「データがUIを動かす（Data-Driven UI）」という本質への理解が一段と深まりました。

`Identifiable` や `CaseIterable` といったプロトコルは、単なるコードの決まり事ではなく、`ForEach` などのSwiftUIの強力な機能と地図（MapKit）を安全に連携させるための重要な「鍵」です。

iOS 17の新しいMapKitは、これまでのリスト表示（`List`）と全く同じ感覚で実装できる一貫性があり、マーカーの識別（`tag`）からオプショナル型を用いた詳細カードの出し分け（`if let`）に繋げる流れは、モダンなアプリ開発における王道の設計パターン（ベストプラティス）であると確信しました。未来の自分も、データ連携に迷ったらこの設計に立ち戻るようにしてください。
