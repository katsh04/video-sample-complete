# 手順1: Roomを作成

この手順ではビデオチャット（ミーティング）が行われるRoomを作成します。

## 1-1. Twilio Functionsのハンドラを非同期 functionに変更

[Twilio Nodeヘルパーライブラリー](https://jp.twilio.com/docs/libraries/node)を利用し、[Room REST API](https://jp.twilio.com/docs/video/api/rooms-resource)を呼び出しRoomを作成できます。ただし、Room一覧の表示や作成は非同期で実行されます。そのため、まずはTwilio Functionsのハンドラを非同期に設定します。

`/functions/video-token.js`をコードエディタで開き、ハンドラのfunctionに`async`キーワードを追加します。

```js
//非同期に設定
exports.handler = async function(context, event, callback) {
    // ...
}
```
この結果、function内部で`await`キーワードが利用できるようになりました。

## 1-2 Twilioクライアントを取得

Twilio Functionsのハンドラに引数として渡されるcontextからTwilio Clientを取得できます。
`const ROOM = 'myroom';`の後に次のコードを追加します。

```js
exports.handler = async function(context, event, callback) {
  /*
   * Change these values for your use case
   * REMINDER: This identity is only for prototyping purposes
   */
  const IDENTITY = event.identity;
  const ROOM = 'myroom';

  const { ACCOUNT_SID } = context;

  // ここにコードを追加
  // Twilioクライアントを取得
  const client = context.getTwilioClient();
  let room;

  // ...省略
});
```

## 1-3 ビデオチャットルームをuniqueNameで検索

ビデオチャットルームは`uniqueName`という属性で検索できます。今回は`list`メソッドを利用し、進行中（`in-progress`）状態かつ、`myroom`というユニーク名を持つRoomを検索します。`try`以降を追加してください。
```js
// ... 省略

// Twilioクライアントを取得
const client = context.getTwilioClient();
let room;

// ここからのコードを追加
try {
  // 進行中のビデオチャットルームを一意の名前で検索
  let rooms = await client.video.rooms.list({
    status : 'in-progress',
    uniqueName: ROOM
  });
}
catch(error) {
  // エラーの場合
  console.error(error);
  callback(error);
}
```

# 1-4 検索したビデオチャットルームの存在確認と作成

先ほど検索したRoomが存在するかを確認します。存在しない場合は新たに作成し`room`変数に格納します。

```js
// Twilioクライアントを取得
const client = context.getTwilioClient();
let room;
try {
  // 進行中のビデオチャットルームを一意の名前で検索
  let rooms = await client.video.rooms.list({
    status : 'in-progress',
    uniqueName: ROOM
  });
  // 存在確認
  // ここからコードを追加
  // ビデオルームが存在するかを確認
  if (rooms.length)
    room = rooms[0];
  else {
    // 存在しない場合は作成
    room = await client.video.rooms.create({ 
      uniqueName: ROOM,
      type: 'go'
    });
  }// ここまで  
}
catch(error) {
  console.error(error);
  callback(error);
}
```

## 1-4. テスト実行

再度下記のコードを利用し実行結果が想定通り得られているか確認します。

```
http://localhost:3000/video-token?identity=<任意のユーザー名>
```

作成されたルームは[Programmable Videoコンソール](https://jp.twilio.com/console/video/dashboard)で確認できます。ただし、反映までにしばらく時間が必要です。

これでアクセストークン生成リクエスト時、同時にビデオチャットRoomを作成できるようになりました。

ここまでのコードは次のようになります。
```js
/**
 *  Video Token
 *
 *  This Function shows you how to mint Access Tokens for Twilio Video. Please note, this is for prototyping purposes
 *  only. You will want to validate the identity of clients requesting Access Token in most production applications and set
 *  the identity when minting the Token.
 *
 *  Pre-requisites
 *   - Create an API Key (https://www.twilio.com/console/runtime/api-keys)
 */

exports.handler = async function (context, event, callback) {
  /*
   * Change these values for your use case
   * REMINDER: This identity is only for prototyping purposes
   */
  const IDENTITY = event.identity;
  const ROOM = 'myroom';

  const { ACCOUNT_SID } = context;

  // Twilioクライアントを取得
  const client = context.getTwilioClient();
  let room;

  // ここからのコードを追加
  try {

    // 進行中のビデオチャットルームを一意の名前で検索
    let rooms = await client.video.rooms.list({
      status : 'in-progress',
      uniqueName: ROOM
    });
    // ここからコードを追加
    // ビデオルームが存在するかを確認
    if (rooms.length)
      room = rooms[0];
    else {
      // 存在しない場合は作成
      room = await client.video.rooms.create({ 
        uniqueName: ROOM,
        type: 'go'
      });
    }// ここまで  
  }
  catch(error) {
    // エラーの場合
    console.error(error);
    callback(error);
  }

  // set these values in your .env file
  const { API_KEY, API_SECRET } = context;

  const { AccessToken } = Twilio.jwt;
  const { VideoGrant } = AccessToken;

  const grant = new VideoGrant();
  grant.room = ROOM;

  const accessToken = new AccessToken(ACCOUNT_SID, API_KEY, API_SECRET);

  accessToken.addGrant(grant);
  accessToken.identity = IDENTITY;

  const response = new Twilio.Response();

  /*
   * Uncomment these lines for CORS support
   * response.appendHeader('Access-Control-Allow-Origin', '*');
   * response.appendHeader('Access-Control-Allow-Methods', 'GET');
   * response.appendHeader('Access-Control-Allow-Headers', 'Content-Type');
   */

  response.appendHeader('Content-Type', 'application/json');
  response.setBody({ token: accessToken.toJwt(), room: ROOM });
  callback(null, response);
};

```



## 次のセクション

- [ビデオチャットに参加](../04-Join-Video-Chat/00-Overview.md)