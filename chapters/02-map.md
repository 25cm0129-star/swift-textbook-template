# 第2章：地図アプリの基本

> 執筆者：林楽駿
> 最終更新：2026/5/22

## この章で学ぶこと

この章では、MapKitを使ってアプリ内に地図を表示し、キーワード検索で特定の場所にマーカーを表示する方法を学ぶ。具体的には、テキストフィールドに入力したキーワードをもとに地図上の位置を検索し、その場所にピンを立てて地図を移動させるアプリを題材にする。

## 模範コードの全体像

// ContentView.swift
import SwiftUI

struct ContentView: View {
    @State var inputText: String = ""
    @State var displaySearchKey: String = "東京駅"

    var body: some View {
        VStack {
            TextField("キーワード", text: $inputText, prompt: Text("キーワードを入力してください"))
                .onSubmit {
                    displaySearchKey = inputText
                }
                .padding()
            MapView(searchKey: displaySearchKey)
        }
    }
}

// MapView.swift
import SwiftUI
import MapKit

struct MapView: View {
    let searchKey: String
    @State var targetCoordinate = CLLocationCoordinate2D()
    @State var cameraPosition: MapCameraPosition = .automatic

    var body: some View {
        Map(position: $cameraPosition){
            Marker(searchKey, coordinate: targetCoordinate)
        }
        .onChange(of: searchKey, initial: true) { oldValue, newValue in
            let request = MKLocalSearch.Request()
            request.naturalLanguageQuery = newValue
            let search = MKLocalSearch(request: request)

            search.start { response, error in
                if let mapItems = response?.mapItems,
                   let mapItem = mapItems.first {
                    targetCoordinate = mapItem.placemark.coordinate
                    cameraPosition = .region(MKCoordinateRegion(
                        center: targetCoordinate,
                        latitudinalMeters: 500.0,
                        longitudinalMeters: 500.0
                    ))
                }
            }
        }
    }
}
```

**このアプリは何をするものか：**


テキストフィールドにキーワード（例：「東京駅」）を入力してReturnキーを押すと、その場所を地図上で検索し、ピン（マーカー）を立てて地図が自動的にその位置に移動するアプリ。起動時はデフォルトで「東京駅」が表示される。
## コードの詳細解説

### データモデル（ランドマーク構造体）

@State var inputText: String = ""
@State var displaySearchKey: String = "東京駅"

**何をしているか：**
ユーザーがテキストフィールドに入力中の文字列をinputTextに保持し、Returnキーを押したときにdisplaySearchKeyに反映させる。displaySearchKeyが変わると地図の検索が走る仕組みになっている。

**なぜこう書くのか：**

2つの変数を分けることで、入力中の文字がリアルタイムで検索に反映されるのを防ぐことができる。Returnキーを押したときだけ検索が実行される。

**もしこう書かなかったら：**

inputTextを直接MapViewに渡すと、1文字入力するたびに検索が走り、API呼び出しが大量に発生してしまう。

---

### 地図の表示とカメラ制御

@State var cameraPosition: MapCameraPosition = .automatic

cameraPosition = .region(MKCoordinateRegion(
    center: targetCoordinate,
    latitudinalMeters: 500.0,
    longitudinalMeters: 500.0
))
**何をしているか：**

cameraPositionで地図の表示位置を管理している。検索結果の緯度経度を中心に、500m四方の範囲を表示するよう地図を移動させている。

**なぜこう書くのか：**
MapCameraPositionを使うことで、地図の表示範囲をコードから制御できる。latitudinalMetersとlongitudinalMetersで表示する範囲の広さを指定できる。


**もしこう書かなかったら：**

.automaticのままだと地図が自動的に最適な位置を決めるが、検索結果の場所に移動しない。

---

### マーカーの表示
Map(position: $cameraPosition){
    Marker(searchKey, coordinate: targetCoordinate)
}

**何をしているか：**

地図上にtargetCoordinateの位置にピンを表示し、ラベルとしてsearchKey（検索キーワード）を表示している。

**なぜこう書くのか：**

Mapのクロージャ内にMarkerを書くだけで地図上にピンを表示できる。SwiftUIのMapKitは宣言的に書けるため、コードが短くシンプルになる。


**もしこう書かなかったら：**

Markerを書かなければピンが表示されず、どこを指しているのかユーザーに伝わらない。

---

### フィルター機能

.onChange(of: searchKey, initial: true) { oldValue, newValue in
    let request = MKLocalSearch.Request()
    request.naturalLanguageQuery = newValue
    let search = MKLocalSearch(request: request)

    search.start { response, error in
        if let mapItems = response?.mapItems,
           let mapItem = mapItems.first {
            targetCoordinate = mapItem.placemark.coordinate
        }
    }
}
**何をしているか：**

searchKeyが変わったときにMKLocalSearchでキーワード検索を実行し、最初の検索結果の緯度経度をtargetCoordinateに保存している。initial: trueによってビュー表示時にも最初から検索が実行される。

**なぜこう書くのか：**

MKLocalSearchを使うことで、住所や施設名などの自然言語で場所を検索できる。緯度経度を自分で調べなくてもキーワードだけで地図を動かせる。

**もしこう書かなかったら：**
initial: trueがないと、最初に表示したときに検索が実行されず、地図が初期位置のままになってしまう。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Map` | SwiftUIで地図を表示するビューコンポーネント | `Map(position: .constant(.region(region)))` |
| 例：`Marker` | 地図上に位置をマーキングするコンポーネント | `Marker("名前", coordinate: coordinate)` |
|MKLocalSearch | キーワードで場所を検索するMapKitのクラス|let search = MKLocalSearch(request: request) |
|MapCameraPosition |地図の表示位置・範囲を管理する型 |cameraPosition = .region(...) |
|.onChange(of:initial:) |値の変化を検知して処理を実行するモディファイア |.onChange(of: searchKey, initial: true) |

## 自分の実験メモ
実験1：
cameraPosition = .region(MKCoordinateRegion(
    center: targetCoordinate,
    latitudinalMeters: 5000.0,
    longitudinalMeters: 5000.0
))
実験２
if let mapItems = response?.mapItems,
   let mapItem = mapItems.last {
    targetCoordinate = mapItem.placemark.coordinate

**実験1：**
実験1：

やったこと：latitudinalMetersとlongitudinalMetersを500.0から5000.0に変更してみた
結果：地図の表示範囲が広くなり、より広いエリアが表示された
わかったこと：この数値を変えることで地図のズームレベルを調整できる


実験２：

やったこと：latitudinalMetersとlongitudinalMetersを500.0から5000.0に変更してみた
結果：地図の表示範囲が広くなり、より広いエリアが表示された
わかったこと：この数値を変えることで地図のズームレベルを調整できる



## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：**
  質問1： onChange(of:initial:)のinitial: trueは何のためにあるか？
得られた理解： initial: trueをつけることで、ビューが最初に表示されたときにもonChangeの処理が実行される。つけないと、最初のキーワード「東京駅」での検索が実行されず、地図が初期位置のままになってしまう。

2. **質問：**
  質問2： MKLocalSearchと直接緯度経度を指定する方法の違いは何か？
得られた理解： 緯度経度を直接指定する場合は事前に座標を調べる必要があるが、MKLocalSearchを使えばキーワードだけで検索できる。ユーザーが自由に場所を入力できるアプリにはMKLocalSearchが適している。
3. **質問：**
 質問3： $cameraPositionの$は何を意味するか？
得られた理解： $はバインディングを意味する。MapがcameraPositionを読み取るだけでなく、地図をドラッグしたときにcameraPositionの値を更新できるようにするために必要。$なしだと地図の位置をコードから変えられない。

## この章のまとめ

この章では、MapKitを使ってキーワードで場所を検索し、地図上にマーカーを表示する方法を学んだ。MKLocalSearchを使えば住所や施設名などの自然言語で場所を検索でき、MapCameraPositionで地図の表示範囲をコントロールできることがわかった。また、onChange(of:initial:true)を使うことでキーワードの変化を検知して自動的に地図を更新できる仕組みも理解できた。
