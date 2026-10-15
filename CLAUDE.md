# 萩原歯科 患者進行状況ボード — プロジェクト引き継ぎメモ

このファイルは、これまでClaude(チャット)で行ってきた開発作業をClaude Codeに引き継ぐための要約です。
作業を始める前に、必ずこのファイル全体を読んでください。

## プロジェクト概要

- 歯科医院(萩原歯科医院、群馬県吾妻郡長野原町)向けの、Kanban形式の患者進行状況管理アプリ
- オーナー: 萩原俊一(矯正歯科医)。息子が萩原歯科の院長を継いでいる
- 本番URL: https://hagiwaradental.pages.dev/board (board.html)
- デプロイ先: Cloudflare Pages(プロジェクト名「hagiwaradental」)。GitHub連携ではなく、
  Cloudflareダッシュボードから **手動でファイルをアップロード** するデプロイ方式
- バックエンド: Firebase / Firestore(リアルタイム同期、`onSnapshot`)
- 認証: Firebase Authentication(メール+パスワード)

## ファイル構成

- `board.html` — メインアプリ。スタッフが使う患者ボード本体(最も大きく、最も頻繁に変更される)
- `index.html` — クリニックの一般向けトップページ
- `intake.html` — 患者がスマホで記入する問診票フォーム(**現在は運用で使われていない**。ただしコードは残っている)
- `reserve.html` — 患者向けのネット予約フォーム
- `privacy.html` — プライバシーポリシーページ
- `_headers` — Cloudflare Pages用の簡易ヘッダー設定(Content-Typeのみ)

デプロイ時は、これら全ファイルをまとめてCloudflare Pagesの「Create deployment」でアップロードする
(個別ファイル差し替えではなく、フォルダの中身をまとめてアップロードする運用)。

## board.htmlの主要な仕組み

- `patients` collection: 1来院=1ドキュメント。`createdAt`をその日の日付として、
  `attachPatientsListener()`が`where('createdAt','>=',todayStart).where('createdAt','<',tomorrowStart)`
  で「今日の分だけ」をリアルタイム購読する
- `status`フィールドで列(問診済み→待ち中→診療中→会計待ち→会計済み)を管理。
  `needsTechnicianOrder`は別軸のフラグで、技工指示済み列は`status`と独立して表示される
  (同じ患者が「未来院」かつ「技工指示済み」に同時に出ることがあるが、これは仕様)
- `patientRoster` collection: カルテ番号をキーにした患者マスタ(氏名・連絡先・アレルギー等の基本情報、
  次回処置予定のメモなど「毎回変わらない情報」を保持)
- 「次回来院予定」を日時付きで登録すると、`createLinkedFuturePatient()`がその日付の
  `patients`ドキュメントを`status:"問診済み"`で事前作成し、予約日に自動で「未来院」欄に出てくる仕組みがある
- `externallyLoadedPatientIds`: 履歴検索などで「今日以外の日付」の患者を一時的に開いたときに
  使うSet。ボードの各列の表示・CSV出力・技工一括印刷などは、すべてこのSetを除外してから表示する
  必要がある(過去にこの除外漏れが何度かバグの原因になった)

## これまでに発生・修正した不具合(時系列)

1. 次回予約カテゴリタグの編集可能化・自動配色・4箇所への反映漏れ修正
2. 未来院患者でもアレルギー/基本情報が表示されない問題 → render()呼び忘れ・全角半角カルテ番号不一致を修正
3. 次回処置予定の内容が、実際の再診時に引き継がれない問題 → `findCarriedForwardPlan()`を追加
4. カルテ内容が部位ごとに改行されない → `\n`→`<br>`変換を修正
5. X線画像保存機能を一度実装 → ユーザーの指示で全面リバート(「一旦見送り」)。再実装時は
  Firebase Storage、患者モーダル内・基本情報と同じ場所に表示、JPEG前提、という仕様で合意済み
6. バックアップ/復元に`staffShifts`コレクションが含まれていなかった → 追加
7. ボードカードの「本日/次回予定」タグと「基本情報」タグの色・順序が紛らわしい → 新しい紫色の
  `.planned-treatment-note`クラスを追加し、黄色の`.card-base-note`は基本情報専用のまま維持
8. ②③タグの挿入先取り違え(全体メモ欄と個別診断メモ欄) → `insertGeneralNoteTag`/
  `insertDiagnosisNoteTag`/`insertOrthoWireTag`に分離
9. 技工一括印刷/CSV出力が`externallyLoadedPatientIds`を除外していなかった → 修正

### 直近の大きな不具合調査(2026-09-28〜29、このセッションで対応)

**症状**: 朝ボードを開くと、何もしていないのに前日の患者が「会計待ち」「技工指示済み」等の
古いステータスのまま表示されてしまう。

**根本原因と修正(複数あった)**:

1. **デモデータの誤表示**: Firebase初期化が例外で失敗した場合、`firebaseReady=false`のまま
   `needsTechnicianOrder:true`や`status:"会計待ち"`を含むデモ用ダミーデータをそのまま表示していた。
   → 「設定未入力」と「接続失敗」を区別し、接続失敗時はダミーデータを出さず、全画面の赤い警告
   (`#firebaseErrorOverlay`)を表示してボード操作自体をブロックするよう修正。

2. **日付切り替えの検知漏れ(本命)**: `attachPatientsListener()`は「今日」の日付範囲でFirestoreを
   購読するが、日付が変わったことを検知する仕組みが「1分おきの`setInterval`」と
   `visibilitychange`/`focus`/`pageshow`イベントに依存していた。**Macがスリープ(蓋を閉じる)した
   場合、タブが元々フォアグラウンドのままだと復帰時にこれらのイベントが発火しないことがあり、
   日付切り替えが長時間検知されない**ことが実際に再現して確認された(2日連続で発生)。
   → 対策として、`render()`関数の先頭でも毎回「今日かどうか」を再チェックするようにし、
   さらに`click`/`touchstart`/`keydown`イベントでも強制チェックするようにした(保険の多重化)。
   **この対策は未検証(ユーザーがまだ翌朝の挙動を確認していない)。再発する場合は、
   Firestoreのデータそのもの(該当ドキュメントの`createdAt`の実際の値)を直接確認する必要がある。**

3. **同一患者の未来院/待ち中への二重登録**: `findTodayPatientByChartNumber()`が
   `externallyLoadedPatientIds`を除外しておらず、かつ全角/半角カルテ番号の表記ゆれで
   「今日まだ来ていない」と誤判定し、新しいレコードを作ってしまうことがあった。
   → 比較時に`toHalfWidthDigits()`で正規化し、`externallyLoadedPatientIds`も除外するよう修正。
   intake.html/reserve.htmlのカルテ番号入力にも同様の全角→半角変換を追加(intake.htmlは
   現在未使用とのことだが、過去データに全角カルテ番号が残っている可能性があるため保険として維持)。

4. **モーダルを開いた直後、次回予定欄が一瞬空に見える**: Firestoreへの書き込み直後に
   モーダルを開くと、リアルタイム反映のタイミング差で「次回予定処置」表示が一瞬空になることがあった。
   → `renderPlannedTreatmentSection()`として分離し、リアルタイム同期でデータが届いた瞬間に
   自動で再描画するようにした。

## 未解決・要フォローアップ

- **上記3の日付切り替え不具合は、修正の効果が未確認。** ユーザーに「翌朝また同じ現象が起きるか」
  を確認してもらう必要がある。再発する場合は、Firebaseコンソール(または`db.collection('patients')`
  への直接クエリ)で該当ドキュメントの生の`createdAt`値を確認し、本当に「today」の範囲に
  入ってしまっているのか、それとも別の原因(クライアント側の表示バグ)かを切り分けること。
- X線画像保存機能は意図的に未実装のまま(設計のみ合意済み、上記5参照)。実装時はFirebase Storageの
  Blazeプラン(要クレジットカード)が必要になる点をユーザーに再確認すること。
- intake.html(問診票)は現在クリニックで使われていない。将来的に削除するか判断が必要かもしれないが、
  ユーザーからの明示的な指示があるまでは残しておくこと。

## 開発・検証の進め方(このセッションでの慣習)

- board.htmlは非常に大きい(17000行弱)単一HTMLファイル。Reactやビルドツールは使っていない、
  素のJS+Firebase compat SDK
- コード変更後は、jsdomで実際に`<script>`を実行してランタイムエラーが出ないか確認する運用を
  取っていた(特にtemporal dead zone系のエラーに注意)。Node.jsで以下のように検証できる:
  ```js
  const { JSDOM } = require('jsdom');
  const dom = new JSDOM(fs.readFileSync('board.html','utf8'), {
    runScripts: 'dangerously', resources: 'usable', url: 'https://example.com/board.html'
  });
  ```
- HTMLの`<div>`開閉タグ数が一致しているかのような単純な構造チェックも併用していた
- 変更後は、ユーザーには常に更新した3〜5ファイル(index.html, board.html, intake.html, reserve.html,
  privacy.html)をまとめたzipで渡し、Cloudflare Pagesへの手動アップロード手順を案内していた
- ユーザーとのやり取りは日本語。ユーザーは開発者ではないが自分でツールを作るタイプなので、
  技術的な説明はある程度詳しく行って問題ない
