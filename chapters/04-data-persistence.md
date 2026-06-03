# 第4章：データの永続化

> 執筆者：卞宜璇
> 最終更新：2026-06-03

import SwiftUI
import SwiftData

// MARK: - SwiftDataモデル
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

// MARK: - メインビュー
struct ContentView: View {
    @Environment(\.modelContext) private var modelContext
    @Query(sort: \Memo.createdAt, order: .reverse) private var memos: [Memo]
    @AppStorage("sortByFavorite") private var sortByFavorite: Bool = false
    @AppStorage("userName") private var userName: String = ""
    @State private var isShowingSettings = false

    // 表示用のフィルター/ソート済みのメモ一覧
    var displayedMemos: [Memo] {
        if sortByFavorite {
            // お気に入りがONのものを優先して上に、その他は作成日時順
            return memos.sorted { $0.isFavorite && !$1.isFavorite }
        } else {
            return memos
        }
    }

    var body: some View {
        NavigationStack {
            List {
                ForEach(displayedMemos) { memo in
                    NavigationLink(destination: MemoEditView(memo: memo)) {
                        HStack {
                            VStack(alignment: .leading, spacing: 4) {
                                Text(memo.title.isEmpty ? "無題のメモ" : memo.title)
                                    .font(.headline)
                                Text(memo.createdAt, style: .date)
                                    .font(.caption)
                                    .foregroundStyle(.secondary)
                            }
                            Spacer()
                            if memo.isFavorite {
                                Image(systemName: "star.fill")
                                    .foregroundStyle(.yellow)
                            }
                        }
                    }
                }
                .onDelete(perform: deleteMemos)
            }
            .navigationTitle(userName.isEmpty ? "マイメモ" : "\(userName)のメモ")
            .toolbar {
                ToolbarItem(placement: .topBarLeading) {
                    Button {
                        isShowingSettings = true
                    } label: {
                        Image(systemName: "gear")
                    }
                }
                ToolbarItem(placement: .topBarTrailing) {
                    Button(action: addMemo) {
                        Image(systemName: "plus")
                    }
                }
            }
            .sheet(isPresented: $isShowingSettings) {
                SettingsView(userName: $userName, sortByFavorite: $sortByFavorite)
            }
        }
    }

    // メモの追加
    private func addMemo() {
        let newMemo = Memo(title: "新規メモ", content: "")
        modelContext.insert(newMemo)
    }

    // メモの削除
    private func deleteMemos(offsets: IndexSet) {
        for index in offsets {
            modelContext.delete(displayedMemos[index])
        }
    }
}

// MARK: - 編集画面（@Bindableの活用）
struct MemoEditView: View {
    @Bindable var memo: Memo

    var body: some View {
        Form {
            Section("タイトル") {
                TextField("タイトル", text: $memo.title)
            }
            Section("内容") {
                TextEditor(text: $memo.content)
                    .frame(minHeight: 200)
            }
            Section {
                Toggle("お気に入り", isOn: $memo.isFavorite)
            }
        }
        .navigationTitle("メモを編集")
        .navigationBarTitleDisplayMode(.inline)
    }
}

// MARK: - 設定画面（AppStorageの活用）
struct SettingsView: View {
    @Binding var userName: String
    @Binding var sortByFavorite: Bool
    @Environment(\.dismiss) private var dismiss

    var body: some View {
        NavigationStack {
            Form {
                Section("ユーザー設定") {
                    TextField("あなたの名前", text: $userName)
                }
                Section("表示設定") {
                    Toggle("お気に入りを上に表示", isOn: $sortByFavorite)
                }
                Section {
                    Text("設定はアプリを閉じても保存されます")
                        .font(.caption)
                        .foregroundStyle(.secondary)
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
