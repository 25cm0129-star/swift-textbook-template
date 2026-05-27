# 第3章：カメラの利用

> 執筆者：林楽駿
> 最終更新：2026/5/27

## この章で学ぶこと

このアプリは、フォトライブラリから写真を選ぶか、カメラで撮影して、その画像を画面に表示するだけのシンプルなアプリ。「ライブラリ」ボタンを押すと写真の一覧が出てきて、選んだ写真が大きく表示される。「カメラ」ボタンはシミュレータでは自動的に無効になり、実機でのみ撮影できる。
## 模範コードの全体像



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
                // 画像表示エリア
                imageDisplayArea

                // ボタンエリア
                HStack(spacing: 20) {
                    // フォトライブラリから選択
                    PhotosPicker(selection: $selectedItem, matching: .images) {
                        Label("ライブラリ", systemImage: "photo.on.rectangle")
                    }
                    .buttonStyle(.bordered)

                    // カメラで撮影（シミュレータには未搭載のため自動的に無効化）
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

**このアプリは何をするものか：**

「ライブラリ」ボタンを押すとフォトライブラリが開いて写真を選べる。「カメラ」ボタンを押すとカメラが起動して撮影できる。選択・撮影した写真は画面中央に大きく表示される。カメラはシミュレータでは使えないため、ボタンが自動的にグレーアウトされる。

## コードの詳細解説

### PhotosPickerによる写真選択

PhotosPicker(selection: $selectedItem, matching: .images) {
    Label("ライブラリ", systemImage: "photo.on.rectangle")
}
.buttonStyle(.bordered)

何をしているか：
ボタンとして表示され、タップするとiOS標準のフォトライブラリUIが開く。ユーザーが写真を選ぶと、その結果が selectedItem（PhotosPickerItem?型）に自動で入る。matching: .images は動画を除外して画像だけを表示するフィルター。
なぜこう書くのか：
PhotosPicker はSwiftUIネイティブの部品（iOS 16+）なので、UIKitを一切使わずにライブラリ連携が完結する。以前は PHPickerViewController をUIKitからラップする必要があり、コードが数十行規模になっていた。
もしこう書かなかったら：
matching: .images を外すと動画も選択肢に出てくる。その場合、loadTransferable(type: Data.self) で動画を読もうとして失敗し、画像が表示されなくなる。

### 画像の非同期読み込み

.onChange(of: selectedItem) { _, newItem in
    Task {
        await loadImage(from: newItem)
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
**何をしているか：**

selectedItem が変わった瞬間に loadImage を呼び出し、PhotosPickerItem から実際の画像データを取り出している。loadTransferable で生のバイト列（Data）を取得し、UIImage → Image の順に変換して画面に表示できる形にしている。
**なぜこう書くのか：**

PhotosPickerItem は写真の「参照」を持っているだけで、実際のデータはまだ読まれていない。データ取得には時間がかかる可能性があるため async/await で非同期にしている。.onChange のクロージャは同期コンテキストなので、await を直接書けず Task { } で非同期処理を起動している。
**もしこう書かなかったら：**
Task { } を外すとコンパイルエラーになる（同期コンテキストで await は書けない）。guard let item = item else { return } を外すと、写真の選択を解除したとき（selectedItem がnilになったとき）にクラッシュする。
---

### UIViewControllerRepresentableによるカメラ連携

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
}

**何をしているか：**
SwiftUIにはカメラを起動する部品がないため、UIKitの UIImagePickerController を借りてきている。UIViewControllerRepresentable はSwiftUIとUIKitの橋渡しプロトコルで、これに準拠することでSwiftUIのコードの中でUIKitのViewControllerを使えるようになる。makeUIViewController でカメラのインスタンスを作って返すのが主な仕事。
**なぜこう書くのか：**

SwiftUIから直接 UIImagePickerController() をインスタンス化しても表示できない（SwiftUIはUIKitのViewControllerをそのままレンダリングできないため）。UIViewControllerRepresentable を経由することで、SwiftUIのライフサイクルにUIKitを組み込める。

**もしこう書かなかったら：**

このプロトコルを使わずにカメラを実装しようとすると、AVFoundation でカメラセッションをゼロから組む必要があり、コード量が数倍になる。picker.delegate = context.coordinator を忘れると、撮影してもアプリに画像が渡ってこない。
---

### Coordinatorパターン

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

**何をしているか：**
「撮影完了」「キャンセル」などのイベントを受け取る専用クラス。UIKitはデリゲートパターンでイベントを通知してくるので、その受け口になっている。撮影完了時は info[.originalImage] から画像を取り出し parent.capturedImage に渡す。その後 parent.dismiss() でカメラ画面を閉じる。
**なぜこう書くのか：**

UIKitのデリゲートプロトコルはクラスにしか準拠できない（AnyObject 制約がある）。SwiftUIの View は構造体なのでデリゲートを直接受け取れず、クラスである Coordinator を仲介役にするのが公式の解決策。let parent: CameraView で親への参照を持つことで、受け取った画像を親のBindingに書き戻せる。

**もしこう書かなかったら：**

Coordinator がないとデリゲートを設定できず、撮影してもアプリ側に何も通知されない。カメラ画面も自動で閉じられないため、ユーザーがアプリに戻れなくなる。
---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`PhotosPicker` | フォトライブラリから画像を選択するコンポーネント | `PhotosPicker(selection: $selectedItem, matching: .images)` |
| 例：`UIImagePickerController` | カメラまたはフォトライブラリにアクセスするUIKitコンポーネント | `picker.sourceType = .camera` |
| loadTransferable(type:)|PhotosPickerItem から非同期でデータを読み込むメソッド |try await item.loadTransferable(type: Data.self) |
|UIViewControllerRepresentable | UIKitのViewControllerをSwiftUIで使うためのプロトコル|struct CameraView: UIViewControllerRepresentable |
| makeCoordinator()|Coordinatorのインスタンスを生成するメソッド |return Coordinator(self) |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
実験1：撮影後に画像を保存する

やること：UIImageWriteToSavedPhotosAlbum() を使い、撮影した画像をカメラロールに保存するボタンを追加する
学べること：iOSの写真保存APIの使い方。Info.plist に NSPhotoLibraryAddUsageDescription が必要になることも体験できる

**実験2：**
実験2：matching を変えて動画も選べるようにする

やること：matching: .images を matching: .any(of: [.images, .videos]) に変えて、何が起きるか確認する
学べること：PHPickerFilter の仕組みと、動画を選んだときに loadTransferable(type: Data.self) がどう振る舞うか（失敗するはず）

## AIに聞いて特に理解が深まった質問 TOP3

1.「Task { await ... } はなぜ必要なの？await だけじゃダメ？」
得られる理解：.onChange のクロージャは同期コンテキストなので、その中で await を直接書けない。Task を作ることで非同期処理を「起動」している、という構造が分かる。
2.「Coordinator がクラスでないといけない理由は？」
得られる理解：デリゲートプロトコルは AnyObject（クラスのみ）に制約されており、SwiftUIの View（構造体）では準拠できない。クラスと構造体の違いが実際の制約として体感できる。
3.「@ViewBuilder って何？なぜ imageDisplayArea に付いているの？」
得られる理解：if/else で異なる型のビューを返す計算プロパティには @ViewBuilder が必要。これがないとコンパイルエラーになることを試せる。
## この章のまとめ

この章の一番大事なポイントは、SwiftUIとUIKitは別の世界だということ。SwiftUIだけで完結するなら PhotosPicker のように1行で書ける。しかしカメラのようにUIKitにしかないものを使いたいときは、UIViewControllerRepresentable という「橋」を自分で作る必要がある。その橋を渡すときに「デリゲート（イベント通知）をどこで受け取るか」という問題が出てきて、それを解決するのが Coordinator パターン。最初は仰々しく見えるが、役割ごとに分けると「ViewControllerを作る」「イベントを受け取る」「親に結果を渡す」という3つのことをしているだけ。今後UIKitのコンポーネントを使いたくなったときは、毎回この同じパターンを繰り返すことになる。
