# 第3章：写真を選択・撮影して表示するアプリ

> 執筆者：林 楽駿
> 最終更新：2026-07-29

## この章で学ぶこと

この章では、iPhoneの写真ライブラリから写真を選ぶ方法を学ぶ。また、カメラで写真を撮って、その写真を画面に表示する方法も学ぶ。 

## 模範コードの全体像

```swift
// ============================================
// 第3章（基本）：写真を選択・撮影して表示するアプリ
// ============================================
// PhotosPickerを使ってフォトライブラリから写真を選択し、画面に表示します。
// 「カメラ」ボタンで撮影もできます。
//
// 【動作環境】
//   - フォトライブラリから選択：シミュレータでも動作します。
//   - カメラ撮影：実機（iPhone / iPad）専用。シミュレータでは
//     カメラボタンが自動的に無効化されます。
//
// 【注意】実機でカメラを使う場合は Info.plist に以下を追加してください：
//   - NSCameraUsageDescription
//     値: "撮影した写真を表示するためにカメラを使用します"
// ============================================

import SwiftUI
import PhotosUI

struct ContentView: View {
    @State private var selectedItem: PhotosPickerItem?
    @State private var selectedImage: Image?
    @State private var isShowingCamera = false
    @State private var capturedUIImage: UIImage?

    var body: some View {
        NavigationStack {
            VStack(spacing: 20) {
                imageDisplayArea

                HStack(spacing: 20) {
                    PhotosPicker(
                        selection: $selectedItem,
                        matching: .images
                    ) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    Button {
                        isShowingCamera = true
                    } label: {
                        Label("カメラ", systemImage: "camera")
                    }
                    .buttonStyle(.borderedProminent)
                    .disabled(!UIImagePickerController.isSourceTypeAvailable(.camera))
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

struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss

    func makeUIViewController(context: Context) -> UIImagePickerController {
        let picker = UIImagePickerController()
        picker.sourceType = .camera
        picker.delegate = context.coordinator
        return picker
    }

    func updateUIViewController(
        _ uiViewController: UIImagePickerController,
        context: Context
    ) {}

    func makeCoordinator() -> Coordinator {
        Coordinator(self)
    }

    class Coordinator: NSObject,
                       UIImagePickerControllerDelegate,
                       UINavigationControllerDelegate {

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

        func imagePickerControllerDidCancel(
            _ picker: UIImagePickerController
        ) {
            parent.dismiss()
        }
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

このアプリは、写真ライブラリから写真を選んだり、カメラで写真を撮ったりできるアプリである。

選んだ写真や撮った写真は、アプリの画面に表示される。

まだ写真を選んでいないときは、「写真を選択または撮影してください」と表示される。

## コードの詳細解説

### 1. 写真や画面の状態を保存する

```swift
@State private var selectedItem: PhotosPickerItem?
@State private var selectedImage: Image?
@State private var isShowingCamera = false
@State private var capturedUIImage: UIImage?
```

**何をしているか：**

写真やカメラ画面の状態を保存している。

`selectedItem`は、ライブラリで選んだ写真を保存する。

`selectedImage`は、画面に表示する写真を保存する。

`isShowingCamera`は、カメラ画面を開くかどうかを決める。

`capturedUIImage`は、カメラで撮った写真を保存する。

**なぜこう書くのか：**

`@State`を使うと、値が変わったときに画面も自動で変わるからである。

**もしこう書かなかったら：**

写真を選んでも、画面が変わらないことがある。

---

### 2. 写真ライブラリを開く

```swift
PhotosPicker(
    selection: $selectedItem,
    matching: .images
) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
```

**何をしているか：**

「ライブラリ」ボタンを作っている。

ボタンを押すと、iPhoneの写真ライブラリが開く。

**なぜこう書くのか：**

`PhotosPicker`を使うと、簡単に写真を選べるからである。

`matching: .images`は、写真だけを選べるようにしている。

**もしこう書かなかったら：**

写真ライブラリから写真を選べなくなる。

---

### 3. カメラボタンを作る

```swift
Button {
    isShowingCamera = true
} label: {
    Label("カメラ", systemImage: "camera")
}
.disabled(!UIImagePickerController.isSourceTypeAvailable(.camera))
```

**何をしているか：**

「カメラ」ボタンを押すと、カメラ画面を開く。

また、カメラが使えないときは、ボタンを押せないようにしている。

**なぜこう書くのか：**

シミュレータにはカメラがないからである。

**もしこう書かなかったら：**

シミュレータでもカメラボタンを押せるようになり、エラーになることがある。

---

### 4. 写真が選ばれたことを確認する

```swift
.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
    }
}
```

**何をしているか：**

新しい写真が選ばれたときに、その写真を読み込んでいる。

**なぜこう書くのか：**

写真を選んだだけでは、すぐに画面に表示できないからである。

一度写真のデータを読み込む必要がある。

**もしこう書かなかったら：**

写真を選んでも、画面に表示されない。

---

### 5. 写真を画面に表示する

```swift
if let image = selectedImage {
    image
        .resizable()
        .aspectRatio(contentMode: .fit)
        .frame(maxHeight: 400)
}
```

**何をしているか：**

写真があるとき、その写真を画面に表示している。

**なぜこう書くのか：**

`resizable()`を使うと、写真の大きさを変えられる。

`aspectRatio(contentMode: .fit)`を使うと、写真の形をこわさずに表示できる。

**もしこう書かなかったら：**

写真が大きすぎたり、形が変になったりすることがある。

---

### 6. 写真がないときの画面

```swift
RoundedRectangle(cornerRadius: 16)
    .fill(.gray.opacity(0.1))
    .frame(height: 300)
```

**何をしているか：**

まだ写真がないときに、灰色の四角い場所を表示している。

その中に写真のマークと説明文を表示している。

**なぜこう書くのか：**

ユーザーに、どこに写真が表示されるのか分かりやすくするためである。

**もしこう書かなかったら：**

写真がないとき、画面が空白になって分かりにくい。

---

### 7. 選んだ写真を読み込む

```swift
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
```

**何をしているか：**

選んだ写真をデータとして読み込み、画面に表示できる写真に変えている。

**なぜこう書くのか：**

`PhotosPickerItem`は、そのままでは画面に表示できないからである。

一度`Data`にして、そのあと`UIImage`と`Image`に変えている。

**もしこう書かなかったら：**

選んだ写真を画面に表示できない。

---

### 8. カメラ画面を表示する

```swift
.fullScreenCover(isPresented: $isShowingCamera) {
    CameraView(capturedImage: $capturedUIImage)
}
```

**何をしているか：**

`isShowingCamera`が`true`になったとき、カメラ画面を全画面で表示する。

**なぜこう書くのか：**

カメラは画面全体で使うことが多いからである。

**もしこう書かなかったら：**

カメラボタンを押しても、カメラ画面が表示されない。

---

### 9. 撮った写真を表示する

```swift
.onChange(of: capturedUIImage) { _, newImage in
    if let uiImage = newImage {
        selectedImage = Image(uiImage: uiImage)
    }
}
```

**何をしているか：**

カメラで撮った写真を、画面に表示する写真に変えている。

**なぜこう書くのか：**

カメラの写真は`UIImage`であるが、SwiftUIでは`Image`を使うからである。

**もしこう書かなかったら：**

写真を撮っても、画面に表示されない。

---

### 10. SwiftUIでカメラを使う

```swift
struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
}
```

**何をしているか：**

SwiftUIの画面で、iPhoneのカメラ機能を使えるようにしている。

**なぜこう書くのか：**

カメラ機能の`UIImagePickerController`はUIKitの機能だからである。

SwiftUIで使うために、`UIViewControllerRepresentable`が必要になる。

**もしこう書かなかったら：**

SwiftUIからカメラを開けない。

---

### 11. カメラを作る

```swift
func makeUIViewController(context: Context) -> UIImagePickerController {
    let picker = UIImagePickerController()
    picker.sourceType = .camera
    picker.delegate = context.coordinator
    return picker
}
```

**何をしているか：**

カメラ画面を作っている。

`sourceType = .camera`で、カメラを使うように設定している。

**なぜこう書くのか：**

`UIImagePickerController`は、カメラを開くために必要だからである。

**もしこう書かなかったら：**

カメラが開かない。

---

### 12. 写真を撮ったあとの処理

```swift
if let image = info[.originalImage] as? UIImage {
    parent.capturedImage = image
}
parent.dismiss()
```

**何をしているか：**

撮った写真を保存して、そのあとカメラ画面を閉じている。

**なぜこう書くのか：**

撮った写真を前の画面に渡す必要があるからである。

**もしこう書かなかったら：**

撮った写真が保存されない。

また、カメラ画面も閉じないことがある。

---

### 13. キャンセルしたときの処理

```swift
func imagePickerControllerDidCancel(
    _ picker: UIImagePickerController
) {
    parent.dismiss()
}
```

**何をしているか：**

カメラをキャンセルしたとき、元の画面に戻る。

**なぜこう書くのか：**

キャンセルしたあとも、カメラ画面を閉じる必要があるからである。

**もしこう書かなかったら：**

キャンセルしてもカメラ画面が閉じない。

## 新しく学んだSwiftの文法・API

| 項目                        | 説明             | 使用例                                        |
| ------------------------- | -------------- | ------------------------------------------ |
| `@State`                  | 値を保存して、画面を更新する | `@State private var selectedImage: Image?` |
| `@Binding`                | 親画面と子画面で同じ値を使う | @State private var capturedUIImage: UIImage?
 CameraView(capturedImage: $capturedUIImage)
`@Binding var capturedImage: UIImage?`  
parent.capturedImage = image。                                                            |
| `PhotosPicker`            | 写真ライブラリから写真を選ぶ | `PhotosPicker(selection: $selectedItem)`   |
| `onChange`                | 値が変わったことを確認する  | `onChange(of: selectedItem)              |
| `Task`                    | 時間がかかる処理を行う    | `Task { await loadImage(...) }`            |
| `async`                   | 時間がかかる処理につける   | `func loadImage(...) async`                |
| `await`                   | 処理が終わるまで待つ     | `await loadImage(from: newItem)`           |
| `if let`                  | 値があるか安全に確認する   | `if let image = selectedImage`             |
| `guard let`               | 値がないとき処理を終わる   | `guard let item = item else { return }`    |
| `fullScreenCover`         | 別の画面を全画面で開く    | `.fullScreenCover(...)`                    |
| `UIImagePickerController` | カメラを使う         | `UIImagePickerController()`                |
| `dismiss`                 | 今の画面を閉じる       | `parent.dismiss()`                         |
親画面で、@Stateを使って写真を保存します。
↓
ドルマーク（$）を付けて、CameraViewに渡します。
↓
子画面では、@Bindingを使って受け取ります。
↓
子画面で撮影した写真を保存します。
↓
親画面の写真も同時に更新されます。

## 自分の実験メモ

**実験1：写真の大きさを変えた**

* やったこと：
  `.frame(maxHeight: 400)`を`.frame(maxHeight: 200)`に変えた。

* 結果：
  写真が小さく表示された。

* わかったこと：
  `frame`の数字を変えると、写真の大きさを変えられる。

**実験2：写真の角をもっと丸くした**

* やったこと：
  `cornerRadius: 16`を`cornerRadius: 40`に変えた。

* 結果：
  写真の角がもっと丸くなった。

* わかったこと：
  数字を大きくすると、角がもっと丸くなる。


## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：`selectedItem`と`selectedImage`の違いは何ですか。**

   **得られた理解：**
   `selectedItem`はライブラリで選んだ写真の情報である。`selectedImage`は実際に画面に表示する写真である。

2. **質問：なぜ`UIViewControllerRepresentable`を使いますか。**

   **得られた理解：**
   カメラはUIKitの機能なので、SwiftUIで使うために必要である。

3. **質問：なぜ`async`と`await`を使いますか。**

   **得られた理解：**
   写真の読み込みには少し時間がかかるため、画面を止めずに処理するために使う。

## この章のまとめ

この章では、写真ライブラリから写真を選ぶ方法を学んだ。

また、カメラで写真を撮って、画面に表示する方法も学んだ。

写真を選んだあとは、その写真を読み込んで、SwiftUIの`Image`に変える必要がある。

カメラはUIKitの機能なので、SwiftUIで使うために`UIViewControllerRepresentable`を使う。

シミュレータではカメラが使えないため、カメラ機能は実際のiPhoneで確認する必要がある。
