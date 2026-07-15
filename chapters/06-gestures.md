# 第6章：ジェスチャー操作

> 執筆者：林 楽駿
> 最終更新：2026-07-15

## この章で学ぶこと

この章では、画面上で行われる指の操作を検出するジェスチャー機能について学ぶ。タップ、ロングプレス、ドラッグ、拡大縮小、回転などの基本的な操作を実装し、最後にドラッグ操作とアニメーションを組み合わせたTinder風スワイプカードUIを作成する。

## 模範コードの全体像

```swift
// 教員から配布された模範コード全体をここに貼り付ける
```

**このアプリは何をするものか：**

このアプリは、SwiftUIでさまざまなジェスチャー操作を確認するためのアプリである。

画面上の図形やカードに対して、タップ、長押し、ドラッグ、ピンチ操作、回転操作などを行うことができる。また、最後の応用例では、動物のカードを左右にスワイプして、好きな動物と好きではない動物に分類できる。

カードを右にスワイプすると「LIKE」、左にスワイプすると「NOPE」と表示され、スワイプした方向に応じてカウントが増える。すべてのカードを分類すると「完了！」と表示され、「もう一度」ボタンで最初からやり直すことができる。

## コードの詳細解説

### 基本ジェスチャー（タップ、ロングプレス）

```swift
@State private var message = "タップしてください"

Text(message)
    .padding()
    .background(Color.blue)
    .foregroundStyle(.white)
    .onTapGesture {
        message = "タップされました"
    }
    .onLongPressGesture {
        message = "長押しされました"
    }
```

**何をしているか：**

`onTapGesture`は、画面上の部品が1回タップされたことを検出する。タップされると、`message`の値を変更し、画面に表示される文字を更新する。

`onLongPressGesture`は、一定時間指を押し続けた操作を検出する。長押しされた場合も、`message`の内容を変更する。

**なぜこう書くのか：**

SwiftUIでは、ビューに対して`.onTapGesture`や`.onLongPressGesture`を追加することで、簡単にジェスチャー処理を設定できる。

`message`を`@State`で宣言しているため、値が変更されるとSwiftUIが自動的に画面を再描画する。

**もしこう書かなかったら：**

`onTapGesture`を書かなければ、画面をタップしても何も処理されない。

また、`message`を通常の変数として宣言すると、値を変更しても画面表示が正しく更新されない。画面上で変化する値には`@State`が必要である。

---

### ドラッグジェスチャーとオフセット管理

```swift
@State private var offset: CGSize = .zero

Circle()
    .fill(Color.blue)
    .frame(width: 120, height: 120)
    .offset(offset)
    .gesture(
        DragGesture()
            .onChanged { value in
                offset = value.translation
            }
            .onEnded { value in
                withAnimation(.spring) {
                    offset = .zero
                }
            }
    )
```

**何をしているか：**

`DragGesture`を使って、指のドラッグ操作を検出している。

ドラッグ中は、`value.translation`から指が最初の位置からどれくらい移動したかを取得し、その値を`offset`に代入している。

`.offset(offset)`によって、図形の表示位置が指の動きに合わせて移動する。

指を離した後は、`offset`を`.zero`に戻して、図形を元の位置に戻している。

**なぜこう書くのか：**

`value.translation`には横方向と縦方向の移動量が含まれているため、ドラッグした方向に図形を自然に動かすことができる。

位置情報を`@State`で管理することで、ドラッグ中の値が変わるたびに画面が更新される。

また、`withAnimation(.spring)`を使用することで、元の位置に戻る動きを自然なバネのようなアニメーションにできる。

**もしこう書かなかったら：**

`.offset(offset)`を書かなければ、ドラッグ操作は検出されても、図形は画面上で移動しない。

`.onChanged`を書かなければ、指を動かしている途中の位置を取得できない。

`.onEnded`で`offset`を戻さなければ、指を離した場所に図形が残る。

---

### 拡大縮小と回転

```swift
@State private var scale: CGFloat = 1.0
@State private var rotation: Angle = .zero

Image(systemName: "star.fill")
    .font(.system(size: 100))
    .foregroundStyle(.yellow)
    .scaleEffect(scale)
    .rotationEffect(rotation)
    .gesture(
        MagnificationGesture()
            .onChanged { value in
                scale = value
            }
    )
    .simultaneousGesture(
        RotationGesture()
            .onChanged { value in
                rotation = value
            }
    )
```

**何をしているか：**

`MagnificationGesture`を使って、2本の指を広げたり縮めたりするピンチ操作を検出している。

取得した値を`scale`に代入し、`.scaleEffect(scale)`で画像の大きさを変更している。

`RotationGesture`は、2本の指を使った回転操作を検出する。取得した角度を`rotation`に代入し、`.rotationEffect(rotation)`で画像を回転させている。

**なぜこう書くのか：**

拡大率と回転角度をそれぞれ`@State`で保存することで、指の操作に合わせて画像をリアルタイムで変化させることができる。

`.simultaneousGesture`を使うことで、拡大縮小ジェスチャーと回転ジェスチャーを同時に認識できる。

**もしこう書かなかったら：**

`.scaleEffect(scale)`を書かなければ、ピンチ操作を行っても画像の大きさは変わらない。

`.rotationEffect(rotation)`を書かなければ、回転ジェスチャーを認識しても画像は回転しない。

通常の`.gesture`を複数重ねるだけでは、一方のジェスチャーが優先され、拡大と回転を同時に操作できない場合がある。

---

### ジェスチャーの組み合わせとアニメーション

```swift
@State private var offset: CGSize = .zero
@State private var scale: CGFloat = 1.0
@State private var rotation: Angle = .zero

RoundedRectangle(cornerRadius: 20)
    .fill(Color.orange)
    .frame(width: 200, height: 200)
    .offset(offset)
    .scaleEffect(scale)
    .rotationEffect(rotation)
    .gesture(
        DragGesture()
            .onChanged { value in
                offset = value.translation
            }
    )
    .simultaneousGesture(
        MagnificationGesture()
            .onChanged { value in
                scale = value
            }
    )
    .simultaneousGesture(
        RotationGesture()
            .onChanged { value in
                rotation = value
            }
    )
```

**何をしているか：**

ドラッグ、拡大縮小、回転の3種類のジェスチャーを1つの図形に設定している。

ドラッグすると図形が移動し、ピンチすると大きさが変わり、2本の指を回すと図形も回転する。

**なぜこう書くのか：**

`.simultaneousGesture`を使うと、複数のジェスチャーを同時に認識できる。

それぞれの操作結果を`offset`、`scale`、`rotation`という別の状態変数で管理することで、処理内容が分かりやすくなる。

**もしこう書かなかったら：**

すべてのジェスチャーを通常の`.gesture`で設定すると、最後に設定したジェスチャーだけが反応したり、他のジェスチャーが認識されなかったりする場合がある。

また、状態変数を分けなければ、位置、大きさ、角度を正しく管理できない。

---

### Tinder風スワイプカード

```swift
@State private var offset: CGSize = .zero
@State private var rotation: Double = 0

private let swipeThreshold: CGFloat = 100

.gesture(
    DragGesture()
        .onChanged { value in
            offset = value.translation
            rotation = Double(value.translation.width / 20)
        }
        .onEnded { value in
            if value.translation.width > swipeThreshold {
                withAnimation(.easeOut(duration: 0.3)) {
                    offset = CGSize(width: 500, height: 0)
                }

                DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                    onSwipe(.right)
                }
            } else if value.translation.width < -swipeThreshold {
                withAnimation(.easeOut(duration: 0.3)) {
                    offset = CGSize(width: -500, height: 0)
                }

                DispatchQueue.main.asyncAfter(deadline: .now() + 0.3) {
                    onSwipe(.left)
                }
            } else {
                withAnimation(.spring) {
                    offset = .zero
                    rotation = 0
                }
            }
        }
)
```

**何をしているか：**

カードをドラッグした距離によって、右スワイプまたは左スワイプを判定している。

横方向の移動距離が`100`を超えた場合、スワイプが成立する。

右に100ポイント以上動かすとカードが画面右側へ移動し、`onSwipe(.right)`が実行される。左に100ポイント以上動かすと画面左側へ移動し、`onSwipe(.left)`が実行される。

100ポイント未満の場合は、カードが元の位置に戻る。

**なぜこう書くのか：**

少し触っただけでカードが削除されないように、`swipeThreshold`という基準値を設定している。

また、横方向の移動量を20で割って回転角度にすることで、カードがドラッグ方向に少し傾き、Tinderのような自然な動きになる。

カードが画面外へ移動するアニメーションが終わってからデータを削除するため、`DispatchQueue.main.asyncAfter`を使って0.3秒後に`onSwipe`を実行している。

**もしこう書かなかったら：**

`swipeThreshold`がなければ、少しカードを動かしただけでもスワイプが成立してしまう。

`DispatchQueue.main.asyncAfter`を使わず、すぐにカードを削除すると、画面外へ移動するアニメーションが表示される前にカードが消える。

`rotation`を変更しなければ、カードは傾かず、横方向に平行移動するだけになる。

---

### カード背景と透明度

```swift
RoundedRectangle(cornerRadius: 20)
    .fill(animal.color.opacity(0.15))
    .overlay(
        RoundedRectangle(cornerRadius: 20)
            .stroke(animal.color.opacity(0.3), lineWidth: 2)
    )
```

**何をしているか：**

角が丸い長方形をカードの背景として表示している。

`.fill(animal.color.opacity(0.15))`では、動物ごとに設定された色を透明度15％で背景に使用している。

`.stroke(animal.color.opacity(0.3), lineWidth: 2)`では、透明度30％、太さ2ポイントの枠線を表示している。

**なぜこう書くのか：**

背景色の透明度を低くすることで、文字や絵文字が読みやすくなり、柔らかいデザインになる。

背景と同じ色を枠線にも使うことで、カード全体の色を統一できる。

**もしこう書かなかったら：**

`opacity(0.15)`を`1.0`にすると、背景色が濃くなり、文字が見づらくなる場合がある。

逆に`opacity(0.0)`にすると、背景色が完全に透明になる。

`.overlay`を省略すると、カードの外側の枠線が表示されなくなる。

## 新しく学んだSwiftの文法・API

| 項目                              | 説明                   | 使用例                                                  |
| ------------------------------- | -------------------- | ---------------------------------------------------- |
| `onTapGesture`                  | ビューがタップされたことを検出する    | `.onTapGesture { count += 1 }`                       |
| `onLongPressGesture`            | ビューが長押しされたことを検出する    | `.onLongPressGesture { message = "長押し" }`            |
| `DragGesture`                   | 指によるドラッグ操作を検出する      | `.gesture(DragGesture().onChanged { value in ... })` |
| `value.translation`             | ドラッグ開始位置からの移動量を取得する  | `offset = value.translation`                         |
| `MagnificationGesture`          | ピンチ操作による拡大・縮小を検出する   | `.gesture(MagnificationGesture())`                   |
| `RotationGesture`               | 2本指による回転操作を検出する      | `.gesture(RotationGesture())`                        |
| `simultaneousGesture`           | 複数のジェスチャーを同時に認識する    | `.simultaneousGesture(RotationGesture())`            |
| `offset`                        | ビューの表示位置を移動する        | `.offset(offset)`                                    |
| `scaleEffect`                   | ビューを拡大・縮小する          | `.scaleEffect(scale)`                                |
| `rotationEffect`                | ビューを指定した角度だけ回転する     | `.rotationEffect(rotation)`                          |
| `withAnimation`                 | 状態の変化にアニメーションを付ける    | `withAnimation(.spring) { offset = .zero }`          |
| `opacity`                       | ビューや色の透明度を設定する       | `.opacity(0.5)`                                      |
| `DispatchQueue.main.asyncAfter` | 指定した時間が経過した後に処理を実行する | `.asyncAfter(deadline: .now() + 0.3)`                |
| `reversed()`                    | 配列の表示順を逆にする          | `ForEach(animals.reversed())`                        |
| `shuffled()`                    | 配列の順番をランダムに並べ替える     | `Animal.sampleData.shuffled()`                       |
| `removeAll`                     | 条件に一致する要素を配列から削除する   | `animals.removeAll { $0.id == animal.id }`           |

## 自分の実験メモ

**実験1：カード背景の透明度を変更した**

* やったこと：
  `.fill(animal.color.opacity(0.15))`の`0.15`を`0.5`に変更した。

* 結果：
  カードの背景色が濃くなり、動物ごとの色が分かりやすくなった。一方で、色によっては説明文が少し読みづらくなった。

* わかったこと：
  `opacity`の値は`0.0`から`1.0`まで設定できる。値が小さいほど透明になり、値が大きいほど色が濃くなる。

**実験2：スワイプ判定距離を変更した**

* やったこと：
  `swipeThreshold`を`100`から`50`に変更した。

* 結果：
  少しカードを横に動かしただけで、LIKEまたはNOPEとして判定されるようになった。

* わかったこと：
  しきい値が小さすぎると誤操作が増え、大きすぎるとカードを大きく動かす必要がある。操作しやすさを考えて適切な値を設定する必要がある。

## AIに聞いて特に理解が深まった質問 TOP3

1. **質問：カードの透明度はどのコードで設定しているか。**
   **得られた理解：**
   `.fill(animal.color.opacity(0.15))`の`0.15`が背景色の透明度である。また、`.stroke(animal.color.opacity(0.3), lineWidth: 2)`の`0.3`は枠線の透明度である。

2. **質問：なぜカードをドラッグすると傾くのか。**
   **得られた理解：**
   `rotation = Double(value.translation.width / 20)`によって、横方向の移動距離を回転角度に変換しているためである。右に動かすと右方向に、左に動かすと左方向に傾く。

3. **質問：なぜ複数のジェスチャーを同時に操作できない場合があるのか。**
   **得られた理解：**
   `.gesture`を複数使用すると、ジェスチャー同士が競合する場合がある。`.simultaneousGesture`を使用すると、ドラッグ、拡大縮小、回転などを同時に認識できる。

## この章のまとめ

この章では、SwiftUIでタップ、長押し、ドラッグ、拡大縮小、回転などのジェスチャーを実装する方法を学んだ。

ジェスチャーを検出するだけでなく、`@State`を使って位置、大きさ、角度などの状態を管理し、`.offset`、`.scaleEffect`、`.rotationEffect`を使って画面表示に反映することが重要である。

また、`withAnimation`を使用すると、状態の変化を滑らかに表示できる。複数のジェスチャーを同時に使用する場合は、`.simultaneousGesture`を使う必要がある。

Tinder風スワイプカードでは、ドラッグ距離を利用して左右の方向を判定し、しきい値を超えた場合だけカードを分類する仕組みを理解できた。今後、画像編集アプリや地図アプリなどを作る際にも、今回学んだジェスチャー操作を活用できると考える。
