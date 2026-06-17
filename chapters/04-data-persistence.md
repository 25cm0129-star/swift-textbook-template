# 第4章：データの永続化

> 執筆者：林楽駿
> 最終更新：2026 /6/17

## この章で学ぶこと

この章では、AppStorageとSwiftDataを使ってアプリのデータを端末に永続的に保存する方法を学ぶ。具体的にはSwiftDataを使ったメモアプリを題材として、@Modelでデータモデルを定義し、modelContextを使ったデータ操作、@Queryによる動的なデータ取得、そして@AppStorageによるユーザー設定の保存を実装する。
## 模範コードの全体像
import SwiftUI
import SwiftData

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}

struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingAddSheet = false
    @State private var isShowingSettings = false

    var displayedMemos: [Memo] {
        if sortByFavorite {
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        }
        return memos
    }

    var body: some View {
        NavigationStack {
            Group {
                if memos.isEmpty {
                    ContentUnavailableView(
                        "メモがありません",
                        systemImage: "note.text",
                        description: Text("右上の＋ボタンからメモを追加してください")
                    )
                } else {
                    List {
                        ForEach(displayedMemos) { memo in
                            NavigationLink(destination: MemoEditView(memo: memo)) {
                                MemoRow(memo: memo)
                            }
                        }
                        .onDelete(perform: deleteMemos)
                    }
                }
            }
            .navigationTitle(userName.isEmpty ? "メモ帳" : "\(userName)のメモ帳")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button { isShowingSettings = true } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button { isShowingAddSheet = true } label: {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingAddSheet) { MemoAddView() }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    func deleteMemos(at offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(displayedMemos[index])
        }
    }
}

struct MemoAddView: View {
    @Environment(\.modelContext) private var modelContext
    @Environment(\.dismiss) private var dismiss
    @State private var title = ""
    @State private var content = ""

    var body: some View {
        NavigationStack {
            Form {
                Section("タイトル") { TextField("メモのタイトル", text: $title) }
                Section("内容") {
                    TextEditor(text: $content).frame(minHeight: 200)
                }
            }
            .navigationTitle("新しいメモ")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .cancellationAction) {
                    Button("キャンセル") { dismiss() }
                }
                ToolbarItem(placement: .confirmationAction) {
                    Button("保存") {
                        modelContext.insert(Memo(title: title, content: content))
                        dismiss()
                    }
                    .disabled(title.isEmpty)
                }
            }
        }
    }
}

struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") { TextField("タイトル", text: $memo.title) }
            Section("内容") {
                TextEditor(text: $memo.content).frame(minHeight: 200)
            }
            Section { Toggle("お気に入り", isOn: $memo.isFavorite) }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") { TextField("あなたの名前", text: $userName) }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption).foregroundStyle(.secondary)
                }
            }
            .navigationTitle("設定")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .confirmationAction) {
                    Button("完了") { dismiss() }
                }
            }
        }
    }
}

#Preview {
    ContentView()
        .modelContainer(for: Memo.self, inMemory: true)
}

**このアプリは何をするものか：**

メモを作成・編集・削除できるシンプルなメモ帳アプリ。メモにはタイトル・本文・作成日時・お気に入りフラグがあり、端末を再起動してもデータが消えない。設定画面ではユーザー名の入力とお気に入り優先表示の切り替えができ、この設定もアプリを閉じても保持される。


## コードの詳細解説

### SwiftDataモデル（@Model）

@Model
class Memo {
    var title: String
    var content: String
    var createdAt: Date
    var isFavorite: Bool

    init(title: String, content: String, createdAt: Date = .now, isFavorite: Bool = false) {
        self.title = title
        self.content = content
        self.createdAt = createdAt
        self.isFavorite = isFavorite
    }
}
何をしているか：

@Modelマクロを付けることで、通常のSwiftクラスをSwiftDataが管理する永続化オブジェクトとして登録している。title・content・createdAt・isFavoriteの各プロパティは、そのまま内部ストア（SQLiteベース）に保存されるカラムに対応する。initでcreatedAtとisFavoriteにデフォルト値を持たせているため、呼び出し側はtitleとcontentだけ渡せばインスタンスを作れる。
なぜこう書くのか：

@Modelはクラス（参照型）に適用する設計になっている。SwiftDataが変更を追跡して自動保存するには、プロパティの変化がオブジェクト全体で共有される参照型である必要があるため、structではなくclassで定義している。デフォルト値を持たせるのは、メモ追加時のコードを簡潔にし、作成日時を毎回明示的に渡す必要をなくすため。
もしこう書かなかったら：

@Modelを外して通常のclassにすると、@Queryでの取得やmodelContext.insert/deleteの対象にできずビルドエラーになる。structとして定義すると@Modelマクロ自体が適用できず、これもビルドエラーになる。
---

### データの追加・削除（modelContext）

// 追加（MemoAddView）
modelContext.insert(Memo(title: title, content: content))
dismiss()

// 削除（ContentView）
func deleteMemos(at offsets: IndexSet) {
    for index in offsets {
        modelContext.delete(displayedMemos[index])
    }
}

何をしているか：

@Environment(\.modelContext)で取得したmodelContextは、SwiftDataの永続化ストアへの窓口となるオブジェクト。insert()で新しいMemoをストアに登録し、delete()で指定したMemoをストアから削除している。明示的にsave()を呼ばなくても、SwiftUIとの連携によって変更が自動的にディスクへ反映される。
なぜこう書くのか：

データの操作をmodelContextに集約することで、@Queryが監視しているデータと表示が常に同期する。自前で配列を@Stateとして持って操作すると、画面表示とディスク上のデータがずれてしまう可能性があるため、SwiftData側に「単一の真実の源」を任せる設計になっている。
もしこう書かなかったら：

modelContextを経由せず配列を直接操作すると、見た目上はリストが変わったように見えても永続化ストアには反映されず、アプリを再起動するとデータが元に戻ってしまう。

### @Queryによるデータ取得

@Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]

何をしているか：

@QueryはSwiftDataのストアからMemoを全件取得し、createdAtの降順（新しい順）で並べた配列をmemosに自動的にバインドする。ストア内のデータが変化すると自動的に検知され、memosの内容とView全体が再描画される。
なぜこう書くのか：

キーパスで並び替え条件を指定するだけで宣言的にソートが行え、データ変更時の再フェッチ処理を自分で書く必要がない。これによりContentView側のデータ管理コードを大幅に削減できる。
もしこう書かなかったら：

@Queryを使わず自前でfetchするコードを書くと、データが変わるたびに明示的な再取得と@State更新が必要になり、更新漏れによる表示崩れのリスクが生まれる。
### @AppStorageによる設定保存

@AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
@AppStorage("userName") private var userName: String = ""

何をしているか：

@AppStorageは指定したキーを使って内部的にUserDefaultsへ値を読み書きするプロパティラッパー。値の変更が自動的にUserDefaultsに保存され、アプリ再起動後も復元される。
なぜこう書くのか：

ユーザー名や表示設定のような軽量な値はSwiftDataの構造化データベースに保存するには大げさで、UserDefaultsをラップした@AppStorageの方が適している。宣言するだけで読み書きと永続化が自動化される。
もしこう書かなかったら：

通常の@Stateで宣言すると、アプリを終了して再起動した際に値が初期値にリセットされ、保存した設定が毎回失われてしまう。


（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
|@AppStorage |UserDefaultsと自動的に同期するプロパティラッパー | @AppStorage("userName") var userName: String = ""|
|modelContext |SwiftDataストアへのデータ操作の窓口となる環境値 |@Environment(\.modelContext) var modelContext |
| @Bindable|参照型（@Model）のプロパティへ直接Bindingを作るマクロ |@Bindable var memo: Memo → TextField(..., text: $memo.title) |

## 自分の実験メモ

（模範コードを改変して試したことを書く）

**実験1：**
やったこと：SettingsViewでユーザー名を「林」に変更し、アプリを終了して再起動した。
結果：再起動後もナビゲーションタイトルが「林のメモ帳」のまま表示された。
わかったこと：@AppStorageで保存した値はUserDefaultsに保存されるため、アプリを終了してもデータが保持されることが分かった。

**実験2：**
やったこと：複数のメモを作成し、一部のメモだけをお気に入りに設定して「お気に入りを上に表示」をONにした。
結果：お気に入りに設定したメモが一覧の上部へ移動した。
わかったこと：表示順はデータベースの保存順ではなく、displayedMemosで加工した結果を利用して変更できることが分かった。また、データ自体を変更しなくても表示方法だけを変更できることを理解した。
## AIに聞いて特に理解が深まった質問 TOP3

1. 質問：

なぜSwiftDataでは@Modelを付けたclassを使うのですか。

得られた理解：
最初はstructでも同じように保存できると思っていた。しかし、SwiftDataはオブジェクトの変更を監視するため、参照型であるclassを利用していることが分かった。また、@Modelを付けることでSwiftDataが自動的に永続化対象として認識することを理解した。

2. 質問：

@Queryはどのような仕組みでデータを自動更新しているのですか。

得られた理解：
最初は画面を更新するために毎回データを取得し直していると思った。しかし、@QueryはSwiftDataの変更を監視しており、データが追加・削除・編集されたときに自動的にViewを再描画していることが分かった。そのため、自分で再読み込み処理を書く必要がないことを理解した。

3. 質問：

@AppStorageとSwiftDataはどのように使い分けるのですか。

得られた理解：
最初はすべてSwiftDataで保存すればよいと思っていた。しかし、ユーザー名や表示設定のような単純な値は@AppStorageの方が簡単に管理できることが分かった。一方で、メモのように複数の項目を持つデータはSwiftDataで管理する方が適していることを理解した。
## この章のまとめ
今回の章では、SwiftDataとAppStorageを利用してデータを永続化する方法を学んだ。

特に印象に残ったのは、SwiftDataではデータベースを直接操作するようなコードを多く書かなくても、@Modelや@Queryを利用することで簡単にデータ管理ができる点である。また、データが変更されるとSwiftUIが自動的に画面を更新するため、表示とデータの整合性を保ちやすいことも理解した。

さらに、ユーザー設定のような簡単な情報は@AppStorageで保存し、メモのような構造化されたデータはSwiftDataで保存するという使い分けも学んだ。

最初は「データ保存」と聞くと難しく感じていたが、SwiftUIとSwiftDataの仕組みを利用することで、少ないコードで永続化機能を実現できることが分かった。今後アプリを作る際には、保存するデータの種類に応じて適切な保存方法を選択できるようになりたい。
