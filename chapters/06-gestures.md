# 第6章：ジェスチャー操作

> 執筆者：林 楽駿  
> 最終更新：2026-07-15

## この章で学ぶこと

この章では、SwiftUIでユーザーの指の動きを検出するジェスチャー機能について学ぶ。タップ、ロングプレス、ドラッグ、ピンチによる拡大縮小、2本指による回転を実装し、最後に複数のジェスチャーを同時に使用する方法を確認する。

## 模範コードの全体像

```swift
// ============================================
// 第6章（基本）：ジェスチャーで操作するカードアプリ
// ============================================
// タップ、ロングプレス、ドラッグ、ピンチ、回転の
// 各ジェスチャーを実際に体験しながら学びます。
// ============================================

import SwiftUI

// MARK: - メインビュー

struct ContentView: View {
    var body: some View {
        NavigationStack {
            List {
                NavigationLink("タップ & ロングプレス") {
                    TapDemoView()
                }
                NavigationLink("ドラッグ") {
                    DragDemoView()
                }
                NavigationLink("ピンチ（拡大縮小）") {
                    MagnifyDemoView()
                }
                NavigationLink("回転") {
                    RotateDemoView()
                }
                NavigationLink("組み合わせ") {
                    CombinedDemoView()
                }
            }
            .navigationTitle("ジェスチャー体験")
        }
    }
}

// MARK: - タップ & ロングプレス

struct TapDemoView: View {
    @State private var tapCount = 0
    @State private var backgroundColor: Color = .blue
    @State private var isPressed = false

    var body: some View {
        VStack(spacing: 30) {
            Text("タップ回数: \(tapCount)")
                .font(.title)

            // シングルタップ
            RoundedRectangle(cornerRadius: 16)
                .fill(backgroundColor)
                .frame(width: 200, height: 200)
                .overlay {
                    Text("タップしてね")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .onTapGesture {
                    tapCount += 1
                    backgroundColor = Color(
                        hue: Double.random(in: 0...1),
                        saturation: 0.7,
                        brightness: 0.9
                    )
                }

            // ロングプレス
            Circle()
                .fill(isPressed ? .green : .orange)
                .frame(width: 120, height: 120)
                .scaleEffect(isPressed ? 1.3 : 1.0)
                .overlay {
                    Text(isPressed ? "成功!" : "長押し")
                        .foregroundStyle(.white)
                        .font(.headline)
                }
                .animation(.spring(duration: 0.3), value: isPressed)
                .onLongPressGesture(minimumDuration: 1.0) {
                    isPressed = true
                    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
                        isPressed = false
                    }
                }
        }
        .navigationTitle("タップ & ロングプレス")
    }
}

// MARK: - ドラッグ

struct DragDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero

    var body: some View {
        VStack {
            Text("カードをドラッグしてみよう")
                .font(.headline)
                .padding()

            Spacer()

            RoundedRectangle(cornerRadius: 20)
                .fill(
                    LinearGradient(
                        colors: [.purple, .blue],
                        startPoint: .topLeading,
                        endPoint: .bottomTrailing
                    )
                )
                .frame(width: 200, height: 280)
                .shadow(radius: 8)
                .overlay {
                    VStack {
                        Image(systemName: "hand.draw")
                            .font(.system(size: 40))
                        Text("ドラッグ")
                            .font(.title3)
                    }
                    .foregroundStyle(.white)
                }
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ドラッグ")
    }
}

// MARK: - ピンチ（拡大縮小）

struct MagnifyDemoView: View {
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0

    var body: some View {
        VStack {
            Text("ピンチで拡大縮小")
                .font(.headline)
                .padding()

            Text(String(format: "倍率: %.1fx", scale))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "star.fill")
                .font(.system(size: 100))
                .foregroundStyle(.yellow)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .scaleEffect(scale)
                .gesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    scale = 1.0
                    lastScale = 1.0
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("ピンチ")
    }
}

// MARK: - 回転

struct RotateDemoView: View {
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("2本指で回転")
                .font(.headline)
                .padding()

            Text(String(format: "角度: %.0f°", angle.degrees))
                .font(.caption)
                .foregroundStyle(.secondary)

            Spacer()

            Image(systemName: "arrow.up")
                .font(.system(size: 80))
                .foregroundStyle(.red)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .rotationEffect(angle)
                .gesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("回転")
    }
}

// MARK: - 組み合わせ

struct CombinedDemoView: View {
    @State private var offset: CGSize = .zero
    @State private var lastOffset: CGSize = .zero
    @State private var scale: CGFloat = 1.0
    @State private var lastScale: CGFloat = 1.0
    @State private var angle: Angle = .zero
    @State private var lastAngle: Angle = .zero

    var body: some View {
        VStack {
            Text("ドラッグ・ピンチ・回転を同時に")
                .font(.headline)
                .padding()

            Spacer()

            Image(systemName: "photo.artframe")
                .font(.system(size: 120))
                .foregroundStyle(.indigo)
                // タッチ判定を300×300の透明な領域に広げる
                .frame(width: 300, height: 300)
                .contentShape(Rectangle())
                .scaleEffect(scale)
                .rotationEffect(angle)
                .offset(offset)
                .gesture(
                    DragGesture()
                        .onChanged { value in
                            offset = CGSize(
                                width: lastOffset.width + value.translation.width,
                                height: lastOffset.height + value.translation.height
                            )
                        }
                        .onEnded { _ in
                            lastOffset = offset
                        }
                )
                // 複数のジェスチャーを「同時に」効かせるには
                // .gesture を重ねるのではなく .simultaneousGesture を使う
                .simultaneousGesture(
                    MagnifyGesture()
                        .onChanged { value in
                            scale = lastScale * value.magnification
                        }
                        .onEnded { _ in
                            lastScale = scale
                        }
                )
                .simultaneousGesture(
                    RotateGesture()
                        .onChanged { value in
                            angle = lastAngle + value.rotation
                        }
                        .onEnded { _ in
                            lastAngle = angle
                        }
                )

            Spacer()

            Button("リセット") {
                withAnimation(.spring) {
                    offset = .zero
                    lastOffset = .zero
                    scale = 1.0
                    lastScale = 1.0
                    angle = .zero
                    lastAngle = .zero
                }
            }
            .buttonStyle(.bordered)
            .padding()
        }
        .navigationTitle("組み合わせ")
    }
}

#Preview {
    ContentView()
}
```

**このアプリは何をするものか：**

このアプリは、SwiftUIで使用できる基本的なジェスチャーを実際に操作しながら確認するためのアプリである。

最初の画面には「タップ & ロングプレス」「ドラッグ」「ピンチ（拡大縮小）」「回転」「組み合わせ」の5つの項目が表示される。各項目を選択すると別の画面へ移動し、それぞれのジェスチャーを体験できる。

タップ画面ではタップ回数と図形の色が変化し、長押しすると円の色・大きさ・文字が変化する。ドラッグ画面ではカードを自由に移動でき、ピンチ画面では星を拡大縮小できる。回転画面では矢印を2本指で回転できる。組み合わせ画面では、画像の移動、拡大縮小、回転を同時に行うことができる。

## コードの詳細解説

### 画面遷移

```swift
NavigationStack {
    List {
        NavigationLink("タップ & ロングプレス") {
            TapDemoView()
        }
        NavigationLink("ドラッグ") {
            DragDemoView()
        }
        NavigationLink("ピンチ（拡大縮小）") {
            MagnifyDemoView()
        }
        NavigationLink("回転") {
            RotateDemoView()
        }
        NavigationLink("組み合わせ") {
            CombinedDemoView()
        }
    }
    .navigationTitle("ジェスチャー体験")
}
```

**何をしているか：**

`NavigationStack`の中に`List`を配置し、5種類のジェスチャーを選択できるメニュー画面を作っている。`NavigationLink`をタップすると、それぞれのデモ画面へ移動する。

**なぜこう書くのか：**

ジェスチャーごとに画面を分けることで、各機能の動作やコードを個別に確認しやすくなる。また、`NavigationStack`を使用すると、遷移先の画面から戻る操作も自動的に利用できる。

**もしこう書かなかったら：**

`NavigationStack`がなければ、`NavigationLink`による画面遷移を正しく管理できない。すべての機能を1画面に置くと、表示内容が多くなり、どのジェスチャーを操作しているのか分かりにくくなる。

---

### 基本ジェスチャー（タップ、ロングプレス）

```swift
@State private var tapCount = 0
@State private var backgroundColor: Color = .blue
@State private var isPressed = false
```

```swift
.onTapGesture {
    tapCount += 1
    backgroundColor = Color(
        hue: Double.random(in: 0...1),
        saturation: 0.7,
        brightness: 0.9
    )
}
```

```swift
.animation(.spring(duration: 0.3), value: isPressed)
.onLongPressGesture(minimumDuration: 1.0) {
    isPressed = true
    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
        isPressed = false
    }
}
```

**何をしているか：**

`onTapGesture`は角丸の四角形に対するタップを検出している。タップされるたびに`tapCount`を1増やし、`Color`の色相をランダムに変更している。

`onLongPressGesture`は円を1秒以上長押しした操作を検出する。長押しに成功すると`isPressed`が`true`になり、円がオレンジ色から緑色へ変わる。また、大きさが1.3倍になり、文字も「長押し」から「成功!」へ変化する。1秒後に`isPressed`を`false`へ戻して、元の状態に戻している。

**なぜこう書くのか：**

`tapCount`、`backgroundColor`、`isPressed`は画面の表示に影響する値なので、`@State`で管理している。`@State`の値が変化すると、SwiftUIが画面を自動的に再描画する。

`minimumDuration: 1.0`を指定することで、1秒以上押した場合だけ長押しとして認識できる。`.animation(..., value: isPressed)`は、`isPressed`が変化したときだけ色や大きさの変更にアニメーションを付けるために使用している。

**もしこう書かなかったら：**

`@State`を使用しなければ、タップ回数や色などの変更を画面へ正しく反映できない。`minimumDuration`を短くすると、少し触っただけでも長押しとして判定されやすくなる。アニメーションを指定しなければ、円の大きさが瞬間的に切り替わる。

---

### ドラッグジェスチャーとオフセット管理

```swift
@State private var offset: CGSize = .zero
@State private var lastOffset: CGSize = .zero
```

```swift
.offset(offset)
.gesture(
    DragGesture()
        .onChanged { value in
            offset = CGSize(
                width: lastOffset.width + value.translation.width,
                height: lastOffset.height + value.translation.height
            )
        }
        .onEnded { _ in
            lastOffset = offset
        }
)
```

**何をしているか：**

`DragGesture`でカードのドラッグ操作を検出している。ドラッグ中は、前回の操作終了時の位置である`lastOffset`に、今回のドラッグ距離である`value.translation`を加え、現在位置を`offset`に保存している。

指を離したときは、現在の`offset`を`lastOffset`へ保存する。そのため、次にドラッグするときは前回の位置から続けて移動できる。

**なぜこう書くのか：**

`value.translation`は、今回のドラッグを開始した位置からの移動量である。前回の位置を加えないと、ドラッグを始めるたびに最初の位置を基準として計算される。`lastOffset`を使用することで、複数回のドラッグ操作を連続して反映できる。

**もしこう書かなかったら：**

`lastOffset`を使わずに`offset = value.translation`だけを書くと、2回目以降のドラッグで位置が急に変わる場合がある。また、`.offset(offset)`を書かなければ、ドラッグ操作を検出してもカードは画面上で移動しない。

---

### ドラッグ位置のリセット

```swift
Button("リセット") {
    withAnimation(.spring) {
        offset = .zero
        lastOffset = .zero
    }
}
```

**何をしているか：**

「リセット」ボタンを押すと、現在位置の`offset`と、保存されている前回位置の`lastOffset`を`.zero`に戻している。

**なぜこう書くのか：**

2つの変数を両方リセットすることで、カードの表示位置だけでなく、次回ドラッグ時の基準位置も初期状態に戻せる。`withAnimation(.spring)`を使うことで、カードが元の位置へ滑らかに戻る。

**もしこう書かなかったら：**

`offset`だけをリセットして`lastOffset`を戻さない場合、次にドラッグしたときに以前の位置情報が加算され、カードが急に移動する可能性がある。

---

### ピンチによる拡大縮小

```swift
@State private var scale: CGFloat = 1.0
@State private var lastScale: CGFloat = 1.0
```

```swift
.frame(width: 300, height: 300)
.contentShape(Rectangle())
.scaleEffect(scale)
.gesture(
    MagnifyGesture()
        .onChanged { value in
            scale = lastScale * value.magnification
        }
        .onEnded { _ in
            lastScale = scale
        }
)
```

**何をしているか：**

`MagnifyGesture`で2本指のピンチ操作を検出し、星の画像を拡大縮小している。`value.magnification`は現在のピンチ倍率を表し、前回の倍率`lastScale`と掛け合わせた値を`scale`へ保存している。

操作終了時には現在の倍率を`lastScale`へ保存するため、次のピンチ操作でも現在の大きさから続けて拡大縮小できる。

**なぜこう書くのか：**

`value.magnification`は、1回のピンチ操作を開始した時点を1.0として変化する。そのため、前回の倍率を掛け合わせる必要がある。

`.frame(width: 300, height: 300)`と`.contentShape(Rectangle())`を使用することで、星そのものだけでなく、周囲の透明な300×300の領域でもピンチ操作を認識できる。

**もしこう書かなかったら：**

`lastScale`を使用しない場合、ピンチ操作を始めるたびに倍率の基準が1.0へ戻り、画像の大きさが不自然に変化する可能性がある。`contentShape`がなければ、星の表示部分だけが操作対象となり、2本指で触りにくくなる。

---

### 2本指による回転

```swift
@State private var angle: Angle = .zero
@State private var lastAngle: Angle = .zero
```

```swift
.frame(width: 300, height: 300)
.contentShape(Rectangle())
.rotationEffect(angle)
.gesture(
    RotateGesture()
        .onChanged { value in
            angle = lastAngle + value.rotation
        }
        .onEnded { _ in
            lastAngle = angle
        }
)
```

**何をしているか：**

`RotateGesture`で2本指の回転操作を検出している。今回の回転量である`value.rotation`を、前回までの角度`lastAngle`に加え、現在の角度を`angle`へ保存している。

`.rotationEffect(angle)`によって、保存された角度に合わせて矢印を回転させている。操作終了時には現在角度を`lastAngle`へ保存する。

**なぜこう書くのか：**

回転操作を複数回行っても、前回の角度から続けて回転できるようにするためである。`Angle`型を使用すると、SwiftUIの`.rotationEffect`へそのまま値を渡せる。

**もしこう書かなかったら：**

`lastAngle`を使用しない場合、次の回転操作を開始するときに角度の基準がゼロへ戻り、矢印が急に別の角度へ変化する可能性がある。`.rotationEffect(angle)`がなければ、角度の値が変化しても画面上の矢印は回転しない。

---

### ジェスチャーの組み合わせ

```swift
.scaleEffect(scale)
.rotationEffect(angle)
.offset(offset)
.gesture(
    DragGesture()
        .onChanged { value in
            offset = CGSize(
                width: lastOffset.width + value.translation.width,
                height: lastOffset.height + value.translation.height
            )
        }
        .onEnded { _ in
            lastOffset = offset
        }
)
.simultaneousGesture(
    MagnifyGesture()
        .onChanged { value in
            scale = lastScale * value.magnification
        }
        .onEnded { _ in
            lastScale = scale
        }
)
.simultaneousGesture(
    RotateGesture()
        .onChanged { value in
            angle = lastAngle + value.rotation
        }
        .onEnded { _ in
            lastAngle = angle
        }
)
```

**何をしているか：**

1つの画像に、ドラッグ、ピンチによる拡大縮小、2本指による回転の3種類のジェスチャーを設定している。画像は移動しながら拡大縮小でき、同時に回転させることもできる。

**なぜこう書くのか：**

最初のドラッグ操作は`.gesture`で設定し、残りの拡大縮小と回転は`.simultaneousGesture`で追加している。`.simultaneousGesture`を使用すると、複数のジェスチャーを同時に認識できる。

それぞれの状態を`offset`、`scale`、`angle`に分けて管理することで、位置、大きさ、角度を独立して変更できる。

**もしこう書かなかったら：**

複数の`.gesture`をそのまま重ねると、ジェスチャー同士が競合し、一部の操作が反応しない場合がある。`.simultaneousGesture`を使わなければ、ピンチしながら回転するような操作が難しくなる。

---

### 組み合わせ画面のリセット

```swift
Button("リセット") {
    withAnimation(.spring) {
        offset = .zero
        lastOffset = .zero
        scale = 1.0
        lastScale = 1.0
        angle = .zero
        lastAngle = .zero
    }
}
```

**何をしているか：**

画像の位置、倍率、角度と、それぞれの前回値をすべて初期状態へ戻している。

**なぜこう書くのか：**

現在値だけでなく、次の操作の基準となる`lastOffset`、`lastScale`、`lastAngle`も同時に初期化する必要がある。これにより、リセット後のジェスチャー操作を正常に開始できる。

**もしこう書かなかったら：**

前回値をリセットしない場合、表示上は初期状態に戻っていても、次の操作時に以前の位置、倍率、角度が加算され、画像が急に変化する可能性がある。

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| `NavigationStack` | 画面遷移を管理するためのコンテナ | `NavigationStack { List { ... } }` |
| `NavigationLink` | タップしたときに別の画面へ移動する | `NavigationLink("ドラッグ") { DragDemoView() }` |
| `@State` | ビュー内で変化する値を管理し、画面を更新する | `@State private var tapCount = 0` |
| `onTapGesture` | タップ操作を検出する | `.onTapGesture { tapCount += 1 }` |
| `onLongPressGesture` | 長押し操作を検出する | `.onLongPressGesture(minimumDuration: 1.0) { ... }` |
| `DragGesture` | ドラッグ操作を検出する | `.gesture(DragGesture())` |
| `value.translation` | ドラッグ開始位置からの移動量を取得する | `value.translation.width` |
| `CGSize` | 横方向と縦方向の大きさや移動量を表す | `CGSize(width: 100, height: 50)` |
| `.zero` | 数値やサイズなどの初期値としてゼロを表す | `offset = .zero` |
| `MagnifyGesture` | 2本指のピンチ操作を検出する | `.gesture(MagnifyGesture())` |
| `value.magnification` | ピンチ操作中の拡大率を取得する | `scale = lastScale * value.magnification` |
| `RotateGesture` | 2本指の回転操作を検出する | `.gesture(RotateGesture())` |
| `value.rotation` | 回転ジェスチャー中の角度を取得する | `angle = lastAngle + value.rotation` |
| `Angle` | 回転角度を表す型 | `@State private var angle: Angle = .zero` |
| `simultaneousGesture` | 複数のジェスチャーを同時に認識する | `.simultaneousGesture(MagnifyGesture())` |
| `contentShape` | タッチ判定に使用する形を指定する | `.contentShape(Rectangle())` |
| `offset` | ビューの表示位置を移動する | `.offset(offset)` |
| `scaleEffect` | ビューを拡大・縮小する | `.scaleEffect(scale)` |
| `rotationEffect` | ビューを回転させる | `.rotationEffect(angle)` |
| `LinearGradient` | 複数の色を滑らかに変化させる | `LinearGradient(colors: [.purple, .blue], ...)` |
| `withAnimation` | 状態の変更にアニメーションを付ける | `withAnimation(.spring) { offset = .zero }` |
| `DispatchQueue.main.asyncAfter` | 指定時間後に処理を実行する | `.asyncAfter(deadline: .now() + 1)` |
| `String(format:)` | 数値の表示形式を指定する | `String(format: "倍率: %.1fx", scale)` |

## 自分の実験メモ

**実験1：長押し時間を変更した**

- やったこと：  
  `minimumDuration: 1.0`を`2.0`に変更した。

- 結果：  
  円を2秒以上押し続けなければ「成功!」と表示されなくなった。

- わかったこと：  
  `minimumDuration`の値を変更することで、長押しとして認識されるまでの時間を調整できる。

**実験2：ドラッグ後の位置を保存しないようにした**

- やったこと：  
  `DragGesture`の`.onEnded`にある`lastOffset = offset`を一時的に削除した。

- 結果：  
  2回目にカードをドラッグしたとき、前回の位置から自然に続けて移動できず、位置が不自然に変化した。

- わかったこと：  
  複数回のジェスチャー操作を連続して反映するには、操作終了時の値を別の変数へ保存する必要がある。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：なぜ`offset`と`lastOffset`の2つが必要なのか。**  
   **得られた理解：**  
   `offset`は現在の表示位置を表し、`lastOffset`は前回のドラッグ終了位置を保存する。2つを使うことで、前回の位置から続けてカードを動かせる。

2. **質問：なぜ`MagnifyGesture`で`lastScale * value.magnification`と書くのか。**  
   **得られた理解：**  
   `value.magnification`は毎回のピンチ開始時を1.0として計算されるため、前回までの倍率`lastScale`を掛ける必要がある。

3. **質問：なぜ複数のジェスチャーに`.simultaneousGesture`を使うのか。**  
   **得られた理解：**  
   `.gesture`を複数重ねるとジェスチャー同士が競合する場合がある。`.simultaneousGesture`を使用すると、ドラッグ、拡大縮小、回転を同時に認識できる。

## この章のまとめ

この章では、SwiftUIでタップ、ロングプレス、ドラッグ、ピンチによる拡大縮小、2本指による回転を実装する方法を学んだ。

ジェスチャーを検出するだけではなく、`@State`を使って位置、倍率、角度などの状態を管理し、`.offset`、`.scaleEffect`、`.rotationEffect`を使って画面表示へ反映することが重要である。

また、ドラッグの`lastOffset`、拡大縮小の`lastScale`、回転の`lastAngle`のように、操作終了時の値を保存することで、次の操作を現在の状態から続けられることが分かった。

複数のジェスチャーを同時に使用する場合は、`.simultaneousGesture`を使う必要がある。さらに、`.contentShape`で操作範囲を広げると、小さな画像でもジェスチャーを認識しやすくなる。今後、画像編集アプリや地図アプリなどを作成するときにも、今回学んだジェスチャー操作と状態管理を活用したい。
