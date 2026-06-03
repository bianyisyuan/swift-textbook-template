# 第7章：センサーの活用

> 執筆者：卞宜璇
> 最終更新：2026-06-03

## この章で学ぶこと

この章では、iPhoneに内蔵されている高度なハードウェアセンサーである **「モーションセンサー（CoreMotion）」** と、GPSをはじめとする **「位置情報システム（CoreLocation）」** を完全に融合させ、リアルタイムでユーザーの運動量を計測する本格的な **「活動トラッカー（Pedometer & Distance Tracker）」** アプリの構築方法を学びます。

具体的には、デバイスの歩数・歩行距離をバックグラウンドスレッドから非同期で安全に連続取得する `CMPedometer` の制御、移動速度（m/s）から時速（km/h）への単位変換とデータ加工、そしてSwiftUIのグラフィックス描画API（ `Circle().trim` ）を用いた、自動車のタコメーターのような美しい「カスタム速度メーター」の設計手法をマスターします。

---

## 模範コードの全体像

```swift
// ============================================
// 第7章（応用）：歩数計・移動距離トラッカー
// ============================================
// CoreMotion（歩数計）とCoreLocation（移動距離）を
// 組み合わせて、今日の活動を記録するアプリです。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSMotionUsageDescription
//     値: "歩数を計測するためにモーションセンサーを使用します"
//   - NSLocationWhenInUseUsageDescription
//     値: "移動距離を計測するために位置情報を使用します"
// ============================================

import SwiftUI
import CoreMotion
import CoreLocation

// MARK: - 活動トラッカー

@Observable
class ActivityTracker: NSObject, CLLocationManagerDelegate {
    // 歩数関連
    private let pedometer = CMPedometer()
    var stepCount: Int = 0
    var distance: Double = 0     // メートル
    var isPedometerAvailable: Bool = false

    // 位置関連
    private let locationManager = CLLocationManager()
    var currentSpeed: Double = 0  // m/s
    var locations: [CLLocationCoordinate2D] = []

    // 状態
    var isTracking: Bool = false
    var startTime: Date?

    override init() {
        super.init()
        locationManager.delegate = self
        locationManager.desiredAccuracy = kCLLocationAccuracyBest
        locationManager.requestWhenInUseAuthorization()
        isPedometerAvailable = CMPedometer.isStepCountingAvailable()
    }

    func startTracking() {
        isTracking = true
        startTime = Date()
        stepCount = 0
        distance = 0
        locations = []

        // 歩数計の開始
        if isPedometerAvailable {
            pedometer.startUpdates(from: Date()) { [weak self] data, error in
                guard let self = self, let data = data else { return }

                DispatchQueue.main.async {
                    self.stepCount = data.numberOfSteps.intValue
                    if let dist = data.distance {
                        self.distance = dist.doubleValue
                    }
                }
            }
        }

        // 位置情報の開始
        locationManager.startUpdatingLocation()
    }

    func stopTracking() {
        isTracking = false
        pedometer.stopUpdates()
        locationManager.stopUpdatingLocation()
    }

    // MARK: - CLLocationManagerDelegate

    func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
        guard let location = newLocations.last else { return }
        currentSpeed = max(0, location.speed)
        locations.append(location.coordinate)
    }

    // MARK: - 計算プロパティ

    var elapsedTime: TimeInterval {
        guard let start = startTime else { return 0 }
        return Date().timeIntervalSince(start)
    }

    var distanceInKm: Double {
        distance / 1000
    }

    var speedInKmh: Double {
        currentSpeed * 3.6
    }

    var caloriesBurned: Double {
        // 簡易計算：歩数 × 0.04 kcal（目安）
        Double(stepCount) * 0.04
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var tracker = ActivityTracker()
    @State private var timer = Timer.publish(every: 1, on: .main, in: .common).autoconnect()

    var body: some View {
        NavigationStack {
            ScrollView {
                VStack(spacing: 20) {
                    // タイマー表示
                    timerSection

                    // メイン統計
                    statsGrid

                    // スタート/ストップボタン
                    controlButton

                    // 速度メーター
                    if tracker.isTracking {
                        SpeedMeter(speed: tracker.speedInKmh)
                    }
                }
                .padding()
            }
            .navigationTitle("活動トラッカー")
            .onReceive(timer) { _ in
                // タイマーの更新をトリガー（UI再描画のため）
                if tracker.isTracking {
                    // @Observableなので自動で更新される
                }
            }
        }
    }

    // MARK: - タイマーセクション

    private var timerSection: some View {
        VStack(spacing: 4) {
            Text(formatTime(tracker.elapsedTime))
                .font(.system(size: 48, weight: .thin, design: .monospaced))

            if tracker.isTracking {
                Text("計測中")
                    .font(.caption)
                    .foregroundStyle(.green)
            }
        }
        .padding()
    }

    // MARK: - 統計グリッド

    private var statsGrid: some View {
        LazyVGrid(columns: [
            GridItem(.flexible()),
            GridItem(.flexible()),
        ], spacing: 16) {
            StatCard(
                icon: "figure.walk",
                value: "\(tracker.stepCount)",
                unit: "歩",
                color: .blue
            )
            StatCard(
                icon: "map",
                value: String(format: "%.2f", tracker.distanceInKm),
                unit: "km",
                color: .green
            )
            StatCard(
                icon: "flame",
                value: String(format: "%.0f", tracker.caloriesBurned),
                unit: "kcal",
                color: .orange
            )
            StatCard(
                icon: "speedometer",
                value: String(format: "%.1f", tracker.speedInKmh),
                unit: "km/h",
                color: .purple
            )
        }
    }

    // MARK: - コントロールボタン

    private var controlButton: some View {
        Button {
            if tracker.isTracking {
                tracker.stopTracking()
            } else {
                tracker.startTracking()
            }
        } label: {
            HStack {
                Image(systemName: tracker.isTracking ? "stop.fill" : "play.fill")
                Text(tracker.isTracking ? "ストップ" : "スタート")
            }
            .font(.title3)
            .frame(maxWidth: .infinity)
            .padding()
            .background(tracker.isTracking ? Color.red : Color.green)
            .foregroundStyle(.white)
            .clipShape(RoundedRectangle(cornerRadius: 16))
        }
    }

    // MARK: - 時間フォーマット

    func formatTime(_ interval: TimeInterval) -> String {
        let hours = Int(interval) / 3600
        let minutes = Int(interval) / 60 % 60
        let seconds = Int(interval) % 60
        return String(format: "%02d:%02d:%02d", hours, minutes, seconds)
    }
}

// MARK: - 統計カード

struct StatCard: View {
    let icon: String
    let value: String
    let unit: String
    let color: Color

    var body: some View {
        VStack(spacing: 8) {
            Image(systemName: icon)
                .font(.title2)
                .foregroundStyle(color)

            Text(value)
                .font(.title)
                .bold()

            Text(unit)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .frame(maxWidth: .infinity)
        .padding()
        .background(
            RoundedRectangle(cornerRadius: 12)
                .fill(color.opacity(0.08))
        )
    }
}

// MARK: - 速度メーター

struct SpeedMeter: View {
    let speed: Double

    var body: some View {
        VStack(spacing: 8) {
            Text("現在の速度")
                .font(.caption)
                .foregroundStyle(.secondary)

            ZStack {
                Circle()
                    .trim(from: 0, to: 0.75)
                    .stroke(.gray.opacity(0.2), lineWidth: 8)
                    .rotationEffect(.degrees(135))

                Circle()
                    .trim(from: 0, to: min(speed / 15.0, 1.0) * 0.75)
                    .stroke(speedColor, style: StrokeStyle(lineWidth: 8, lineCap: .round))
                    .rotationEffect(.degrees(135))
                    .animation(.spring, value: speed)

                VStack {
                    Text(String(format: "%.1f", speed))
                        .font(.system(size: 32, weight: .bold, design: .monospaced))
                    Text("km/h")
                        .font(.caption)
                        .foregroundStyle(.secondary)
                }
            }
            .frame(width: 150, height: 150)
        }
        .padding()
    }

    var speedColor: Color {
        if speed < 4 { return .green }
        if speed < 8 { return .orange }
        return .red
    }
}

#Preview {
    ContentView()
}
```

📌 **このアプリは何をするものか：**

ユーザーがウォーキングやランニングの際に起動する「アクティビティ計測・記録アプリ」です。
画面上部には計測開始からの経過時間がタイマー（時:分:秒）でリアルタイムに刻まれます。中央の「グリッド表示」には、iPhoneのモーションセンサーが検知した正確な「歩数」、センサーが算出した「移動距離（km）」、歩数から自動計算された「消費カロリー（kcal）」、そしてGPSから算出した「現在の時速（km/h）」の4つの統計データが美しく並びます。
「スタート」ボタンを押すと一斉にリアルタイム計測が始まり、動的に表示される画面最下部の「速度メーター」が、ユーザーの移動スピードに応じてアークゲージ（円弧）を滑らかに伸縮させます。さらに、速度が上がるとメーターの色が「緑色（徒歩） ➔ オレンジ色（早走り） ➔ 赤色（自転車・車移動など）」へと自動変化する、近未来的で視覚的にも楽しい高機能トラッカーアプリです。

---

## コードの詳細解説

### 歩数計（CMPedometer）の制御

```swift
if isPedometerAvailable {
    pedometer.startUpdates(from: Date()) { [weak self] data, error in
        guard let self = self, let data = data else { return }

        DispatchQueue.main.async {
            self.stepCount = data.numberOfSteps.intValue
            if let dist = data.distance {
                self.distance = dist.doubleValue
            }
        }
    }
}
```

* **何をしているか：** `CMPedometer` の `startUpdates` メソッドを呼び出し、計測を開始した瞬間（ `Date()` ）からの歩数と距離の変更イベントをリアルタイムに監視（サブスクライブ）しています。データが更新されるたびにコールバックが走り、安全にメモリ管理を行う記述（ `[weak self]` ）を挟んだ上で、メインスレッド（ `DispatchQueue.main.async` ）に戻してクラスのプロパティ（ `stepCount` / `distance` ）を書き換えています。
* **なぜこう書くのか：** モーションセンサー（CoreMotion）のデータ受信ブロックは、UIを管理するメインスレッドとは異なるスレッド（バックグラウンドスレッド）で呼び出されます。iOSのルールとして、画面描画（SwiftUIのデータ同期）は必ずメインスレッドで実行しなければアプリがクラッシュする危険があるため、 `DispatchQueue.main.async` を挟んで処理をメインスレッドへ明示的に投げ戻すのが絶対的なルールだからです。
* **もしこう書かなかったら：** `DispatchQueue.main.async` を記述せずに直接プロパティを更新すると、Xcodeのデバッグ画面に紫色のアラート（Main Thread Checker違反）が大量に出力され、最悪の場合、画面の表示が一切更新されなくなったり、アプリが突然強制終了（クラッシュ）したりします。

---

### CoreLocationとの連携と速度取得

```swift
func locationManager(_ manager: CLLocationManager, didUpdateLocations newLocations: [CLLocation]) {
    guard let location = newLocations.last else { return }
    currentSpeed = max(0, location.speed)
    locations.append(location.coordinate)
}
```

* **何をしているか：** 位置情報マネージャーのデリゲートメソッド（ `didUpdateLocations` ）です。GPSの捕捉によって新しい位置情報の配列（ `newLocations` ）が届くたびに、その最新の要素（ `.last` ）を取り出します。そして、そこに含まれる移動速度（ `location.speed` ）を安全に処理した上で `currentSpeed` に格納し、同時に移動した奇跡の座標を配列 `locations` に追加記録しています。
* **なぜこう書くのか：** `location.speed` は、GPSが一時的に電波障害や初期化ミスを起こすと、エラー値として「マイナスの値（ `-1.0` など）」を返してくる仕様になっています。そのまま時速に変換すると画面に負の速度が表示されてしまうバグが起きるため、 `max(0, location.speed)` を使うことで、どんな異常データが来ても確実に最低値を `0` （停止状態）で綺麗に踏みとどまらせるための防衛コードが必要だからです。
* **もしこう書かなかったら：** 地下に入ったり、GPSの電波が乱れたりした瞬間に、速度メーターが突然マイナスの数値を指し示し、ユーザーに対して非常に不正確でバグだらけな印象を与えるUIになってしまいます。

---

### 計算プロパティによる統計データの加工

```swift
var distanceInKm: Double {
    distance / 1000
}

var speedInKmh: Double {
    currentSpeed * 3.6
}

var caloriesBurned: Double {
    Double(stepCount) * 0.04
}
```

* **何をしているか：** センサーから取得した生のシステムデータ（メートル、秒速メートル）を、一般ユーザーが見て一目で理解できる単位（キロメートル、時速キロメートル、消費カロリー）へとリアルタイムに変換・算出するための「計算プロパティ群」です。
* **なぜこう書くのか：** `CMPedometer` から取得できる距離は「メートル（m）」であり、 `CLLocation` から取れる速度は「秒速（m/s）」という非常に細かい単位です。これらを直接画面に出すと「現在の速度：1.1111 m/s」という、人間にはピンと来ない表示になってしまいます。そのため、データを保持する変数自体はシステム準拠の生データ（ Double ）で綺麗に持っておき、ビューに表示する段階で「1000で割る」「3.6を掛ける」といった加工を施すことで、ロジックと表示の責任を美しく分離できるからです。
* **もしこう書かなかったら：** センサーデータが更新されるたびに、わざわざ手動で「時速に変換した変数」と「メートルに変換した変数」を何種類も別々に計算して `@State` に保存し直す必要があり、コードが非常に肥大化し、計算ミスのバグの原因になります。

---

### SwiftUIによるカスタム速度メーター（SpeedMeter）の設計

```swift
Circle()
    .trim(from: 0, to: 0.75)
    .stroke(.gray.opacity(0.2), lineWidth: 8)
    .rotationEffect(.degrees(135))

Circle()
    .trim(from: 0, to: min(speed / 15.0, 1.0) * 0.75)
    .stroke(speedColor, style: StrokeStyle(lineWidth: 8, lineCap: .round))
    .rotationEffect(.degrees(135))
    .animation(.spring, value: speed)
```

* **何をしているか：** 円（ `Circle()` ）を `.trim` を使って「全体の75%（0.75）」だけに切り取り、それを `.rotationEffect` で135度回転させることで、下側が開いた「270度の円弧のメーター土台」を作っています。その上に、現在の速度（マックスを15.0km/hと想定）の割合に応じた長さの円弧を重ねて描き、速度変化（ `value: speed` ）に対して心地よいバネアニメーション（ `.spring` ）を適用して針（ゲージ）が動くようにしています。
* **なぜこう書くのか：** SwiftUIには「速度メーター」という標準コンポーネントは存在しません。しかし、このように基本図形である `Circle` に対して「一部分を切り取る（ `.trim` ）」「線のスタイルを丸める（ `lineCap: .round` ）」というモディファイアの組み合わせテクニックを使うだけで、UIKit時代のような重い描画ライブラリを一切導入することなく、完全オリジナルの美しい円弧メーターを自作できるからです。
* **もしこう書かなかったら：** メーター表示を諦めて、単なる数字の「0.0 km/h」というテキスト表示だけで終わってしまい、スポーツアプリやアクティビティアプリとして最も重要な「走っていて今どれくらいのスピードが出ているのか」という直感的なユーザー体験（視覚的ギミック）が失われてしまいます。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
| :--- | :--- | :--- |
| **`CMPedometer`** | iPhoneのモーションコプロセッサにアクセスし、歩数、歩行距離、上った階段の段数などを自動計測するCoreMotionの主役API。 | `let pedometer = CMPedometer()` |
| **`Timer.publish`** | 指定された時間間隔（例：1秒）ごとに定期的なイベント（シグナル）を発行し続けるストリーム（結合API）。 | `Timer.publish(every: 1, on: .main, ...)` |
| **`.onReceive`** | `Timer` などのパブリッシャーから送られてくる定期的な通知をビューがキャッチし、内部の処理を実行するモディファイア。 | `.onReceive(timer) { _ in ... }` |
| **`.trim(from:to:)`** | パスや図形（円や四角など）の描画範囲を割合（0.0〜1.0）で切り取り、部分的な線を描画するためのグラフィックスAPI。 | `Circle().trim(from: 0, to: 0.75)` |
| **`StrokeStyle`** | 描画する線の太さ（lineWidth）だけでなく、線の先端の形を丸くする（ `lineCap: .round` ）などの詳細なスタイルを定義する構造体。 | `StrokeStyle(lineWidth: 8, lineCap: .round)` |
| **`[weak self]`** | クロージャ（非同期処理のブロック）の内部で、自身（クラス）への強固なメモリ参照を解放し、画面を閉じた際のメモリリークを完全に防ぐ記述。 | `pedometer.startUpdates(...) { [weak self] data, _ in ... }` |

---

## 自分の実験メモ

### 🧪 実験1：タイマーの更新間隔を「0.1秒」に変えてメーターの動きを検証した
* **やったこと：** `ContentView` のタイマー発行設定を `every: 1` （1秒ごと）から `every: 0.1` （0.1秒ごと）へと書き換え、タイマー表示を小数点第1位まで表示させてみた。
* **結果：** 経過時間タイマーの表示が超高速でスムーズに流れるようになり、さらにGPSの細かな移動速度の数値変化が画面に反映されるラグが減り、メーターの動きがさらに未来感のあるものになった。
* **わかったこと：** 更新頻度を上げるとUIは非常に滑らかになるが、その分アプリが毎秒10回も画面を書き換える（CPUに負荷をかける）ことになる。活動トラッカーのようなバッテリー持ちが最優先されるアプリでは、模範コード通りの「1秒（ `every: 1` ）」が最適解であるというトレードオフの関係が分かった。

### 🧪 実験2：速度メーターの最大値（上限）を書き換えて、自動車スケールにしてみた
* **やったこと：** `SpeedMeter` 内の比率計算 `speed / 15.0` の部分を、 `speed / 120.0` に変更し、 `speedColor` の閾値を「時速40km/h以下 ➔ 緑」「時速80km/h以下 ➔ オレンジ」「それ以上 ➔ 赤」へと改変してみた。
* **結果：** 人間の歩行用メーターから、完全に「自動車のスピードメーター」の挙動へとスケールが切り替わった。Macのシミュレータの「Features > Location > Freeway Drive」の擬似走行テストで、時速が100km/hを超えたあたりでメーターが真っ赤に染まる演出を確認できた。
* **わかったこと：** 計算プロパティやトリミングの比率の数字（分母）を変えるだけで、同じグラフィックスコンポーネントでありながら、ランニング用、自転車用、ドライブ用など、あらゆるユースケースのアプリへ無限に応用・流用が可能である汎用性の高さを実感した。

---

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：** `CMPedometer.isStepCountingAvailable()` はなぜわざわざ初期化（ `init` ）のタイミングでチェックしているのですか？
   * **得られた理解：** iOSデバイスのすべてに歩数センサーが載っているわけではないからです（例：一部の古いiPadや、Mac上で動くシミュレータの特定環境など）。もしこの確認をせずにいきなり計測をスタート（ `startUpdates` ）させてしまうと、センサー非搭載の端末ではアプリがその瞬間にエラーを起こして不正終了してしまいます。事前に「この端末は歩数計が使えるか？」をボルトロックのように確認し、安全なフラグ（ `isPedometerAvailable` ）として持っておくのが大原則なのだと分かりました。

2. **質問：** `ContentView` の中にある `.onReceive(timer)` の中でコメントアウトされている「 `@Observable` なので自動で更新される」とはどういう意味ですか？空っぽでも動くのはなぜですか？
   * **得られた理解：** 新しい `@Observable` マクロの凄さが分かりました！ `.onReceive(timer)` が毎秒イベントを検知すると、SwiftUIは「このビューの中に描かれている何かが変わる可能性がある」と判断し、画面全体の再描画を自動でかけます。その瞬間に、 `tracker.elapsedTime` （計算プロパティ）の引き算が再計算されるため、クロージャの中身に手動で更新コード（ `self.time = ...` など）を一行も書かなくても、自動的にタイマーが1秒ずつ進むという、魔法のようなスマートな同期の裏側を理解できました。

3. **質問：** `Circle().trim(from: 0, to: 0.75)` の回転効果（ `.rotationEffect(.degrees(135))` ）という計算は、なぜ135度という中途半端な数字なのですか？
   * **得られた理解：** 綺麗な対称形のメーターを作るための美しい数学的トリックでした。円の全周は360度で、その75%（0.75）を切り取ると「270度の円弧」が残ります。切り取られた残りの「90度の空白」を真下（南向き）に綺麗に配置したい場合、円の開始地点（デフォルトは時計の3時の位置）から時計回りに135度回転させてあげると、左右のバランスが完全に均等になり、車のメーターと同じおなじみのデザインになるという幾何学的な理由に深く感動しました。

---

## この章のまとめ

この章を終えたことで、iPhoneというデバイスが持つ強力な物理センサー（運動量とGPS空間情報）をコードの力で完全に掌握し、1つの統合型アプリケーションに仕立て上げる技術を習得しました。

* **マルチスレッドへの完全な理解：** ハードウェアの裏側で動く「バックグラウンド処理（センサー監視）」と、ユーザーの目の前で動く「メインスレッド（UI更新）」という、中級以上のiOSエンジニアに絶対に求められるマルチスレッドプログラミングの基礎（ `DispatchQueue.main.async` ）の必然性を完璧にマスターできました。
* **表現力の拡大：** 既存のテキスト表示やリスト表示を飛び越え、 `Circle().trim` などのグラフィカルな描画APIと物理的な数値（時速・歩数）をリアルタイムに掛け合わせる楽しさを知ったことで、これまでに学んだ SwiftData などの知識と組み合わせ、今後さらにリッチで実用的なスポーツ・ライフスタイル系アプリを自由自在に開発できる強固な土台が完成しました！
