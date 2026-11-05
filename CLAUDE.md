# sqlite-amalgamation-new

SQLite の amalgamation（`sqlite3.c` / `sqlite3.h` / `shell.c`）に、VC++ 6.0 でビルドできるようにするパッチを当てたフォーク。

## ブランチの役割

- `autobuild`: upstream の amalgamation を自動で取り込むブランチ（コミットメッセージは `autobuild`）。パッチは載せない。
- `master`: `autobuild` の上に VC6 対応パッチ（`old compiler/sdk fix` と、その後の VC6 fixes）を載せたもの。
- remote は `origin`（自分のフォーク）と `upstream`（azadkuh/sqlite-amalgamation）。**upstream へ PR を出さない**（グローバル指示どおり）。push 先は `origin` のみ。

## autobuild の更新を master に取り込む手順

1. 状況確認: `git log --oneline --graph --all`、`git log master..autobuild`。
2. バックアップ: `git branch backup/master-before-rebase master`（既にあれば別名にする）。
3. `git rebase autobuild`（master のパッチを最新の autobuild の上に載せ直す）。
4. 衝突はほぼ `sqlite3.c` に出る。upstream が該当箇所を書き直していたら upstream 側を採用し、VC6 回避が必要かどうかは手順 5〜6 で判断する。衝突マーカーが残っていないことを `grep -n '^<<<<<<<\|^>>>>>>>' sqlite3.c` で確認し、`git add sqlite3.c && GIT_EDITOR=true git rebase --continue`。
5. VC6 で実際にコンパイルして問題を洗い出す（下記）。推測で「通るはず」と言わない。
6. 見つかった問題を直して別コミットにする（コミットメッセージは英語。リポジトリの慣習）。
7. 既定オプションと、拡張を有効にした構成の両方でビルド・リンク・実行を確認する。
8. push する場合、rebase で履歴が変わるので `git push --force-with-lease origin master`。ユーザーが許可したときだけ行う。

## VC6 でのビルド確認

VC6 は `C:\Program Files (x86)\Microsoft Visual Studio\VC98\bin\` にある（`cl.exe`、`VCVARS32.BAT`）。

- Git Bash から `cl /nologo` のように `/` 始まりのオプションを渡すと Windows パスに化ける。`.bat` を Write ツールで作り、PowerShell から `cmd /c` で実行する。オプションは `-nologo` のように `-` 始まりで書く。
- `shell.c` は VC6 ではコンパイルできない。`sqlite3.h` だけを使う小さな `main()`（`test.c`）を書いてリンクする。
- 出力は cp932。`python -c "open('build.log',encoding='cp932',errors='replace').read()"` で読む。エラー行番号から `sed -n 'Np' sqlite3.c` で該当箇所を見る。
- `%ERRORLEVEL%` はパイプの後だと最後のコマンドのものになるので、`cl` の直後に `echo CL_EXIT=%ERRORLEVEL%` をログへ書く。
- 一時ファイル（`.bat`、`test.c`、`.exe`、`.obj`、ログ）はリポジトリに置かない。

`build.bat` の例（`REPO`/`OUT` は環境に合わせる）:

```bat
@echo off
call "C:\Program Files (x86)\Microsoft Visual Studio\VC98\bin\VCVARS32.BAT" >nul
set REPO=C:\D_DRIVE\MyDocuments\GitHub\sqlite-amalgamation-new
set OUT=<scratchpad>
cd /d %OUT%
cl -nologo -MT -D_WIN32_WINNT=0x0400 -I%REPO% test.c %REPO%\sqlite3.c -Fe%OUT%\t.exe -Fo%OUT%\ > %OUT%\build.log 2>&1
echo CL_EXIT=%ERRORLEVEL% >> %OUT%\build.log
```

拡張側の確認は、`cl` に次を足して同様にビルドする（既定では compile されない `rtree` などの修正を確かめるため）:

```
-DSQLITE_ENABLE_RTREE -DSQLITE_ENABLE_FTS3 -DSQLITE_ENABLE_FTS4 -DSQLITE_ENABLE_FTS5 -DSQLITE_ENABLE_GEOPOLY -DSQLITE_ENABLE_MATH_FUNCTIONS -DSQLITE_ENABLE_DBSTAT_VTAB -DSQLITE_ENABLE_STAT4 -DSQLITE_ENABLE_COLUMN_METADATA -DSQLITE_ENABLE_SESSION -DSQLITE_ENABLE_PREUPDATE_HOOK -DSQLITE_ENABLE_UNLOCK_NOTIFY
```

`test.c` は、`sqlite3_exec` で次のような SQL を流して結果を目で確認する程度でよい: 64bit 整数の境界値の insert と order by、`sum`/`total`、`json_extract`・`jsonb`（巨大整数、`0xFFFFFFFFFFFFFFFF`）、`abs(-0.0)`、`cast(1e30 as integer)`。

## VC6 を阻む典型的な要素（過去に見つかったもの）

新しい upstream コードに入りやすいので、取り込みのたびにこの観点で見る。

| 要素 | VC6 での症状 | 対処 |
|---|---|---|
| `LL` / `ULL` / `ull` 接尾辞のリテラル | C2059 など。`OptimizationEnabled` マクロ経由だと百数十件のエラーになる | `INT64_C()` / `UINT64_C()`（`sqlite3.c` 冒頭近くで MSVC 用に定義済み）、または `((u64)1)<<n` |
| `unsigned __int64` → `double` の変換 | C2520 | `(double)(sqlite3_int64)(u>>1)*2.0 + (double)(sqlite3_int64)(u&1)` のように二分する。符号付きにキャストするだけでは bit63 が立った値が負になる |
| C99 の `signbit()` など | リンク時 `_signbit` 未解決 | IEEE754 のビットを `memcpy` で読む |
| 関数名だけを返す（関数ポインタ） | 警告・エラー | `&func` と書く |
| `GetTickCount64`、SRWLOCK など新しい Windows API | 宣言なし・リンク不可 | `GetTickCount`、`_WIN32_WINNT` で SRWLOCK を切り替え、`CRITICAL_SECTION` へフォールバック |
| `TryEnterCriticalSection` | `_WIN32_WINNT<0x0400` だと未宣言 | `-D_WIN32_WINNT=0x0400` を付ける（パッチ側でも宣言を補っている） |

`NDEBUG` と `INFINITY` は `sqlite3.c` 自身が対処済みなので、冒頭で定義しない。`sqlite3.c` は `i64`/`Ui64` 系のリテラルのせいで MSVC 専用であり、GCC/Clang では通らない（同梱の `CMakeLists.txt` も同様）。
