<!doctype html>
<html lang="ja">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>五枚の画像と役名一覧</title>
  <link rel="stylesheet" href="styles.css" />
</head>
<body>
  <header>
    <h1>五枚の画像と役名を挿入して一覧表示</h1>
    <p>各スロットに画像と役名を入れて「一覧を表示」ボタンを押すと、下に一覧が作成されます。入力はローカルストレージに保存されます。</p>
  </header>

  <main>
    <section class="input-area" aria-labelledby="input-heading">
      <h2 id="input-heading">入力（5スロット）</h2>

      <form id="cards-form">
        <div class="slots" id="slots">
          <!-- スロットはscript.jsで動的に作成 -->
        </div>

        <div class="controls">
          <button type="button" id="preview-btn">一覧を表示</button>
          <button type="button" id="save-btn">保存</button>
          <button type="button" id="clear-btn">全てクリア</button>
        </div>
      </form>
    </section>

    <section class="list-area" aria-labelledby="list-heading">
      <h2 id="list-heading">一覧プレビュー</h2>
      <div id="cards-list" class="cards-list">
        <!-- プレビュー表示領域 -->
      </div>
    </section>
  </main>

  <footer>
    <small>作成: サンプルページ — 画像はブラウザのファイルを使用します</small>
  </footer>

  <script src="script.js"></script>
</body>
</html>
