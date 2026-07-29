# 第3章：写真を選択・撮影して表示するアプリ

> 執筆者：林 楽駿
> 最終更新：2026-07-29

## この章で学ぶこと

この章では、`PhotosPicker`を使ってiPhoneのフォトライブラリから写真を選択し、SwiftUIの画面に表示する方法を学ぶ。また、UIKitの`UIImagePickerController`をSwiftUIから利用し、カメラで撮影した写真を表示する方法についても学ぶ。

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

// MARK: - メインビュー

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

このアプリは、フォトライブラリから写真を選択したり、iPhoneのカメラで新しい写真を撮影したりして、その写真を画面上に表示するアプリである。

「ライブラリ」ボタンを押すと写真選択画面が表示され、選んだ写真がアプリ画面に表示される。「カメラ」ボタンを押すとカメラが起動し、撮影した写真がアプリ画面に表示される。

写真を選択していない場合は、灰色の表示エリアと「写真を選択または撮影してください」という案内メッセージが表示される。

## コードの詳細解説

### 状態変数の定義

```swift
@State private var selectedItem: PhotosPickerItem?
@State private var selectedImage: Image?
@State private var isShowingCamera = false
@State private var capturedUIImage: UIImage?
```

**何をしているか：**

アプリの画面で使用する写真や、カメラ画面の表示状態を保存している。

`selectedItem`には、フォトライブラリで選択した項目が入る。

`selectedImage`には、実際に画面へ表示するSwiftUIの`Image`が入る。

`isShowingCamera`は、カメラ画面を表示するかどうかを管理する。

`capturedUIImage`には、カメラで撮影した`UIImage`が入る。

**なぜこう書くのか：**

`@State`を使用すると、値が変更されたときにSwiftUIの画面が自動的に再描画されるためである。

例えば、`selectedImage`に新しい写真が設定されると、画像表示エリアが自動的に更新される。

また、写真がまだ選ばれていない状態もあるため、画像に関係する変数はオプショナル型として定義されている。

**もしこう書かなかったら：**

`@State`を付けなかった場合、値を変更しても画面が正しく更新されない可能性がある。

また、オプショナル型にしなかった場合は、最初から必ず画像を設定する必要があり、写真が選ばれていない状態を表現しにくくなる。

---

### フォトライブラリから写真を選択する処理

```swift
PhotosPicker(
    selection: $selectedItem,
    matching: .images
) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
.buttonStyle(.bordered)
```

**何をしているか：**

`PhotosPicker`を使って、ユーザーがフォトライブラリから写真を選択できるボタンを作成している。

`selection`には、選択結果を保存する`selectedItem`をバインディングしている。

`matching: .images`を指定することで、フォトライブラリの中から画像だけを選択できるようにしている。

**なぜこう書くのか：**

`PhotosPicker`はSwiftUIで利用できる写真選択用の部品であり、比較的少ないコードでフォトライブラリを表示できるためである。

`$selectedItem`のように先頭に`$`を付けることで、`PhotosPicker`と状態変数を双方向に接続できる。

**もしこう書かなかったら：**

`selection`を指定しなかった場合、ユーザーが選んだ写真をアプリ側で受け取れない。

また、`matching: .images`を指定しなかった場合、画像以外のデータも選択対象になる可能性がある。

---

### カメラボタンの処理

```swift
Button {
    isShowingCamera = true
} label: {
    Label("カメラ", systemImage: "camera")
}
.buttonStyle(.borderedProminent)
.disabled(!UIImagePickerController.isSourceTypeAvailable(.camera))
```

**何をしているか：**

「カメラ」ボタンが押されたときに、`isShowingCamera`を`true`に変更してカメラ画面を表示する。

また、端末でカメラが利用できない場合は、ボタンを自動的に無効化している。

**なぜこう書くのか：**

iOSシミュレータには通常カメラ機能がないため、カメラが利用可能かどうかを事前に確認する必要がある。

`UIImagePickerController.isSourceTypeAvailable(.camera)`を使うことで、現在の端末でカメラが利用できるか確認できる。

**もしこう書かなかったら：**

カメラが利用できない環境でもボタンを押せるようになり、カメラ画面を開こうとしたときにアプリが正しく動作しない可能性がある。

シミュレータで実行した場合も、カメラボタンが有効なままになってしまう。

---

### 写真選択の変化を監視する処理

```swift
.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
    }
}
```

**何をしているか：**

`selectedItem`の値が変更されたことを監視している。

ユーザーが新しい写真を選択したときに、`loadImage`関数を呼び出して画像データを読み込む。

**なぜこう書くのか：**

写真を選んだだけでは、`PhotosPickerItem`が保存されるだけであり、そのままではSwiftUIの`Image`として画面に表示できない。

そのため、選択内容が変化したタイミングで画像データを読み込む必要がある。

`loadImage`関数は非同期処理なので、`Task`と`await`を使用して実行している。

**もしこう書かなかったら：**

フォトライブラリで写真を選択しても、画像を読み込む処理が実行されないため、画面上の写真は変化しない。

---

### 画像表示エリア

```swift
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
```

**何をしているか：**

画像が選択されている場合と、まだ選択されていない場合で、表示内容を切り替えている。

画像がある場合は、その画像を画面サイズに合わせて表示する。

画像がない場合は、灰色の角丸四角形、写真アイコン、案内メッセージを表示する。

**なぜこう書くのか：**

`selectedImage`はオプショナル型なので、`if let`を使って画像が存在するかを安全に確認している。

また、`@ViewBuilder`を使用することで、条件によって異なるViewを返せるようにしている。

`.aspectRatio(contentMode: .fit)`を使用すると、画像の縦横比を保ったまま表示できる。

**もしこう書かなかったら：**

画像が存在しない状態で強制的に画像を使おうとすると、エラーやクラッシュの原因になる。

`.resizable()`を書かなかった場合は、画像サイズを自由に変更できない。

`.aspectRatio(contentMode: .fit)`を書かなかった場合は、画像が引き伸ばされ、縦横比が崩れる可能性がある。

---

### 選択した画像を読み込む処理

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

フォトライブラリで選択された`PhotosPickerItem`から画像データを読み込み、`UIImage`へ変換したあと、SwiftUIの`Image`へ変換している。

最終的に、変換した画像を`selectedImage`へ保存している。

**なぜこう書くのか：**

`PhotosPickerItem`は、そのままではSwiftUIの画像として表示できないため、最初に`Data`として読み込む必要がある。

その後、`Data`から`UIImage`を作成し、さらにSwiftUIの`Image`へ変換している。

画像の読み込みは時間がかかる可能性があるため、`async`、`await`を使用した非同期処理になっている。

**もしこう書かなかったら：**

`PhotosPickerItem`から画像データを取り出せないため、選択した画像を画面に表示できない。

また、`do-catch`を書かなかった場合は、画像の読み込みに失敗したときのエラーを適切に処理できない。

---

### カメラ画面の表示

```swift
.fullScreenCover(isPresented: $isShowingCamera) {
    CameraView(capturedImage: $capturedUIImage)
}
```

**何をしているか：**

`isShowingCamera`が`true`になったときに、`CameraView`を全画面で表示している。

撮影した画像は、`capturedUIImage`を通して`ContentView`へ渡される。

**なぜこう書くのか：**

カメラは画面全体を使って撮影することが多いため、`fullScreenCover`を使用して全画面表示している。

`$capturedUIImage`を渡すことで、子画面の`CameraView`から親画面の`ContentView`へ撮影結果を反映できる。

**もしこう書かなかったら：**

`isShowingCamera`を変更しても、カメラ画面が表示されない。

また、バインディングを使わなかった場合は、撮影した画像を親画面へ返せなくなる。

---

### 撮影した画像を表示する処理

```swift
.onChange(of: capturedUIImage) { _, newImage in
    if let uiImage = newImage {
        selectedImage = Image(uiImage: uiImage)
    }
}
```

**何をしているか：**

カメラで撮影した画像が`capturedUIImage`に保存されたことを検知している。

新しい`UIImage`が存在する場合、それをSwiftUIの`Image`に変換し、画面表示用の`selectedImage`に保存している。

**なぜこう書くのか：**

カメラから返される画像は`UIImage`型だが、画面で表示している画像はSwiftUIの`Image`型であるため、型を変換する必要がある。

**もしこう書かなかったら：**

撮影自体は成功しても、撮影した画像が`selectedImage`へ反映されないため、画面上に表示されない。

---

### UIKitとSwiftUIの連携

```swift
struct CameraView: UIViewControllerRepresentable {
    @Binding var capturedImage: UIImage?
    @Environment(\.dismiss) private var dismiss
}
```

**何をしているか：**

UIKitの`UIImagePickerController`をSwiftUIの画面から使えるようにするため、`UIViewControllerRepresentable`を使用している。

`capturedImage`は、撮影した写真を親画面へ渡すためのバインディングである。

`dismiss`は、カメラ画面を閉じるために使用する。

**なぜこう書くのか：**

`UIImagePickerController`はUIKitの画面部品なので、そのままではSwiftUIのViewとして利用できない。

そのため、`UIViewControllerRepresentable`を使ってSwiftUI用に包む必要がある。

**もしこう書かなかったら：**

SwiftUIの画面から`UIImagePickerController`を直接表示できないため、カメラ機能を利用できない。

---

### UIImagePickerControllerの作成

```swift
func makeUIViewController(context: Context) -> UIImagePickerController {
    let picker = UIImagePickerController()
    picker.sourceType = .camera
    picker.delegate = context.coordinator
    return picker
}
```

**何をしているか：**

カメラ画面として使用する`UIImagePickerController`を作成している。

`sourceType`を`.camera`に設定し、カメラを起動するようにしている。

また、撮影完了やキャンセルの処理を受け取るために、`delegate`へCoordinatorを設定している。

**なぜこう書くのか：**

`UIImagePickerController`は、写真ライブラリやカメラを表示できるUIKitの部品である。

今回はカメラを利用するため、`.camera`を指定している。

撮影結果を受け取るには、デリゲートの設定が必要である。

**もしこう書かなかったら：**

`sourceType`を指定しなかった場合、意図したカメラ画面が表示されない可能性がある。

`delegate`を設定しなかった場合、撮影完了やキャンセルを検知できず、画像を受け取れない。

---

### Coordinatorの作成

```swift
func makeCoordinator() -> Coordinator {
    Coordinator(self)
}
```

```swift
class Coordinator: NSObject,
                   UIImagePickerControllerDelegate,
                   UINavigationControllerDelegate {
    let parent: CameraView

    init(_ parent: CameraView) {
        self.parent = parent
    }
}
```

**何をしているか：**

`UIImagePickerController`の撮影完了やキャンセルなどのイベントを受け取るCoordinatorを作成している。

Coordinatorは、UIKitのデリゲート処理とSwiftUIの`CameraView`をつなぐ役割を持つ。

**なぜこう書くのか：**

UIKitでは、操作結果をデリゲートメソッドで受け取る仕組みが多く使われている。

SwiftUIのView構造体は直接UIKitのデリゲートとして利用しにくいため、Coordinatorクラスを間に入れて処理する。

**もしこう書かなかったら：**

撮影ボタンを押した後の画像取得処理や、キャンセル処理を受け取れなくなる。

---

### 撮影完了時の処理

```swift
func imagePickerController(
    _ picker: UIImagePickerController,
    didFinishPickingMediaWithInfo info: [UIImagePickerController.InfoKey: Any]
) {
    if let image = info[.originalImage] as? UIImage {
        parent.capturedImage = image
    }

    parent.dismiss()
}
```

**何をしているか：**

ユーザーが写真を撮影したときに呼ばれる処理である。

撮影結果の`info`から元画像を取り出し、`capturedImage`へ保存している。

その後、カメラ画面を閉じている。

**なぜこう書くのか：**

撮影結果には複数の情報が含まれているため、`info[.originalImage]`を使って元画像を取得する必要がある。

取得した画像を親画面へ渡すことで、撮影した写真を画面に表示できる。

**もしこう書かなかったら：**

写真を撮影しても画像を保存できず、アプリ画面へ表示されない。

また、`dismiss()`を書かなかった場合は、撮影後もカメラ画面が閉じない。

---

### 撮影キャンセル時の処理

```swift
func imagePickerControllerDidCancel(
    _ picker: UIImagePickerController
) {
    parent.dismiss()
}
```

**何をしているか：**

ユーザーが撮影をキャンセルしたときに、カメラ画面を閉じて元の画面へ戻している。

**なぜこう書くのか：**

キャンセルした場合は画像が作成されないため、画像を保存せずに画面だけを閉じる必要がある。

**もしこう書かなかったら：**

ユーザーがキャンセルボタンを押してもカメラ画面を閉じられず、元の画面へ戻れなくなる可能性がある。

## 新しく学んだSwiftの文法・API

| 項目                              | 説明                                | 使用例                                                         |
| ------------------------------- | --------------------------------- | ----------------------------------------------------------- |
| `@State`                        | View内部の状態を保存し、値の変更時に画面を更新する       | `@State private var selectedImage: Image?`                  |
| `@Binding`                      | 親画面と子画面で同じ値を共有する                  | `@Binding var capturedImage: UIImage?`                      |
| `PhotosPicker`                  | フォトライブラリから写真を選択する                 | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| `PhotosPickerItem`              | フォトライブラリで選択した項目を表す                | `@State private var selectedItem: PhotosPickerItem?`        |
| `onChange`                      | 指定した値の変化を監視する                     | `.onChange(of: selectedItem) { ... }`                       |
| `Task`                          | 非同期処理を開始する                        | `Task { await loadImage(from: newItem) }`                   |
| `async / await`                 | 時間のかかる処理を非同期で実行する                 | `try await item.loadTransferable(...)`                      |
| `guard let`                     | 値が存在しない場合に処理を早期終了する               | `guard let item = item else { return }`                     |
| `if let`                        | オプショナル型の値を安全に取り出す                 | `if let image = selectedImage`                              |
| `UIViewControllerRepresentable` | UIKitのViewControllerをSwiftUIで使用する | `struct CameraView: UIViewControllerRepresentable`          |
| `UIImagePickerController`       | カメラや写真ライブラリを表示するUIKitの機能          | `let picker = UIImagePickerController()`                    |
| `Coordinator`                   | UIKitのデリゲート処理とSwiftUIを連携する        | `func makeCoordinator() -> Coordinator`                     |
| `fullScreenCover`               | 別の画面を全画面表示する                      | `.fullScreenCover(isPresented: $isShowingCamera)`           |
| `@Environment(\.dismiss)`       | 現在表示している画面を閉じる                    | `@Environment(\.dismiss) private var dismiss`               |
| `@ViewBuilder`                  | 条件によって異なるViewを作成する                | `@ViewBuilder private var imageDisplayArea: some View`      |

## 自分の実験メモ

**実験1：画像の最大表示サイズを変更した**

* やったこと：
  `.frame(maxHeight: 400)`を`.frame(maxHeight: 200)`に変更した。

* 結果：
  選択した写真が以前より小さく表示された。

* わかったこと：
  `frame`の値を変更することで、画面上の画像サイズを調整できることがわかった。また、`.aspectRatio(contentMode: .fit)`があるため、画像を小さくしても縦横比は維持された。

**実験2：画像の角丸を変更した**

* やったこと：
  `.clipShape(RoundedRectangle(cornerRadius: 16))`の`16`を`40`に変更した。

* 結果：
  写真の四隅が、以前より丸く表示された。

* わかったこと：
  `cornerRadius`の数値を大きくすると、画像の角がより丸くなることがわかった。

**実験3：カメラボタンの無効化処理を確認した**

* やったこと：
  iOSシミュレータでアプリを実行した。

* 結果：
  シミュレータにはカメラがないため、カメラボタンが自動的に無効になった。

* わかったこと：
  `UIImagePickerController.isSourceTypeAvailable(.camera)`を使うことで、端末の機能に応じてボタンを有効または無効にできることがわかった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：`selectedItem`と`selectedImage`は何が違うのか。**

   **得られた理解：**
   `selectedItem`はフォトライブラリで選択した項目を表し、そのままでは画面に表示できない。一方、`selectedImage`はSwiftUIで実際に画面へ表示する画像である。`selectedItem`からデータを読み込み、`selectedImage`へ変換する必要がある。

2. **質問：なぜカメラ機能では`UIViewControllerRepresentable`が必要なのか。**

   **得られた理解：**
   カメラで利用している`UIImagePickerController`はUIKitの機能であり、そのままではSwiftUIのViewとして使えない。そのため、`UIViewControllerRepresentable`を使ってSwiftUIで利用できる形に変換する必要がある。

3. **質問：なぜ画像の読み込みに`async`と`await`を使うのか。**

   **得られた理解：**
   写真データの読み込みには時間がかかる場合がある。`async`と`await`を使うことで、画面の処理を止めずに画像を読み込める。また、読み込みが完了した後に安全に画像を表示できる。

## この章のまとめ

この章では、`PhotosPicker`を使ってフォトライブラリから写真を選択し、画面に表示する方法を学んだ。

写真を選択した後は、`PhotosPickerItem`から画像データを非同期で読み込み、`UIImage`を経由してSwiftUIの`Image`に変換する必要があることがわかった。

また、カメラ機能ではUIKitの`UIImagePickerController`を使用するため、`UIViewControllerRepresentable`とCoordinatorを利用してSwiftUIとUIKitを連携する必要がある。

さらに、`@State`、`@Binding`、`onChange`を使うことで、写真の選択や撮影によるデータの変化を監視し、画面を自動的に更新できることを理解した。

シミュレータではカメラが利用できないため、カメラ機能は実機で確認する必要がある。また、実機でカメラを使用する場合は、`Info.plist`に`NSCameraUsageDescription`を追加する必要がある。
