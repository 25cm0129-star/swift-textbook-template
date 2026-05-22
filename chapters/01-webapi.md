# 第1章：WebAPIの基本

> 執筆者：林楽駿
> 最終更新：2026/5/22
## この章で学ぶこと
この章では、インターネット上のサービス（API）からデータを取得して、アプリ内に表示する方法を学ぶ。具体的にはiTunes Search APIを使って音楽を検索し、その結果をリスト表示するアプリを題材にする。また、MVVMパターンを使ったコード設計と、エラーが発生したときの適切な処理方法についても学ぶ。


## 模範コードの全体像


// ============================================
// 第1章（応用）：エラーハンドリングとMVVM構成
// ============================================
// 基本編のコードをMVVMパターンで書き直し、
// エラーハンドリングとローディング状態を
// より適切に管理するバージョンです。
// ============================================

import SwiftUI

// MARK: - データモデル

struct SearchResponse: Codable {
    let resultCount: Int
    let results: [Song]
}

struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let collectionName: String?
    let artworkUrl100: String
    let previewUrl: String?
    let trackPrice: Double?
    let currency: String?

    var id: Int { trackId }

    var priceText: String {
        guard let price = trackPrice, let currency = currency else {
            return "価格不明"
        }
        return "\(currency) \(String(format: "%.0f", price))"
    }
}

// MARK: - ViewModel

@Observable
class MusicSearchViewModel {
    var songs: [Song] = []
    var searchText: String = ""
    var isLoading: Bool = false
    var errorMessage: String?

    enum SearchError: LocalizedError {
        case invalidURL
        case networkError(Error)
        case decodingError(Error)
        case noResults

        var errorDescription: String? {
            switch self {
            case .invalidURL:
                return "検索URLの作成に失敗しました"
            case .networkError(let error):
                return "通信エラー: \(error.localizedDescription)"
            case .decodingError:
                return "データの読み取りに失敗しました"
            case .noResults:
                return "検索結果が見つかりませんでした"
            }
        }
    }

    func searchMusic() async {
        guard !searchText.trimmingCharacters(in: .whitespaces).isEmpty else { return }

        guard let encodedText = searchText.addingPercentEncoding(
            withAllowedCharacters: .urlQueryAllowed
        ) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }

        let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

        guard let url = URL(string: urlString) else {
            errorMessage = SearchError.invalidURL.errorDescription
            return
        }

        isLoading = true
        errorMessage = nil

        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            let response = try JSONDecoder().decode(SearchResponse.self, from: data)

            if response.results.isEmpty {
                errorMessage = SearchError.noResults.errorDescription
                songs = []
            } else {
                songs = response.results
            }
        } catch let error as DecodingError {
            errorMessage = SearchError.decodingError(error).errorDescription
            songs = []
        } catch {
            errorMessage = SearchError.networkError(error).errorDescription
            songs = []
        }

        isLoading = false
    }
}

// MARK: - メインビュー

struct ContentView: View {
    @State private var viewModel = MusicSearchViewModel()

    var body: some View {
        NavigationStack {
            VStack(spacing: 0) {
                searchBar

                if let errorMessage = viewModel.errorMessage {
                    ErrorBanner(message: errorMessage)
                }

                contentArea
            }
            .navigationTitle("Music Search")
        }
    }

    // MARK: - 検索バー

    private var searchBar: some View {
        HStack {
            TextField("アーティスト名を入力", text: $viewModel.searchText)
                .textFieldStyle(.roundedBorder)
                .onSubmit {
                    Task { await viewModel.searchMusic() }
                }

            Button("検索") {
                Task { await viewModel.searchMusic() }
            }
            .buttonStyle(.borderedProminent)
            .disabled(viewModel.searchText.isEmpty || viewModel.isLoading)
        }
        .padding()
    }

    // MARK: - コンテンツエリア

    @ViewBuilder
    private var contentArea: some View {
        if viewModel.isLoading {
            Spacer()
            ProgressView("検索中...")
            Spacer()
        } else if viewModel.songs.isEmpty {
            ContentUnavailableView(
                "曲を検索してみよう",
                systemImage: "music.note",
                description: Text("アーティスト名を入力して検索ボタンを押してください")
            )
        } else {
            List(viewModel.songs) { song in
                NavigationLink(destination: SongDetailView(song: song)) {
                    SongRow(song: song)
                }
            }
        }
    }
}

// MARK: - 曲の行ビュー

struct SongRow: View {
    let song: Song

    var body: some View {
        HStack(spacing: 12) {
            AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                image.resizable().aspectRatio(contentMode: .fill)
            } placeholder: {
                RoundedRectangle(cornerRadius: 8)
                    .fill(.gray.opacity(0.2))
            }
            .frame(width: 60, height: 60)
            .clipShape(RoundedRectangle(cornerRadius: 8))

            VStack(alignment: .leading, spacing: 4) {
                Text(song.trackName)
                    .font(.headline)
                    .lineLimit(1)
                Text(song.artistName)
                    .font(.subheadline)
                    .foregroundStyle(.secondary)
            }

            Spacer()

            Text(song.priceText)
                .font(.caption)
                .foregroundStyle(.secondary)
        }
        .padding(.vertical, 4)
    }
}

// MARK: - 詳細ビュー

struct SongDetailView: View {
    let song: Song

    var body: some View {
        ScrollView {
            VStack(spacing: 20) {
                AsyncImage(url: URL(string: song.artworkUrl100)) { image in
                    image.resizable().aspectRatio(contentMode: .fit)
                } placeholder: {
                    ProgressView()
                }
                .frame(width: 200, height: 200)
                .clipShape(RoundedRectangle(cornerRadius: 16))
                .shadow(radius: 8)

                Text(song.trackName)
                    .font(.title2)
                    .bold()
                    .multilineTextAlignment(.center)

                Text(song.artistName)
                    .font(.title3)
                    .foregroundStyle(.secondary)

                if let albumName = song.collectionName {
                    Text(albumName)
                        .font(.subheadline)
                        .foregroundStyle(.tertiary)
                }

                Text(song.priceText)
                    .font(.headline)
                    .padding(.horizontal, 16)
                    .padding(.vertical, 8)
                    .background(.blue.opacity(0.1))
                    .clipShape(Capsule())
            }
            .padding()
        }
        .navigationTitle("曲の詳細")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - エラーバナー

struct ErrorBanner: View {
    let message: String

    var body: some View {
        HStack {
            Image(systemName: "exclamationmark.triangle.fill")
                .foregroundStyle(.yellow)
            Text(message)
                .font(.caption)
        }
        .padding(10)
        .frame(maxWidth: .infinity)
        .background(.red.opacity(0.1))
    }
}

#Preview {
    ContentView()
}


**このアプリは何をするものか：**

アーティスト名を入力して検索ボタンを押すと、iTunes Search APIから音楽データを取得し、曲名・アーティスト名・価格をリスト表示するアプリ。曲をタップすると詳細画面に遷移する。通信中はローディング表示、エラー時はバナーでメッセージを表示する。
## コードの詳細解説

### データモデル（Codable構造体）
struct Song: Codable, Identifiable {
    let trackId: Int
    let trackName: String
    let artistName: String
    let collectionName: String?
    let artworkUrl100: String
    let previewUrl: String?
    let trackPrice: Double?
    let currency: String?

    var id: Int { trackId }

    var priceText: String {
        guard let price = trackPrice, let currency = currency else {
            return "価格不明"
        }
        return "\(currency) \(String(format: "%.0f", price))"
    }
}
**何をしているか：**
APIから取得したJSONデータをSwiftの構造体に変換するためのモデルを定義している。CodableによってJSONと構造体を自動的に対応させ、IdentifiableによってListでの表示が可能になる。
**なぜこう書くのか：**

Codableを使うことで、JSONのキー名とプロパティ名を一致させるだけで自動変換できる。手動でJSONを解析するより短く、ミスが少ない。

**もしこう書かなかったら：**

Codableがない場合、JSONのデータを一つ一つ手動で取り出す必要があり、コードが長くなりバグも増える。
---

### API通信の処理

func searchMusic() async {
    guard !searchText.trimmingCharacters(in: .whitespaces).isEmpty else { return }

    let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=25"

    isLoading = true
    errorMessage = nil

    do {
        let (data, _) = try await URLSession.shared.data(from: url)
        let response = try JSONDecoder().decode(SearchResponse.self, from: data)
        songs = response.results
    } catch {
        errorMessage = SearchError.networkError(error).errorDescription
        songs = []
    }

    isLoading = false
}

**何をしているか：**

iTunes APIにリクエストを送り、返ってきたJSONデータをSearchResponseに変換してsongsに保存している。通信中はisLoadingをtrueにし、エラーが発生した場合はerrorMessageにメッセージを入れる。

**なぜこう書くのか：**

async/awaitを使うことで、非同期処理を上から下へ読める形で書ける。do-catchでエラーの種類ごとに対応できるため、ユーザーに適切なエラーメッセージを伝えられる。

**もしこう書かなかったら：**

async/awaitを使わない場合、クロージャ（completion handler）を使う必要があり、ネストが深くなって読みにくいコードになる。

---

### ビューの構成

@ViewBuilder
private var contentArea: some View {
    if viewModel.isLoading {
        ProgressView("検索中...")
    } else if viewModel.songs.isEmpty {
        ContentUnavailableView(
            "曲を検索してみよう",
            systemImage: "music.note"
        )
    } else {
        List(viewModel.songs) { song in
            NavigationLink(destination: SongDetailView(song: song)) {
                SongRow(song: song)
            }
        }
    }
}
**何をしているか：**
状態（ローディング中・結果なし・結果あり）に応じて表示する画面を切り替えている。ローディング中はプログレスビュー、結果がなければ案内メッセージ、結果があればリストを表示する。

**なぜこう書くのか：**

@ViewBuilderを使うことで、条件分岐を含むビューを一つの変数にまとめられる。ContentViewのbodyがすっきりして読みやすくなる。


**もしこう書かなかったら：**
すべてをbodyに直接書くと、条件分岐が増えたときにコードが長くなり、管理しにくくなる。

---

（必要に応じてセクションを増やす）

## 新しく学んだSwiftの文法・API

| 項目 | 説明 | 使用例 |
|------|------|--------|
| 例：`Codable` | JSONデータとSwiftの構造体を相互変換するプロトコル | `struct Song: Codable { ... }` |
| 例：`async/await` | 非同期処理を同期的に書ける構文 | `let data = try await URLSession.shared.data(from: url)` |
|@Observable |クラスの変更をビューに自動反映させるマクロ | @Observable class MusicSearchViewModel|
|@ViewBuilder |条件分岐を含むビューをまとめて返せる属性 | @ViewBuilder private var contentArea: some View|
| ContentUnavailableView|データがない状態を表示するための標準ビュー | ContentUnavailableView("タイトル", systemImage: "music.note")|

## 自分の実験メモ
実験１、let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=jp&limit=5"
実験２、let urlString = "https://itunes.apple.com/search?term=\(encodedText)&media=music&country=us&limit=25"
実験３、class MusicSearchViewModel: ObservableObject {
    @Published var songs: [Song] = []
    @Published var searchText: String = ""
    @Published var isLoading: Bool = false
    @Published var errorMessage: String?
}
**実験1：**
実験1：

やったこと：limit=25をlimit=5に変更してみた
結果：検索結果が5件しか表示されなくなった
わかったこと：URLのパラメータを変えることでAPIの返す件数を制御できる

**実験2：**
実験2：

やったこと：country=jpをcountry=usに変更してみた
結果：価格がUSDで表示されるようになった
わかったこと：countryパラメータで取得する国のデータを切り替えられる

## AIに聞いて特に理解が深まった質問 TOP3

質問1： Codableと手動のJSONパースの違いは何か？
得られた理解： Codableを使えばプロパティ名とJSONのキーを一致させるだけで自動変換できる。手動だとas? Stringなどを使って一つずつ取り出す必要があり、コードが長くなる。

質問2： async/awaitを使わないとどうなるか？
得られた理解： クロージャを使った書き方になり、処理が入れ子になって読みにくくなる。async/awaitは非同期処理を上から下に読める形で書けるため、可読性が大きく上がる。

質問3： @Observableと@ObservableObjectの違いは何か？
得られた理解： @ObservableObjectは古い書き方で、@Publishedを各プロパティにつける必要がある。@ObservableはSwift 5.9以降の新しい書き方で、マクロが自動的に変更を検知するため、@Publishedが不要でコードがシンプルになる。

## この章のまとめ

この章では、WebAPIからデータを取得してアプリに表示する基本的な流れを学んだ。CodableでJSONを構造体に変換し、async/awaitで非同期通信を行い、MVVMパターンでビューとロジックを分離することで、読みやすく管理しやすいコードが書けることがわかった。エラーハンドリングを適切に行うことで、ユーザーに分かりやすいフィードバックを返せることも重要なポイントだった。
