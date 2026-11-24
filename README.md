# media.file — Muse GadgetでMP3を鳴らす

media.file は、MuseのVMにあるMP3をESP32のガジェットで再生するLinkコマンドです。`path` にMP3の場所を指定すると、ガジェットがファイルを取得して再生します。コマンドの応答は、再生が始まった時点で返ります。

## どうして作ったか

ガジェットから話しかけたときに、Museの返答をガジェットのスピーカーで聞けるようにしたかったからです。Muse自身が返答の音声を作れるので、外部のTTSは不要です。 Museが作ったMP3を media.file で指定すれば、ガジェットで鳴らせます。

標準SDKにはMP3を鳴らすテスト機能があったので、その処理を使って、Museから「このMP3を鳴らして」と頼めるコマンドを追加しました。
*コマンド追加の形なので、SDK 自体が更新されても、修正の影響は追加分の中に収まります。*

## 使用例

エージェントから好きなタイミングで音声を送れるので、アラームやスキル完了の通知、ボイスメッセージの再生にも使えます。たとえば、エージェントにこう仕込んでおきます。

このスキルが終わったら、「作業が終わりました」と音声を作って、ガジェットに送って。

トークの返答を音声で返す仕込み例は、別ファイルの muse_setup.md にあります。

path指定となっていますが、実際はpathは不要です。museの内部に格納されているファイルは、ユーザーは意識することはいりません。

またユーザーが作成した任意のMP3を、museに置けば(これもユーザーが気にすることない）、それを送信してと指示すれば、ガジェットに送ります。

## パッチの当て方

対象は Muse Gadget SDK の upstream main の ded4222（2026-11-21、#490）です。`ded4222` のクリーンなツリーを用意し、SDKのルートで実行します。v1を適用済みなら、別の作業ツリーを使ってください。

```
git apply media_file.patch
```

ボードの sdkconfig オーバーレイに CONFIG_HOMEHUB_MP3_COMMAND=y を追加し、ビルドして書き込みます。初期設定では無効です。

## 追加・変更内容

対象は8ファイルです。既存行の書き換えは muse_voice.c の1行です。

| ファイル | 追加・変更内容 |
|---|---|
| main/muse_media.c / main/muse_media.h | 新規。 GET /fs/raw に Range を付けてMP3を取得し、標準の mp3_selftest() をもとに16 kHz・モノラルのPCMに変換します。PCMと常駐ワーカーのスタック（32 KB）はPSRAMに置きます。ワーカーのキューの深さは1です。 |
| components/muse/muse_voice.c / components/muse/muse_voice.h | PCMを受け取る muse_voice_request_pcm() と、voiceタスクでの再生処理を追加。既存の rest 条件に !s_media_play を追加しました。 |
| main/noise_control.cpp | display.draw_file の直後に media.file を登録。 |
| main/app.c | media.file のコマンド分岐と完了コールバックを追加。 |
| main/CMakeLists.txt | muse_media.c と minimp3 への依存を追加。 |
| main/Kconfig.projbuild | HOMEHUB_MP3_COMMAND を追加。初期設定は n。 |

## 注意事項

- トークのターン中のテキスト表示とは同期しません。単独で MP3 を再生するコマンドです。
- ファイルサイズは最大512 KB、再生時間は約60秒までです。コマンドのタイムアウトは60秒です。ファイルサイズや再生時間の上限を超えると、処理を打ち切るか decode_failed を返します。
- トーク中にコマンドを受け取った場合は、そのターンが終わってから再生します。
- 再生中に次のMP3を送ると decode_failed になります。デコード時に最大60秒分のPCM用領域（1.92 MB）を先に確保するため、再生中はPSRAMの空きが足りません。
- 音声機能が無効な構成（MUSE_ENABLED 無効）で media.file を有効にすると、ビルド時にリンクエラーになります。
