# 3. SORACOM Flux アプリを作成して動かす

1. ユーザーコンソールから「SORACOM Flux」>「Flux アプリ」を選択します。
   ![SORACOM Fluxの画面を開く](./image/03-01.png)

   アプリを新しく作成します。「新しいFluxアプリを作成する」を選択します。
   ![アプリを新規作成する](./image/03-02.png)

   最初から作成を選択し、以下のように項目を入力して作成します。
   - アプリ名例：`gototi-info-button`（SIM グループと同じく自分の名前を付ける）
   - 概要例：`SORACOM UG Explorer 2026ハンズオン用アプリ`
     ![アプリ名・概要入力](./image/03-03.png)

2. 空のアプリが作成され、キャンバス（チャネル）が表示されます

## 3-1. イベントソースを追加する

1. 「チャネルを作成する」＞「イベントソース」を追加し、種類で **IoT デバイス（SORACOM Air for セルラー）** を選択する
   ![チャネルを作成する](./image/03-04.png)

   ![IoTデバイスをイベントソースとして選択する](./image/03-05.png)

2. SIMグループは「explorer-handson」を選択する。イベントソースを作成する。
   ![イベントソースの作成](./image/03-06.png)

   イベントソースを追加すると、以下のようにキャンバスに表示されます。
   ![キャンバスに表示されたイベントソース](./image/03-06-2.png)

3. イベントソースが作成出来たら、データを取得するアクションを追加する。イベントソースの「アクション」タブから「＋アクションを追加」をクリックする。
   ![アクションの作成](./image/03-07.png)

4. アクションの種類で **SORACOM API** を選択する。

   ![SORACOM APIアクション](./image/03-08.png)

5. 以下のように設定する。

   **基本設定**
   - 名前例：`get-harvest-data`
   - 有効：ON

   **アクションの実行条件**：空のまま（シングル・ダブル・長押しのどのクリックでも動かす）

   **CONFIG**

   | 項目                        | 設定値                                                                 |
   | --------------------------- | ---------------------------------------------------------------------- |
   | API                         | `DataEntry:getDataEntries`                                             |
   | URL                         | `/v1/data/Subscriber/${event.context.resource.imsi}?sort=desc&limit=1` |
   | HTTP ボディ                 | （空のまま）                                                           |
   | 注意点                      | 内容を確認してチェックを入れる                                         |
   | API を実行する SAM ユーザー | 「既存の SAM ユーザーから選択する」を選択                              |
   | SAM ユーザー                | `FluxAction-SoracomApi-getDataEntries1`                                |
   - URL は API を選択すると `/v1/data/{resource_type}/{resource_id}` と表示されるので、`{resource_type}` を `Subscriber`、`{resource_id}` を `${event.context.resource.imsi}`に書き換え、末尾に `?sort=desc&limit=1`（新しい順に 1 件だけ取得）を付けます。(CONFIGのURLの項目をコピペして貼りなおして構いません)

   ![SORACOM APIアクション1](./image/03-09.png)
   ![SORACOM APIアクション2](./image/03-10.png)

   > ⚠️ `limit=1` を必ず付けてください。SORACOM API アクションはレスポンスが 32,768 バイトを超えるとエラーになります。

   ![SAMユーザーの選択](./image/03-10-2.png)

   > ⚠️ Flux アプリを削除すると、アタッチしている SAM ユーザー（`FluxAction-SoracomApi-getDataEntries1`）も一緒に削除されます。削除を避けたい場合は、事前に SAM ユーザーの「認証設定」>「SORACOM リソース認証」を解除しておいてください。

6. **OUTPUT** を以下のように設定する
   - 「アクションのアウトプットを別のチャネルに送信する」：ON
   - 送信先チャネル：「新しくチャネルを作成する」→ チャネル名 `harvest-data-output`
   - [高度な設定] →「データを変換する」：ON
     - Content-Type：`application/json`
     - Content：

       ```json
       {
         "clickTypeName": "${event.payload.clickTypeName}",
         "time": "${result[0].time}",
         "lat": "${result[0].metadata.location.lat}",
         "lon": "${result[0].metadata.location.lon}"
       }
       ```

   ![SORACOM APIのOutputの設定](./image/03-12.png)

   API のレスポンス（`${result}`）は次のような配列で返ってくるため、先頭の要素 `[0]` から位置情報だけを取り出して次のチャネルに渡しています。

   ```json
   [
     {
       "time": 1760000000000,
       "contentType": "application/json",
       "content": "{\"clickType\":1,\"clickTypeName\":\"SINGLE\",\"batteryLevel\":1,\"binaryParserEnabled\":true}",
       "metadata": {
         "locationQueryResult": "success",
         "location": { "lat": "33.88", "lon": "130.88" }
       }
     }
   ]
   ```

7. 「作成する」をクリックして保存する

8. 動作確認：ボタンを 1 回押し、Flux アプリの「実行履歴」で SORACOM API アクションが成功していること、`harvest-data-output` チャネルの「メッセージ履歴」に `lat` / `lon` が入ったデータが届いていることを確認する

   > ⚠️ ボタンのデータは Harvest Data と Flux に同時に送られるため、タイミングによっては Harvest Data への保存が間に合わず、**1 つ前に押したときのデータ**が取得されることがあります。`time`（UNIX 時間・ミリ秒）がボタンを押した時刻と大きくずれている場合は、もう一度押して確認してください。
   >
   > ⚠️ `lat` / `lon` が空の場合は、2 章の SIM グループ設定で簡易位置測位が ON になっているか、Harvest Data の画面で位置情報が付与されているかを確認してください。

9. SORACOM APIアクションを追加すると、以下のようにキャンバスに表示されます。

   ![キャンバスに表示されたSORACOM APIアクション](./image/03-12-2.png)

## 3-2. AI アクションでご当地情報を生成する

1. 3-1 で作成した `harvest-data-output` チャネルを開き、「アクション」タブから「＋アクションを追加」→ **AI アクション** を選択
   ![AIアクションの追加1](./image/03-13.png)

   ![AIアクションの追加2](./image/03-14.png)

2. モデルを選択（例では`Amazon Bedrock – Amazon Nova Lite`を選択していますが、他のモデルを選んでも構いません。様々なモデルで遊んでみましょう✨）

   ![AIモデル](./image/03-15.png)

3. **プロンプト** を設定

### プロンプト例

```text
あなたは日本各地のご当地情報にくわしい旅のガイドです。
以下の緯度・経度は、IoTボタンが押されたおおよその場所です（数km程度の誤差があります）。

緯度: ${payload.lat}
経度: ${payload.lon}

この場所に近い市区町村を推定し、次の形式で日本語で答えてください。

📍 推定エリア：（都道府県・市区町村）
🍜 ご当地グルメ：1〜2個（ひとことコメント付き）
🏯 おすすめスポット：1〜2個
💡 ちょっとした豆知識：1つ

条件：
- 全体で400文字以内
- 確実でない情報は「〜と言われています」など断定しない表現にする
- 余計な前置きは書かない
- 改行は使わず、改行したい位置には「\n」という2文字を書く（全体を1行で出力する）
- ダブルクォーテーション（"）は使わない
```

　　![プロント](./image/03-16.png)

4. **OUTPUT** を以下のように設定する
   - 「アクションのアウトプットを別のチャネルに送信する」：ON
   - 送信先チャネル：「新しくチャネルを作成する」→ チャネル名 `ai-output`
     ![出力指定](./image/03-17.png)

5. 「作成する」をクリックして保存する

> ✅ `${payload.lat}` / `${payload.lon}` は、3-1 の手順5で「データを変換する」に設定したキー名です。

> 💡 位置精度が粗いため、「市区町村を推定して」と明示すると回答が安定します。誤差がある前提をAIに伝えておくのもポイントです。

6. AIアクションを追加すると、以下のようにキャンバスに表示されます。

   ![キャンバスに表示されたAIアクション](./image/03-16-2.png)

## 3-3. Webhook アクションで Discord に投稿する

1. 3-2 で作成した `ai-output` チャネルを開き、「アクション」タブから「＋アクションを追加」→ **Webhook アクション** を選択
   ![Webhookアクション1](./image/03-18.png)

   ![Webhookアクション2](./image/03-19.png)

2. 以下のように設定

| 項目     | 設定値                           |
| -------- | -------------------------------- |
| メソッド | `POST`                           |
| URL      | 事前準備で控えた Webhook URL     |
| ヘッダー | `Content-Type: application/json` |

3. **ボディ** を設定

```json
{
  "username": "ご当地ボタン",
  "content": "🔘 ボタンが押されました！\n\n${payload.output.text}"
}
```

　![webhookの設定](./image/03-20.png)

> ✅ AI アクションの出力は `{"output":{"text":"..."}, "usage":{...}}` の形で次のチャネルに届くため、生成された文章は `${payload.output.text}` で参照できます。
>
> ⚠️ Discord の `content` は2000文字までです。プロンプトで文字数を制限しておくと安心です。
>
> ⚠️ `${payload.output.text}` はそのままボディに埋め込まれるため、AI の出力に改行や `"` が含まれると JSON が壊れ、Discord から `400 The request body contains invalid JSON.` が返ります。3-2 のプロンプトで「改行の代わりに `\n` と書く」「`"` を使わない」と指示しているのはこのためです。エラーになった場合は、実行履歴で Webhook の入力（`body`）を確認しましょう。

4. 保存

   ![キャンバスに表示されたWebhookアクション](./image/03-20-2.png)

## 3-4. 動かしてみよう 🎉

1. ボタンを押す（シングル・ダブル・長押しのどれでも OK）
2. 数十秒ほど待つ
3. Discord のチャンネルに、ご当地情報が投稿されれば成功！
   ![投稿](./image/03-21.png)

---

[← 2. SIM グループを設定して簡易位置測位を有効にする](./02-sim-group-location.md) ｜ [目次](./README.md) ｜ [4. 後片付け →](./04-cleanup.md)
