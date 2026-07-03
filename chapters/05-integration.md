# 第5章：機能統合の実践

> 執筆者：林楽駿
> 最終更新：2026-06-17

## この章で学ぶこと

この章では、これまで学んだ写真選択、位置情報取得、地図表示、SwiftDataによるデータ保存を組み合わせて、フォトマップアプリを作成する方法を学ぶ。写真と位置情報を同時に保存し、地図や一覧画面で管理することで、複数機能を統合した実践的なアプリ開発の流れを理解する。

（教員から配布された模範コードをここに貼り付ける）

// ============================================
// 第5章：写真 + 地図 + データ保存の統合アプリ
// ============================================
// 写真を選択し、選択時の現在地を地図上に記録する
// 「フォトマップ」アプリです。
// 第2〜4章で学んだ技術を組み合わせて使います。
//
// 【注意】Info.plist に以下のキーを追加してください：
//   - NSLocationWhenInUseUsageDescription
//   - NSPhotoLibraryAddUsageDescription
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


**このアプリは何をするものか：**

このアプリは、写真を選択した場所の位置情報を記録し、地図上に表示できるフォトマップアプリである。

ユーザーは写真を選択し、タイトルやメモを入力して保存できる。保存されたデータはSwiftDataによって端末内に永続化されるため、アプリを終了しても消えない。

地図画面では保存した写真がマーカーとして表示され、一覧画面では保存順に記録を確認できる。マーカーや一覧を選択すると詳細画面が表示され、写真・メモ・撮影場所を確認できる。

## コードの詳細解説

### データモデルの設計

```swift
@Model
class Spot {
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
```

**何をしているか：**
この部分では、保存する場所のデータモデルを作っています。場所のタイトル、メモ、緯度、経度、画像データ、作成日時を保存できるようにしています。
また、`coordinate` では緯度と経度から地図で使える座標を作り、`uiImage` では保存した画像データを画面に表示できる画像に変換しています。

**なぜこう書くのか：**
SwiftDataで保存するために、クラスに `@Model` を付けています。
緯度と経度は `CLLocationCoordinate2D` のまま保存するのではなく、`Double` 型で分けて保存しています。その理由は、SwiftDataではシンプルな型のほうが保存しやすいからです。
画像もそのまま `UIImage` として保存するのではなく、`Data` 型にして保存しています。これにより、SwiftDataで扱いやすくなります。

**もしこう書かなかったら：**
もし `@Model` を付けなかったら、SwiftDataでこのデータを保存できません。
また、緯度と経度を保存していない場合、地図上にピンを表示することができません。
画像を `Data` に変換せずに `UIImage` のまま保存しようとすると、SwiftDataでうまく保存できない可能性があります。

---

### タブ構成の設計

```swift
TabView {
    SpotListView()
        .tabItem {
            Label("一覧", systemImage: "list.bullet")
        }

    MapView()
        .tabItem {
            Label("地図", systemImage: "map")
        }

    AddSpotView()
        .tabItem {
            Label("追加", systemImage: "plus.circle")
        }
}
```

**何をしているか：**
この部分では、アプリの画面をタブで切り替えられるようにしています。
「一覧」では登録した場所を見ることができ、「地図」では場所を地図上で確認できます。「追加」では新しい場所を登録できます。

**なぜこう書くのか：**
機能ごとに画面を分けることで、ユーザーが使いやすくなるからです。
すべての機能を一つの画面に入れると、画面が見にくくなります。タブを使うことで、目的の画面にすぐ移動できます。

**もしこう書かなかったら：**
タブ構成にしなかった場合、画面遷移のボタンをたくさん作る必要があります。
また、一覧画面、地図画面、追加画面の切り替えが分かりにくくなります。
そのため、アプリ全体の操作性が悪くなると思います。

---

### カメラと位置情報の連携

```swift
PhotosPicker(selection: $selectedItem, matching: .images) {
    Text("写真を選択")
}

.onChange(of: selectedItem) {
    Task {
        if let data = try? await selectedItem?.loadTransferable(type: Data.self) {
            imageData = data
        }
    }
}

func getCurrentLocation() {
    if let location = locationManager.location {
        latitude = location.coordinate.latitude
        longitude = location.coordinate.longitude
    }
}
```

**何をしているか：**
この部分では、写真の選択と現在地の取得を行っています。
`PhotosPicker` を使ってスマートフォンの写真から画像を選び、その画像を `Data` 型として取得しています。
また、位置情報から現在の緯度と経度を取得し、場所データとして保存できるようにしています。

**なぜこう書くのか：**
写真は画面に表示するだけでなく、SwiftDataに保存する必要があるため、`Data` 型に変換しています。
位置情報は地図で使うために、緯度と経度を別々に取得しています。
このようにすることで、写真付きの場所データを登録できるようになります。

**もしこう書かなかったら：**
写真を `Data` として取得しなかった場合、画像を保存することができません。
また、位置情報を取得しなかった場合、登録した場所を地図上に表示できません。
その場合、ただのメモアプリのようになってしまい、地図アプリとしての機能が弱くなります。

---

### SwiftDataでの画像保存

```swift
let newSpot = Spot(
    title: title,
    memo: memo,
    latitude: latitude,
    longitude: longitude,
    imageData: imageData
)

modelContext.insert(newSpot)

do {
    try modelContext.save()
} catch {
    print("保存に失敗しました")
}
```

**何をしているか：**
この部分では、入力されたタイトル、メモ、緯度、経度、画像データを使って、新しい場所データを作成しています。
そのあと、`modelContext.insert()` を使ってSwiftDataに追加し、`modelContext.save()` で保存しています。

**なぜこう書くのか：**
SwiftDataでは、保存したいデータをまずモデルのインスタンスとして作ります。
そして、`modelContext` に追加することで、アプリのデータとして管理できます。
画像は `UIImage` ではなく `Data` として保存しているため、あとで取り出して再び画像として表示できます。

**もしこう書かなかったら：**
`modelContext.insert()` を書かなかった場合、新しいデータはSwiftDataに追加されません。
また、`save()` をしなかった場合、アプリを閉じたあとにデータが残らない可能性があります。
画像データを保存していない場合、一覧や詳細画面で写真を表示することができません。


---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目                       | 説明                                | 使用例                                                                |
| ------------------------ | --------------------------------- | ------------------------------------------------------------------ |
| `TabView`                | 複数の画面をタブで切り替えるためのコンポーネント          | `TabView { SpotListView(); MapView() }`                            |
| `CLLocationManager`      | GPSを使って現在地の情報を取得するAPI             | `let location = locationManager.location`                          |
| `PhotosPicker`           | 端末の写真ライブラリから画像を選択するためのAPI         | `PhotosPicker(selection: $selectedItem, matching: .images)`        |
| `@Model`                 | SwiftDataで保存するデータモデルを定義するための記述    | `@Model class Spot { ... }`                                        |
| `modelContext.insert()`  | SwiftDataに新しいデータを追加する処理           | `modelContext.insert(newSpot)`                                     |
| `Data?`                  | 画像などのデータを保存するために使う型。値がない場合はnilになる | `var imageData: Data?`                                             |
| `guard let`              | nilではないか確認して、安全に値を取り出す文法          | `guard let data = imageData else { return nil }`                   |
| `CLLocationCoordinate2D` | 地図で使う緯度・経度の座標を表す型                 | `CLLocationCoordinate2D(latitude: latitude, longitude: longitude)` |

## 自分の実験メモ

**実験1：**

* やったこと：
  `TabView` の中に「一覧」「地図」「追加」の3つの画面を入れて、タブで切り替えられるようにした。
* 結果：
  画面下のタブを押すと、それぞれの画面に移動できた。
* わかったこと：
  `TabView` を使うと、複数の機能を分かりやすく分けられることがわかった。画面遷移のコードをたくさん書かなくても、簡単に画面を切り替えられる。

**実験2：**

* やったこと：
  写真を選択して、`Data` 型として保存し、そのあと `UIImage` に戻して画面に表示した。
* 結果：
  選択した写真を一覧画面や詳細画面で表示できた。
* わかったこと：
  SwiftDataでは `UIImage` をそのまま保存するより、`Data` に変換して保存したほうが扱いやすいことがわかった。また、表示するときは `UIImage(data:)` を使って戻す必要がある。

**実験3：**

* やったこと：
  現在地の緯度と経度を取得して、その値を使って地図上にピンを表示した。
* 結果：
  登録した場所が地図上に表示された。
* わかったこと：
  地図で場所を表示するには、緯度と経度のデータが必要だとわかった。また、保存するときは `Double` 型で保存し、使うときに `CLLocationCoordinate2D` に変換すると便利だとわかった。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
   なぜ緯度と経度を `CLLocationCoordinate2D` のまま保存しないで、`Double` 型で別々に保存するのか。
   **得られた理解：**
   SwiftDataではシンプルな型のほうが保存しやすいため、緯度と経度を `Double` 型で保存することがわかった。そして、地図で使うときだけ `CLLocationCoordinate2D` に変換すればよいと理解した。

2. **質問：**
   なぜ画像を `UIImage` ではなく `Data` 型で保存するのか。
   **得られた理解：**
   `UIImage` は画面表示用の型で、保存用には向いていないことがわかった。画像を保存する場合は `Data` 型に変換し、表示するときに `UIImage(data:)` で戻す必要があると理解した。

3. **質問：**
   `guard let` は何のために使うのか。
   **得られた理解：**
   `imageData` のようにnilになる可能性がある値を、安全に取り出すために使う文法だとわかった。値がない場合は早めに処理を終了できるので、エラーを防ぎやすくなると理解した。


## この章のまとめ

この章で一番重要だと思ったことは、アプリのデータをただ画面に表示するだけでなく、「どのような形で保存し、どのように取り出して使うか」を考えることです。

今回の制作では、場所のタイトルやメモだけでなく、緯度・経度、画像データも一緒に保存しました。特に、地図で使う座標は `CLLocationCoordinate2D` のまま保存するのではなく、`Double` 型の緯度と経度に分けて保存し、使うときに座標へ変換する方法を学びました。また、画像は `UIImage` のまま保存するのではなく、`Data` 型に変換して保存する必要があることも理解できました。

未来の自分が同じようなアプリを作るときは、まず「保存しやすい形」と「画面で使いやすい形」は違う場合があることを思い出したいです。そして、SwiftData、地図、画像、位置情報を組み合わせるときは、それぞれの型の役割を確認しながら作ることが大切だと思いました。

