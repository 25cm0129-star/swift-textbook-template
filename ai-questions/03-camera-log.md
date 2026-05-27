# AI質問ログ：第3章 カメラの利用

## 使用した生成AIツール

Claude

## 質問と回答の記録

Q1
質問：
第3章のドキュメントの空欄を全部埋めてほしい（解説・まとめ・APIテーブルなど）
AIの回答の要点：

各セクション（PhotosPicker・非同期読み込み・UIViewControllerRepresentable・Coordinator）について「何をしているか」「なぜこう書くのか」「もしこう書かなかったら」を埋めてくれた
APIテーブル、実験アイデア3つ、AI質問TOP3の案も出してくれた



Q2
質問：
PhotosPickerによる写真選択に該当するコードはどこか
AIの回答の要点：

PhotosPicker(selection: $selectedItem, matching: .images) { ... } の部分と、.onChange(of: selectedItem) { ... } の2つがセットだと教えてくれた
PhotosPicker 自体はUIを表示するだけで、選ばれた写真は .onChange で検知して loadImage に渡す、という流れ


Q3
質問：
「このアプリは何をするものか」から「Coordinatorパターン」まで、コード抜粋込みで全セクションを埋めてほしい
AIの回答の要点：

各セクションのコード抜粋と3つの観点（何をしているか・なぜこう書くのか・もしこう書かなかったら）を埋めてくれた
特に重要なポイント：

Task { } が必要な理由（.onChange は同期コンテキストのため）
Coordinator がクラスでないといけない理由（UIKitのデリゲートは AnyObject 制約がある）
picker.delegate = context.coordinator を忘れると画像が渡ってこない


今日の質問を振り返って
良かった質問：
Q2のように「この機能に該当するコードはどこか」と範囲を絞って聞くと、コード全体のどの部分が何の役割を担っているかが明確になった。
気になる点・確認したいこと：
AIの回答が正しいかどうか、以下は自分で実際に試して確かめたい：

matching: .images を外したら本当に動画も出てくるか
Task { } を外したらコンパイルエラーになるか
picker.delegate = context.coordinator をコメントアウトしたら撮影後に何も起きないか

次回聞いてみたいこと：
「@ViewBuilder は何？」「fullScreenCover と sheet の違いは？」

