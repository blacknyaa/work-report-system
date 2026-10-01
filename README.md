# 作業完了報告書SYSTEM（プロトタイプ）

病院設備の保守点検報告書を、現場のタブレットから入力してPDFまで出すシステム。
概要書（全26ページ）の画面番号をそのまま実装の単位にしている。

**概要書の35画面すべてが動き、PDF出力・メール添付・Excel入出力まで含めて仕様どおりに動きます。**

- 現場（タブレット 768×1024）：1-1〜5-3 の24画面
- 事務局（PC 1600×900）：K-1〜K-7 の11画面 ＋ K-8 確認事項マスタ

---

## 前提と方針

本番環境が **さくらのレンタルサーバ「ビジネス」** という制約から、次の方針で作っている。

- **PHP + PDO + 素のJavaScript のみ**。Composer / Node / ビルド工程なし → FTPで上げれば動く
- DBは **本番 MySQL / 開発 SQLite** を同じ `app/schema/schema.sql` から作る（`{{PK}}` `{{TAIL}}` をドライバごとに置換）。MySQL（MariaDB 12.3）でも通し試験 422 本すべて通ることを確認済み
- PDFは **TCPDF を同梱**（`app/vendor/tcpdf`、IPAexフォント埋め込み）。画面のA4（HTML/CSS）と同じデータから作るので見た目がずれない
- Excel（.xlsx）の読み書きは `app/lib/Xlsx.php`（zip 拡張だけ）。PhpSpreadsheet は不要
- メールは PHP の `mail()` に MIME を組んで PDF を添付（`app/lib/Mailer.php`）。さくらの sendmail でそのまま送れる
- 依存ライブラリを足さないので、共用サーバーで「動かない」が起きにくい

---

## 動かし方（開発機）

```
setup.cmd     DBを作り直してデモデータを投入（交換部品10,000件を含む）
serve.cmd     開発サーバ起動 → http://localhost:8080
test.cmd      通し試験をまとめて流す（サーバ起動中に）
```

開発機の `php.ini` は変更していない。必要な拡張（pdo_sqlite / gd）は
スクリプト内で `php -d extension=...` として読み込んでいる。

| URL | 内容 |
|---|---|
| `/login` | ユーザーサイト（協力会社・点検作業者） |
| `/admin/login` | 管理者サイト（事務局） |
| `/_dev` | 開発中の確認用インデックス（`debug=false` で無効） |

### デモ用アカウント

| 用途 | ID | パスワード |
|---|---|---|
| 協力会社（メイン） | `ABCDE0001` | `pass1234` |
| 協力会社（別会社） | `ABCDE0002` | `pass1234` |
| 事務局（管理者） | `admin` | `admin1234` |

デモの進めかたは [docs/demo.md](docs/demo.md)。

### オンラインのデモ（Vercel）

**https://work-report-system-delta.vercel.app** に、上と同じIDで入れます。

Vercel は PHP を標準では動かさないため、有志のランタイム（`vercel-php`）を `vercel.json` で
指定し、入口を `api/index.php` に置いている。また Vercel は書き込めないサーバーなので、
`app/config.vercel.php` が DB・サイン画像・控えの置き場所を `/tmp` に差し替え、
`data/demo.sqlite` を起動のたびにそこへ複製している。

> **入力したデータはしばらくすると消えます。** クライアントに見せるためのデモ専用。
> 本番はクライアント指定のさくらのレンタルサーバ（[docs/deploy.md](docs/deploy.md)）。

---

## ディレクトリ

```
public/            → さくらの ~/www に置く中身
  index.php          フロントコントローラ
  .htaccess          mod_rewrite / セキュリティヘッダ
  sw.js              Service Worker（画面を端末に取り込む）
  offline.html       圏外で未取得の画面を開いたときの案内
  assets/css         app.css / admin.css / sheet.css（A4）
  assets/js          app.js / offline.js / mic.js / sign.js
app/               → ~/www の外に置く（Web公開しない）
  bootstrap.php      起動処理（セッション・DB・ヘッダ）
  config.php         設定（config.local.php で上書き）
  routes.php         URL定義
  schema/schema.sql  スキーマ（MySQL/SQLite共用）
  lib/               Database / Router / Auth / Report / InternalReport / Sync / Pdf / Mailer / Xlsx
  views/             レイアウトと画面（sheet/ が画面用A4、pdf/ がPDFの型紙）
  vendor/tcpdf/      TCPDF 6.11.4 と IPAex フォント（同梱。Composer 不要）
  controllers/
data/              → SQLite・署名画像・PDF・控え・dry_run のメール（Web公開しない）
tools/             → migrate.php / seed.php / preflight.php
tests/             → 通し試験（run.php でまとめて実行）
docs/              → deploy.md（設置手順）/ demo.md（デモの進めかた）
```

---

## 主な作り

### オフライン（概要書「重要な検討事項の一つ」）

```
入力するたび        → 端末（IndexedDB）に控える
圏外で「つぎへ」    → 送信内容を端末の送信箱に溜めて次の画面へ
                      （画面はService Workerが取り込んだものを表示）
電波が戻った        → 溜めた順にサーバーへ送信 → 帯が緑に戻る
```

ヘッダー下の帯が3状態で変わる（サーバー送信済み／端末保存中／オフライン）。
溜めた操作には使い捨ての受付番号（`op_id`）を付け、サーバー側の `sync_ops` で
一度しか処理しない（**同じデータが二重登録されない**）。

通信があるときは介入せず、普通のフォーム送信のまま動く。

### PDF

本物のPDFは `app/lib/Pdf.php`（TCPDF）が `app/views/pdf/report.php`・`pdf/internal.php` の型紙から作る。

| URL | 内容 |
|---|---|
| `/report/{id}/pdf` | 客先提出用PDF（`?dl=1` で保存。作るたび `data/pdf/report_{No}.pdf` を更新） |
| `/report/{id}/internal/pdf` | 社内用PDF（同様に `internal_{No}.pdf`） |
| `/admin/report/{id}/pdf`・`/internal-pdf` | 管理者用（K-2 の●） |

メール送信（2-10）はこのPDFを添付し、送った分は `report_{No}_{日時}.pdf` として残す（`mail_logs.attachment`）。
`mail.dry_run = true` のときは配信せず `data/tmp/mail/*.eml` に送るはずだった内容を残す。

画面で見るA4は `app/views/sheet/report.php`・`sheet/internal.php`（HTML/CSS、`sheet.css` が 210×297mm）。
どちらも同じ `Report::sheetData()` から作る。載る量に応じて `d1 / d2 / d3` の3段階で文字を詰め、1枚に収める
（報告事項を30行に増やしてもPDFは1ページに収まることを試験で確認）。

### 概要書から変えたところ

| 箇所 | 概要書 | 実装 | 理由 |
|---|---|---|---|
| ID/パスワード保持 | パスワードを保持 | 使い捨ての合鍵をCookieに置く | 紛失時に他病院の報告書まで見られないように |
| 交換部品マスタの取込 | 書き込む前にDBをクリア | 部品名を鍵にした差分反映＋自動バックアップ | ファイル1つの間違いで1万件消えるのを防ぐ／過去の報告書との紐付けを保つ |
| PDF生成 | Excelテンプレートに流し込む | 同じ体裁を TCPDF（同梱）で描く | 共用サーバーに Excel→PDF の変換ソフトを置けない。フォント埋め込みで見た目が端末に依らない |
| 確認事項の文言 | 3〜5 が未定 | K-8 確認事項マスタで事務局が直せる | 文言が決まったあとも改修なしで変えられる |
| マスタの削除 | （記載なし） | 消さずに隠す | 過去の報告書に載っている名前を残すため |
| 交換部品・作業者・機種名 | （記載なし） | ヨミガナ列を追加 | これが無いと50音順に並ばない |

---

## 通し試験

```
test.cmd                  全部
php tests/run.php admin   1本だけ
```

各試験の前にデモデータを入れ直すので、前の結果を引きずらない。

| 試験 | 内容 | 件数 |
|---|---|---|
| `wizard` | 報告書作成 2-1〜2-6 | 36 |
| `sync` | オフライン・再送（op_id） | 23 |
| `output` | 完了・A4・PDF・印刷・メール添付 | 81 |
| `list` | 一覧・マイページ | 71 |
| `internal` | 社内用報告書 4-1〜4-8（PDF含む） | 87 |
| `admin` | 管理者サイト K-1〜K-8（Excel入出力含む） | 124 |
| | **合計** | **422** |

SQLite・MySQL（MariaDB 12.3）の両方で全件通過。PDF は実際に描画して1ページに収まること、
メールは試験用の SMTP サーバーに実際に送って添付が同一バイトで届くこと、
Excel は実際の Excel で開けること・Excel が保存したファイルを読めることを確認している。

---

## 本番へ

設置手順は [docs/deploy.md](docs/deploy.md)。

置いたあと SSH で一度だけ：

```
php tools/preflight.php
```

置き場所・書き込み権限・DB接続・設定の消し忘れをまとめて確認する。

---

## 進め方

| Phase | 内容 | 状態 |
|---|---|---|
| 1 | 土台（構成・DB設計・デモデータ・デザイン） | 完了 |
| 2 | ログイン（ID保持・3回ロック）／ダッシュボード | 完了 |
| 3 | 報告書作成 2-1〜2-6 | 完了 |
| 4 | マイク入力・オフライン下書き・同期バッジ | 完了 |
| 5 | 完了・プレビュー・印刷・メール送信（A4再現） | 完了 |
| 6 | 報告書一覧・マイページ（3／5-1〜5-3） | 完了 |
| 7 | 社内用報告書（4-1〜4-8） | 完了 |
| 8 | 管理者サイト（K-1〜K-7／Excel・CSV入出力） | 完了 |
| 9 | デモ台本・デプロイ手順・試験の整備 | 完了 |
| 10 | 本物のPDF（TCPDF）・メール添付・Excel入出力・K-8 確認事項マスタ・MySQL通し試験 | 完了 |

---

## クライアントに確認したいこと

1. 現場のタブレットは iPad か Android か（音声認識の動き方が変わる）
2. 交換部品のExcelに **ヨミガナ列**を足せるか
3. パスワードの「半角英数8文字」は、ちょうど8文字か8文字以上か（いまは8文字以上）
4. メール件名の既定値「作業者完了報告書」は誤記か（いまは「作業完了報告書」）
5. 確認署名の「確認事項3〜5」の文言（いまは仮の文言）

## 実機で見ておきたいこと

自動試験では確かめきれない3点。

1. **A4が1枚に収まるか** — プレビューで `?guide=1`、または Ctrl+P
2. **オフラインの動き** — 開発ツールの Network を Offline にして 2-1〜2-5 を進む
3. **マイク入力** — 実際のタブレットで。HTTPS が要る

---

## つくった人

ブラックにゃー（blacknyaa）— 大阪のフリーランスAIエンジニアです。生成AI×Web開発を軸に、業務システムとWebサイトを受託で作っています。

共用サーバーのような制約の多い環境でも動かしきるのが得意です。このリポジトリも、さくらのレンタルサーバで動かす前提から逆算して、Composer も Node も使わない構成にしています。

| | |
|---|---|
| ランサーズ | [ブラックにゃー (Ponta-0363)](https://www.lancers.jp/profile/Ponta-0363) |
| note | [note.com/blacknyaa](https://note.com/blacknyaa) |
| Qiita | [qiita.com/blacknyaa](https://qiita.com/blacknyaa) |
| Zenn | [zenn.dev/blacknyaa](https://zenn.dev/blacknyaa) |
| YOUTRUST | [youtrust.jp/users/blacknyaa](https://youtrust.jp/users/blacknyaa) |
| GitHub | [github.com/blacknyaa](https://github.com/blacknyaa) |

お仕事のご相談は、ランサーズ経由でも直接でも受けています。NDAも対応します。
