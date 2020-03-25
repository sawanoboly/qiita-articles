<!-- too_old -->
> **この記事は最終更新から1年以上経過しています。** 気をつけてね。

通常Xcodeのメニューから呼び出すiOSシミュレータを、Automaterワークフローを使って直接起動のショートカットを作成する。

Automater > ワークフロー作成 からシェルスクリプトの実行を追加して中身を下記にする。
OSXやXcodeの環境でパスが違うのでどちらにあるかは要確認。

Xcode4.4 (mountain lion用?)

```シェル：bin/bash
"/Applications/Xcode.app/Contents/Developer/Platforms/iPhoneSimulator.platform/Developer/Applications/iPhone Simulator.app/Contents/MacOS/iPhone Simulator"
```

Xcode4.3以前？

```シェル：bin/bash
"/Developer/Platforms/iPhoneSimulator.platform/Developer/Applications/iPhone Simulator.app/Contents/MacOS/iPhone Simulator"
```


アイコンもショートカット先の`Resources`の下から取ってくればOK。
