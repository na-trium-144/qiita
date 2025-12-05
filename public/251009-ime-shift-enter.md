---
title: Shift+EnterにIMEの変換確定を割り当てる (Windows & Linux)
tags:
  - Windows
  - AutoHotkey
  - Mozc
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

日本語の変換確定を今までEnterキーで行っていたのですが、CopilotやGeminiなどのAIチャットやDiscordなどのアプリでEnterを押すと即誤送信してしまうので、困っていました。
別のキーで変換を確定できないだろうか。

## macOS

Macはなにもしなくても Shift + Enter で誤送信することなく確実に変換を確定することができます。
Shift+Enterでも誤送信してしまうアプリもあるにはあるが...(Gemini CLIとか)
ほとんどの場合これで良さそう

## Linux (Mozc)

ということであとはMac以外のPCでもShift+Enterで変換を確定させることができたら良さそうです。

自分はLinuxではMozcを使っていますが、Mozcは設定でキーバインディングを変えられます。
Mozcの設定 > キー設定の選択 の 編集... で、既存の Shift Enter のコマンドをすべて削除して

|モード|入力キー|コマンド|
|----|----|----|
|変換前入力中|Shift Enter|確定|
|変換中|Shift Enter|確定|

を追加すればok。
GUIでポチポチするのが面倒なら、キーマップをエクスポートしてdotfilesのリポジトリにでもpushしておき、2回目以降設定する際にはそれをインポートすれば良いです。

## Windows

WindowsではEnter以外にもCtrl+Mキーで変換を確定できるらしいです。(ってGeminiが言ってた)

自分はAutoHotKey v1を使っているので、Shift+EnterをCtrl+Mに変えるスクリプトを書きました。
v2でも動くかは未検証です。

ただし無条件にShift+EnterをCtrl+Mに置き換えると、Discordの改行など本当にShift+Enterを入力したい場合に使えません。
そこで[こちらの記事](https://neokixblog.wordpress.com/2018/02/24/ime%E3%81%AE%E7%8A%B6%E6%85%8B%E3%82%92autohotkey%E3%81%8B%E3%82%89%E7%9F%A5%E3%82%8B%E6%96%B9%E6%B3%95%E3%82%B8%E3%83%A3%E3%83%B3%E3%82%AD%E3%83%BC-%E3%83%8E%E3%83%B3%E3%83%97%E3%83%AD%E3%82%B0/)のコピペで現在のIMEの状態を取得し、IMEが有効の間だけShift+EnterがCtrl+Mになるようにしました。

```ahk
~+vk0D::
WinGet, vcurrentwindow, ID, A
vimestate := DllCall("user32.dll\SendMessageA", "UInt", DllCall("imm32.dll\ImmGetDefaultIMEWnd", "UInt", vcurrentwindow), "UInt", 0x0283, "Int", 0x0005, "Int", 0)
If (vimestate=1) {
    Send ^m
}
return
```

