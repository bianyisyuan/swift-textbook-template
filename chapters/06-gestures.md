# 第6章：ジェスチャー操作

> 執筆者：卞宜璇
> 最終更新：2026-06-03

## この章で学ぶこと

この章では、スマートフォンのUIにおける最も直感的で強力な操作である **「ジェスチャー認識（Gesture Recognition）」** と、それに連動する **「インタラクティブ・アニメーション」** の実装方法を学びます。

具体的には、ユーザーの指の移動距離をリアルタイムに追従する `DragGesture` の使い方、移動距離に応じたカードの動的な「傾き（回転効果）」の計算、そして世界的に有名なマッチングアプリ「Tinder」のような、カードを左右にフリックして「LIKE」と「NOPE」に仕分ける高度なスタックUIの構築に挑戦します。手勢の入力（入力値の加工）からデータの変更、そして滑らかなバネアニメーション（Spring）への着地という、モダンなUXに不可欠な一連の実装フローを体系的にマスターします。

---

## 模範コードの全体像

```swift
// ============================================
// 第6章（応用）：Tinder風スワイプカードUI
// ============================================
// ドラッグジェスチャーとアニメーションを組み合わせて、
// カードを左右にスワイプして仕分けるUIを作ります。
// ============================================

import SwiftUI

// MARK: - データモデル

struct Animal: Identifiable {
    let id = UUID()
    let name: String
    let emoji: String
    let description: String
    let color: Color
}

extension Animal {
    static let sampleData: [Animal] = [
        Animal(name: "ネコ", emoji: "🐱", description: "自由気ままなマイペース派", color: .orange),
        Animal(name: "イヌ", emoji: "🐶", description: "忠実で人懐っこい", color: .brown),
        Animal(name: "ウサギ", emoji: "🐰", description: "おとなしくてかわいい", color: .pink),
        Animal(name: "ペンギン", emoji: "🐧", description: "南極のタキシード紳士", color: .cyan),
        Animal(name: "パンダ", emoji: "🐼", description: "笹が大好きなのんびり屋", color: .green),
        Animal(name: "フクロウ", emoji: "🦉", description: "夜型の知恵者", color: .purple),
    ]
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var animals: [Animal] = Animal.sampleData
    @State private var likedAnimals: [Animal] = []
    @State private var dislikedAnimals: [Animal] = []

    var body: some View {
        VStack(spacing: 20) {
            Text("好きな動物は？")
                .font(.title2)
                .bold()

            // スコア表示
            HStack(spacing: 40) {
                Label("\(dislikedAnimals.count)", systemImage: "hand.thumbsdown")
                    .foregroundStyle(.red)
                Label("\(likedAnimals.count)", systemImage: "hand.thumbsup")
                    .foregroundStyle(.green)
            }
            .font(.headline)

            // カードスタック
            ZStack {
                if animals.isEmpty {
                    VStack(spacing: 12) {
                        Text("完了！")
                            .font(.largeTitle)

                        Button("もう一度") {
                            animals = Animal.sampleData.shuffled()
                            likedAnimals = []
                            dislikedAnimals = []
                        }
                        .buttonStyle(.borderedProminent)
                    }
                } else {
                    ForEach(animals.reversed()) { animal in
                        SwipeCardView(animal: animal) { direction in
                            removeCard(animal: animal, direction: direction)
                        }
                    }
                }
            }
            .frame(height: 400)

            // 手動ボタン
            if !animals.isEmpty {
                HStack(spacing: 40) {
                    Button {
                        if let top = animals.last {
                            removeCard(animal: top, direction: .left)
                        }
                    } label: {
                        Image(systemName: "xmark.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.red)
                    }

                    Button {
                        if let top = animals.last {
                            removeCard(animal: top, direction: .right)
                        }
                    } label: {
                        Image(systemName: "heart.circle.fill")
                            .font(.system(size: 50))
                            .foregroundStyle(.green)
                    }
                }
            }

            Spacer()
        }
        .padding()
    }

    func removeCard(animal: Animal, direction: SwipeDirection) {
        withAnimation(.spring(duration: 0.3)) {
            animals.removeAll { $0.id == animal.id }
        }

        switch direction {
        case .left:
            dislikedAnimals.append(animal)
        case .right:
            likedAnimals.append(animal)
        }
    }
}

// MARK: - スワイプ方向

enum SwipeDirection {
    case left, right
}

// MARK: - スワイプカードビュー

struct SwipeCardView: View {
    let animal: Animal
    let onSwipe: (SwipeDirection) -> Void

    @State private var offset: CGSize = .zero
    @State private var rotation: Double = 0

    private let swipeThreshold: CGFloat = 100

    private var swipeProgress: CGFloat {
        min(abs(offset.width) / swipeThreshold, 1.0)
    }

    var body: some View {
        ZStack {
            // カード背景
            RoundedRectangle(cornerRadius: 20)
                .fill(animal.color.opacity(0.15))
                .overlay(
                    RoundedRectangle(cornerRadius: 20)
                        .stroke(animal.color.opacity(0.3), lineWidth: 2)
                )

            // カード内容
            VStack(spacing: 16) {
                Text(animal.emoji)
                    .font(.system(size: 80))

                Text(animal.name)
                    .font(.title)
                    .bold()

                Text(animal.description)
                    .font(.body)
                    .foregroundStyle(.secondary)
            }

            // いいね / NG オーバーレイ
            if offset.width > 0 {
                Text("LIKE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.green)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(-20))
                    .position(x: 80, y: 60)
            } else if offset.width < 0 {
                Text("NOPE")
                    .font(.system(size: 40, weight: .bold))
                    .foregroundStyle(.red)
                    .opacity(swipeProgress)
                    .rotationEffect(.degrees(20))
                    .position(x: 240, y: 60)
            }
        }
        .frame(width: 300, height: 380)
        .shadow(color: .black.opacity(0.1), radius: 8)
        .offset(offset)
        .rotationEffect(.degrees(rotation))
        .gesture(
            DragGesture()
                .onChanged { value in
                    offset = value.translation
                    rotation = Double(value.translation.width / 20)
                }
                .onEnded { value in
                    if value.translation.width > swipeThreshold {
                        // 右スワイプ → LIKE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: 500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.right)
                        }
                    } else if value.translation.width < -swipeThreshold {
                        // 左スワイプ → NOPE
                        withAnimation(.easeOut(duration: 0.3)) {
                            offset = CGSize(width: -500, height: 0)
                        }
                        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                            onSwipe(.left)
                        }
                    } else {
                        // 元に戻す
                        withAnimation(.spring) {
                            offset = .zero
                            rotation = 0
                        }
                    }
                }
        )
    }
}

#Preview {
    ContentView()
}
```

📌 **このアプリは何をするものか：**

画面中央にかわいい動物の絵文字、名前、特徴が書かれたデザインカードが重なって（スタックして）表示されます。ユーザーはこのカードを指で掴んで左右にドラッグすることができます。
右へスワイプすると、カードの上に緑色の「LIKE」という文字がじわっと浮かび上がり、一定以上の距離（100ポイント）を超えて指を離すと、カードはそのまま右側の画面外へと軽快に飛び去り、「LIKE」のカウントが増えます。逆に左へスワイプすると赤い「NOPE」が浮かび上がり、左の画面外へ飛び去って「NOPE」のカウントが増えます。
もし途中で迷って中央付近で指を離した場合は、心地よいバネの弾力（Spring）を伴って自動的に中心の元の位置へと吸い込まれるように戻ります。画面下部にある「❌」と「❤️」のボタンを押しても自動で左右にスワイプされる演出が走り、すべてのカードを仕分けると「完了！」画面になり、シャッフルしてもう一度遊べるインタラクティブなゲーム風UIアプリです。

---

## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
// 手動ボタン（タップイベント）
Button {
    if let top = animals.last {
        removeCard(animal: top, direction: .left)
    }
} label: {
    Image(systemName: "xmark.circle.fill")
        .font(.system(size: 50))
        .foregroundStyle(.red)
}
```

* **何をしているか：** 画面の下部に配置された「❌」や「❤️」のボタンオブジェクトです。ユーザーが画面を「タップ」した瞬間に内部のアクションを実行し、現在スタックの最前面（配列の最後尾 `animals.last` ）にあるカードを特定して、手動でスワイプアニメーション関数（ `removeCard` ）をトリガーしています。
* **なぜこう書くのか：** 高度な手勢操作（ドラッグなど）だけでなく、標準的なタップ操作（Button）からも同じ「カードを消去してカウントする」というデータ処理を行えるように設計するためです。これにより、片手操作でフリックするのが面倒なユーザーに対しても、親指一本でタップしてサクサク仕分けられるというマルチな操作方法（アクセシビリティ）を提供できます。
* **もしこう書かなかったら：** ジェスチャー（ `DragGesture` ）のコード内にすべてのアクションを閉じ込めてしまうと、ボタンを押したときにカードが滑らかに消えていくアニメーションを共通化できず、ボタン用の個別のアニメーションコードを二重に書くことになり、コードの保守性が著しく低下します。

---

### ドラッグジェスチャーとオフセット管理

```swift
@State private var offset: CGSize = .zero
// ...
.offset(offset)
.gesture(
    DragGesture()
        .onChanged { value in
            offset = value.translation
        }
)
```

* **何をしているか：** カードビューの現在の表示位置をずらすための状態変数 `offset` （横・縦のズレを保持する `CGSize` ）を用意し、それを `.offset(offset)` モディファイアでビューに適用しています。そして、ユーザーのドラッグ中（ `.onChanged` ）に、指が動いた開始地点からの差分距離（ `value.translation` ）をそのままリアルタイムに `offset` に代入し続けています。
* **なぜこう書くのか：** SwiftUIは「データ駆動型」のフレームワークであるため、「指が右に50動いた」という物理的なイベントを、一度 `offset` という内部状態（ `@State` ）のデータに変換する必要があります。データが書き換わるたびにビューがミリ秒単位で自動再描画されることで、まるで自分の指の下にカードがピタッと吸い付いて動いているかのような完璧な同期表現が可能になるからです。
* **もしこう書かなかったら：** この状態管理を行わなかったり、 `.onChanged` の処理を省略した場合、画面上のカードをどれだけ指でこすったりドラッグしたりしても、カードは中央に固定されたままピクリとも動かなくなります。

---

### 拡大縮小と回転

```swift
@State private var rotation: Double = 0
// ...
.rotationEffect(.degrees(rotation))
.gesture(
    DragGesture()
        .onChanged { value in
            offset = value.translation
            rotation = Double(value.translation.width / 20)
        }
)
```

* **何をしているか：** カードが左右にドラッグされる動きに合わせて、カード自体を少し「傾ける（回転させる）」ための処理です。ユーザーが右に引っ張れば引っ張るほど、移動距離（ `value.translation.width` ）を20で割った値（角度）が計算され、 `.rotationEffect` を通じてカードが時計回りに傾きます。左に引っ張れば逆方向に傾きます。
* **なぜこう書くのか：** 人間は、画面上のオブジェクトがただ真横にスライド（平行移動）するだけだと「無機質で不自然」だと感じてしまいます。横への移動量に応じて先端が少し扇状に傾くという「物理的なおもちゃのような遊び心（アナログ感）」を数学的計算で少し加えてあげるだけで、アプリのUIが一気に高級で、触っていて気持ちいい上質なUXへと跳ね上がるからです。
* **もしこう書かなかったら：** `rotation = 0` のまま固定するかこの行を削除すると、カードは完全に地面と水平を保ったまま硬直した状態で左右にスライドします。これだけでも機能は満たしますが、どこか安っぽく、操作していてワクワクしない硬い印象のアプリになってしまいます。

---

### ジェスチャーの組み合わせとアニメーション

```swift
.onEnded { value in
    if value.translation.width > swipeThreshold {
        withAnimation(.easeOut(duration: 0.3)) {
            offset = CGSize(width: 500, height: 0)
        }
        // ディレイ後にデータを削除
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) { onSwipe(.right) }
    } else {
        withAnimation(.spring) {
            offset = .zero
            rotation = 0
        }
    }
}
```

* **何をしているか：** ユーザーが指を画面から「離した瞬間（ `.onEnded` ）」の判定です。横の移動距離が閾値（ `swipeThreshold = 100` ）を超えていれば、画面の外（ `width: 500` ）まで一気に滑り出していく減速アニメーション（ `.easeOut` ）をかけ、0.3秒後に親ビューへ通知してデータを削除します。逆に閾値に届かなかった場合は、バネ（ `.spring` ）の滑らかなアニメーションをかけながら、 `offset` と `rotation` を一瞬で `0` （中央の初期位置）にリセットして弾ませています。
* **なぜこう書くのか：** 指を離したあとの挙動こそが、ジェスチャーUIの命だからです。「仕分けを確定させて画面外へ飛ばす」という意思表示と、「やっぱりやめて元に戻す」というキャンセルの意思表示を、指の離された位置（座標）によってロジック分岐させ、それぞれに最適な物理アニメーションを割り当てることで、ストレスフリーな操作感を実現しています。
* **もしこう書かなかったら：** もし `withAnimation` を使わずに値をリセットすると、指を離した瞬間にカードがコマ送りのように「パチッ」とワープして中心に戻るか、画面外へ一瞬でテレポートして消えるという、極めて不自然で不快な画面表示になってしまいます。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
| :--- | :--- | :--- |
| **`DragGesture`** | ユーザーが画面上で指を触れ、そのまま引きずる（ドラッグする）動きを検知・追跡する手勢用API。 | `DragGesture().onChanged { ... }` |
| **`value.translation`** | ドラッグジェスチャーにおいて、タッチが始まった初期位置から現在の指の位置までの「移動距離（差分）」を表すデータ。 | `let distanceX = value.translation.width` |
| **`.rotationEffect`** | ビューを指定した角度（度数法やラジアン）だけ、中心点を軸に回転させて傾けるモディファイア。 | `.rotationEffect(.degrees(15))` |
| **`animals.reversed()`** | 配列の要素の順序をすべて「逆順（後ろから前）」に反転させたビューコレクションを生成するメソッド。 | `ForEach(animals.reversed()) { ... }` |
| **`.spring`** | Appleの物理エンジンに基づいた、現実世界の「バネ」のような弾力と余韻を持った自然なアニメーション効果。 | `withAnimation(.spring) { offset = .zero }` |
| **`opacity(progress)`** | ビューの不透明度を `0.0` （完全透明）から `1.0` （完全不透明）の間で動的にコントロールする設定。 | `.opacity(swipeProgress)` |

---

## 自分の実験メモ

### 🧪 実験1：バネの弾力を限界まで「ビヨンビヨン」にしてみた
* **やったこと：** 指を離して中央に戻るときのバネアニメーションを、標準の `.spring` から詳細にパラメータがいじれる `.spring(response: 0.2, dampingRatio: 0.15)` に書き換えてみた。
* **結果：** カードが中央に戻ったあと、まるでゼリーや強く引っ張ったゴムのように、左右に4〜5回激しくブレて弾んでから静止するようになった。
* **わかったこと：** `dampingRatio` （減衰率）を小さくすればするほど、エネルギーが吸収されずに激しく揺れ続けることが分かった。やりすぎるとアプリがうるさく見えるが、ゲーム性を高めたいUIには非常に強力な武器になると実感した。

### 🧪 実験2：閾値（ `swipeThreshold` ）を「30」まで極端に下げてみた
* **やったこと：** カードが画面外に飛んでいくフリック判定の境界線を `100` から `30` へと大幅に浅くしてみた。
* **結果：** 指をほんの少し（数ミリ）横にピッと動かしただけで、ユーザーが意図していないのにカードが勢いよく画面外へ大暴走して暴発し、誤判定が多発してまともに操作できなくなった。
* **わかったこと：** ジェスチャーUIにおいて「何ポイント指が動いたら確定とするか」という閾値の設定は、ユーザーの心地よさ（誤操作の防止）に直結する。Appleの多くのアプリや実務で `100前後` が推奨される理由が、身をもって理解できた。

---

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `DragGesture` 中に表示される「LIKE」や「NOPE」の文字の透明度（ `.opacity(swipeProgress)` ）の計算式はどうなっていますか？
   * **得られた理解：** `swipeProgress` は「現在の移動距離 ÷ 閾値（100）」で計算され、 `min(..., 1.0)` で最大値が1に制限されていると分かりました。これにより、指を中心に近づければ文字はほぼ透明になり、スワイプ確定ラインである100ポイントに近づくにつれて文字がくっきりと濃くなっていくため、ユーザーに対して「あとどれくらい引っ張れば確定するか」を視覚的に優しくフィードバックする最高のインジケーターになっているのだと感動しました。

2. **質問：** なぜ手動ボタンの削除（ `ContentView` 側）は `.spring(duration: 0.3)` で、手動ドラッグの削除（ `SwipeCardView` 側）は `.easeOut(duration: 0.3)` と、アニメーションの種類が使い分けられているのですか？
   * **得られた理解：** ユーザーの操作体験（直感）に合わせるためです！指で勢いよくフリックしたときは、指の慣性のままに「シュッ」と画面外へ滑り去る `.easeOut` が最も自然に見えます。一方で、画面下のボタンをポンと「タップ」したときは、指の動きの勢いがないため、カードが意思を持って自ら動き出すような「バネの加速感」がある `.spring` を使った方が、機械的ではなく生き生きとした気持ちのいい演出になるという、プロのデザイナー顔負けの使い分けの理由に驚きました。

3. **質問：** `SwipeCardView` の中に書かれている `onSwipe` というプロパティの正体は何ですか？
   * **得られた理解：** これは「クロージャ（関数閉包）」と呼ばれる仕組みで、子ビューから親ビューへ「カードの処理が終わったから、そっちの配列データを更新してね！」と処理を逆流させて依頼するための伝書鳩（コールバック）だと分かりました。SwiftUIでは、見た目（子ビュー）とデータ管理（親ビュー）の責任を綺麗に分離するため、このクロージャによる通信が必須テクニックになるのだと理解できました。

---

## この章のまとめ

この章を通じて、ただ静的に表示されているだけの画面を、ユーザーの指と完全にシンクロさせて動かす **「手触り感（触覚デザイン）」** の極意を学びました。

* **データの同期とアニメーション：** 物理的な移動量を `@State` 変数へと瞬時にマッピングし、回転（ `rotationEffect` ）や透明度（ `opacity` ）の数式と掛け合わせることで、驚くほど有機的なUIが生み出せることを知りました。
* **UX（ユーザー体験）の重要性：** 閾値を超えたら「画面外へ滑らかに消す」、超えなければ「バネで優しく元に戻す」といった丁寧な条件分岐と、それを可能にする `DispatchQueue` の非同期ディレイの組み合わせは、未来の自分がどんなインタラクティブなアプリを作る際にも必ず見返すことになる、極めて貴重な設計資産になりました！
