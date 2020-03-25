WordPressの環境作成をCIする場合に、一応セットアップまで見ておくかというときに。

## Puppeteerをインストール

- [GoogleChrome/puppeteer: Headless Chrome Node API](https://github.com/GoogleChrome/puppeteer)

```
$ yarn install puppeteer
```

## Puppeteerスクリプト

Passwordの入力周りがJavascriptなので、waitをしておくのが無難だった。

```javascript:wpsetup.js
const puppeteer = require('puppeteer');

(async () => {
  try {
    const browser = await puppeteer.launch({
      // headless: false, // デバッグ用
      ignoreHTTPSErrors: true
    });
    const page = await browser.newPage();
    // 言語選択がデフォルトのenでよければ、いきなりstep=1に飛んでOK
    await page.goto(
      process.argv[2] + '/wp-admin/install.php?step=1',
      {waitUntil: 'networkidle0'}
    );
    await page.type('#weblog_title', 'PUPPETEER')
    await page.type('#user_login', 'admin')
    // それなりに複雑なパスワード
    await page.type('#pass1-text', 'Pass00+345+Word')
    await page.type('#admin_email', 'test@example.com')
    await page.click('#blog_public')
    await page.click('#submit')

    browser.close()
  } catch(e) {
    console.log(e)
    process.exit(1)
  }
})();
```

`node wpsetup.js https://127.0.0.1:8443`のように実行する。

実行はCircleCIでやってるので、Fromイメージには`circleci/node:8-browsers`を使いました。
なんか動かないなーというときは`DEBUG=puppeteer:session`環境変数をつかいます。
