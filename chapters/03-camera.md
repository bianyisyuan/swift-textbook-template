# 第3章：カメラとアルバムの利用

> 執筆者：卞宜璇
> 最終更新：2026-05-22

## この章で学ぶこと

この章では、iOS 17の標準コンポーネントである **`PhotosPicker`** を使ったフォトライブラリからの画像選択と、UIKitのコンポーネントである **`UIImagePickerController`** を SwiftUI に組み込んでカメラ撮影機能を実装する方法について学びます。

具体的には、SwiftUIのデータフロー（`@State`, `@Binding`）を活用し、非同期処理（`Task / async / await`）による画像データの読み込みや、UIKit と SwiftUI の架け橋となる **`UIViewControllerRepresentable`** プロトコルの設計思想をマスターします。

---

## 模範コードの全体像

```swift
import SwiftUI
import PhotosUI // 💡 PhotosPickerを使用するために必要

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImage: Image?
    @State private var isShowingCamera = false
    @State private var capturedUIImage: UIImage?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                // 画像表示エリア
                imageDisplayArea

                // ボタンエリア
                HStack(spacing: 20) {
                    // フォトライブラリから選択
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    // カメラで撮影
                    Button {
                        isShowingCamera = true
                    } label: {
                        Label("カメラ", systemImage: "camera")
                    }
                    .buttonStyle(.borderedProminent)
                }
                .padding()
            }
            .navigationTitle("写真アプリ")
            .onChange(of: selectedItem) { _, newItem in
                Task {
                    await loadImage(from: newItem)
                }
            }
            .fullScreenCover(isPresented: $isShowingCamera) {
                CameraView(capturedImage: $capturedUIImage)
            }
            .onChange(of: capturedUIImage) { _, newImage in
                if let uiImage = newImage {
                    selectedImage = Image(uiImage: uiImage)
                }
            }
        }
    }

    // MARK: - 画像表示エリア
    @ViewBuilder
    private var imageDisplayArea: some View {
        if let image = selectedImage {
            image
                .resizable()
                .aspectRatio(contentMode: .fit)
                .frame(maxHeight: 400)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 4)
                .padding()
        } else {
            RoundedRectangle(cornerRadius: 16)
                .fill(.gray.opacity(0.1))
                .frame(height: 300)
                .overlay {
                    VStack(spacing: 8) {
                        Image(systemName: "photo")
                            .font(.system(size: 48))
                            .foregroundStyle(.gray)
                        Text("写真を選択または撮影してください")
                            .font(.caption)
                            .foregroundStyle(.secondary)
                    }
                }
                .padding()
        }
    }

    // MARK: - 画像の読み込み
    func loadImage(from item: PhotosPickerItem?) async {
        guard let item = item else { return }

        do {
            if let data = try await item.loadTransferable(type: Data.self),
               let uiImage = UIImage(data: data) {
                selectedImage = Image(uiImage: uiImage)
            }
        } catch {
            print("画像の読み込みに失敗: \(error.localizedDescription)")
        }
    }
}

// MARK: - カメラビュー（UIKit連携）
struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(_ uiViewController: UIImagePickerController, context: Context) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
        let parent: CameraView

        init(_ parent: CameraView) {
            self.parent = parent
        }

        func imagePickerController(
            _ picker: UIImagePickerController,
            didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
        ) {
            if let image = info[.originalImage] as? UIImage {
                parent.capturedImage = image
            }
            parent.dismiss()
        }

        func imagePickerControllerDidCancel(_ picker: UIImagePickerController) {
            parent.dismiss()
        }
    }
}

#Preview {
    ContentView()
}
```

**📌 このアプリは何をするものか：**

ユーザーが「ライブラリ」ボタンを押すと、iOS標準のアルバム選択画面が立ち上がり、安全に写真を選択できます。また、「カメラ」ボタンを押すと、画面全体がカメラ撮影モードに切り替わります（`fullScreenCover`）。
撮影または選択された画像データは非同期処理でSwiftUIが扱える形式に変換され、画面中央の角丸カード内に美しく表示されます。

---

## コードの詳細解説

### 1. `PhotosPicker` と非同期データロード（`async/await`）

```swift
PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
    }
}
```

* **何をしているか：**
  iOS 17対応の `PhotosPicker` を配置し、ユーザーが画像を選んだ瞬間に `.onChange` センサーが感知して、バックグラウンドの非同期処理（`Task`）で実際の画像データ（`Data`）をロードしています。
* **なぜこう書くのか：**
  画像ファイルはサイズが大きいため、データの読み込みをメインスレッド（UIを動かす場所）で行うと、読み込み中に画面がカクついたりフリーズしたりします。そのため、`try await item.loadTransferable` を使い、裏側で安全にロードを待つ非同期処理設計にする必要があります。
* **もしこう書かなかったら：**
  非同期処理（`async/await`）を正しく使わずに同期処理として大きな画像をロードしようとすると、最悪の場合、OSに「重いバグのあるアプリ」と判定されてアプリが強制終了させられます。

---

### 2. `UIViewControllerRepresentable` による UIKit（カメラ）の統合

```swift
struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    
    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator // 代理人の設定
        return picker
    }
    // ...
}
```

* **何をしているか：**
  SwiftUIにはまだ標準の「カメラ撮影ビュー」が用意されていないため、従来の UIKit にある `UIImagePickerController` を SwiftUI の中で通常のビュー（`CameraView`）として扱えるようにラップ（変換）しています。
* **なぜこう書くのか：**
  SwiftUI だけでカバーできない古い機能や高度なハードウェア操作（カメラ、通知、高度なテキスト制御など）は、`UIViewControllerRepresentable` を仲介役にすることで、過去の豊富な UIKit の資産をそのまま再利用できる設計になっているためです。
* **もしこう書かなかったら：**
  このプロトコルを使わない場合、SwiftUI でカメラ機能をゼロから実装することは不可能であり、既存の強力なシステムカメラ画面（シャッターボタンやグリッド線）の恩恵を受けることができなくなります。

---

### 3. `Coordinator`（コーディネーター）によるイベントの逆流

```swift
class Coordinator: NSObject, UIImagePickerControllerDelegate, UINavigationControllerDelegate {
    let parent: CameraView
    // ...
    func imagePickerController(_ picker: UIImagePickerController, didFinishPickingMediaWithInfo info: [ContextKey: Any]) {
        if let image = info[.originalImage] as? UIImage {
            parent.capturedImage = image // SwiftUI側の状態にデータを送る
        }
        parent.dismiss() // 画面を閉じる
    }
}
```

* **何をしているか：**
  UIKit のカメラ側で「写真の撮影が完了した」「キャンセルされた」というイベントを検知し、SwiftUI 側のバインド変数（`$capturedUIImage`）にデータを逆流させて通知する「中継役（代理人）」を実装しています。
* **なぜこう書くのか：**
  UIKit は「デリゲート（Delegate）パターン」という通知規約で動いていますが、SwiftUI は「状態管理（Data-Driven）」で動いています。この2つの異なるルールを繋ぐために、内部クラスとして `Coordinator` を作り、カメラの終了イベントを SwiftUI の変数更新に翻訳してあげる必要があるためです。
* **もしこう書かなかったら：**
  カメラを起動して写真を撮ることはできても、撮影した写真を SwiftUI 側に持ち帰ることができず、カメラを閉じる（`dismiss`）こともできなくなってしまいます。

---

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
| :--- | :--- | :--- |
| **`PhotosPicker`** | iOS 16から追加された、プライバシーに配慮した安全な標準画像選択UI。 | `PhotosPicker(selection: $item, matching: .images)` |
| **`loadTransferable`** | 選択されたアイテムから、`Data` や `Image` などの具体的な型へと非同期で安全に変換するメソッド。 | `try await item.loadTransferable(type: Data.self)` |
| **`Task { ... }`** | 同期的な関数（`onChange`等）の中から、非同期関数（`async`）を安全に呼び出すための「非同期のチケット」。 | `.onChange(of: item) { Task { await load() } }` |
| **`UIViewControllerRepresentable`** | UIKit の画面（`UIViewController`）を SwiftUI のビューとして使えるようにするプロトコル。 | `struct CameraView: UIViewControllerRepresentable` |
| **`makeUIViewController`** | `UIViewControllerRepresentable` 内で、表示したい UIKit のコンポーネントを初期化する必須メソッド。 | `func makeUIViewController(context:) -> Picker` |
| **`Coordinator`** | UIKit からの通知（Delegate）を受け取り、SwiftUI の状態（`@Binding`）へデータを渡すための中継クラス。 | `class Coordinator: NSObject, UIImage...Delegate` |

---

## 自分の実験メモ

### 🧪 実験1：`PhotosPicker` の `matching` 条件を動画に変更してみた
* **やったこと：** `matching: .images` だった部分を `matching: .videos` や `matching: .anyOf([.images, .videos])` に書き換えてみた。
* **結果：** ピッカーが立ち上がった際、画像だけでなく端末内の動画も選択可能になり、複数形式の混在に対応できた。
* **わかったこと：** `matching` 引数をコントロールするだけで、アプリに必要なメディアタイプ（写真のみ、動画のみ、あるいは両方）を極めて簡単に制限・管理できると分かった。

### 🧪 実験2：カメラの `sourceType` をシミュレータで動かしてみた
* **やったこと：** 実機ではなく、Mac上の Xcode シミュレータで「カメラ」ボタンを押してみた。
* **結果：** アプリが `UIImagePickerController.sourceType = .camera` の行でクラッシュ、もしくは動作しなかった（シミュレータには物理カメラがないため）。
* **わかったこと：** 実際の開発では `UIImagePickerController.isSourceTypeAvailable(.camera)` を使って、事前にカメラが使える端末かどうかをチェックする防衛コードを書くのがプロの設計であると痛感した。

---
