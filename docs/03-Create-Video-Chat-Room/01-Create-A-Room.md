# 手順1: Roomを作成

この手順ではビデオチャット（ミーティング）が行われるRoomを作成します。

## 1-1. Twilio Functionsのハンドラを非同期 functionに変更

[Twilio Nodeヘルパーライブラリー](https://jp.twilio.com/docs/libraries/node)を利用し、[Room REST API](https://jp.twilio.com/docs/video/api/rooms-resource)を呼び出しRoomを作成できます。ただし、Room一覧の表示や作成は非同期で実行されます。

そのため、Twilio Functionsのハンドラを非同期に設定します。

`/functions/video-token.js`をコードエディタで開き、ハンドラのfunctionに`async`キーワードを追加します。

```js
exports.handler = async function(context, event, callback) {
    // ...
}
```
この結果、function内部で`await`キーワードが利用できるようになりました。

## 1-2 Twilioクライアントを取得

Twilio Functionsのハンドラに引数として渡されるcontextからTwilio Clientを取得できます。
`const ROOM = 'myroom';`の後に次のコードを追加します。

```js
//Twilioクライアントを取得
const client = context.getTwilioClient();
let room;
```

## 1-3 ビデオチャットルームをuniqueNameで検索

ビデオチャットルームは`uniqueName`という属性で検索できます。今回は`list`メソッドを利用し、進行中（`in-progress`）状態かつ、`myroom`というユニーク名を持つRoomを検索します。`try`以降を追加してください。
```js
//Twilioクライアントを取得
const client = context.getTwilioClient();
let room;
try {
  // 進行中のビデオチャットルームを一意の名前で検索
  let rooms = await client.video.rooms.list({
    status: 'in-progress', 
    uniqueName: ROOM
  });

  // 存在確認
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
//Twilioクライアントを取得
const client = context.getTwilioClient();
let room;
try {
  // 進行中のビデオチャットルームを一意の名前で検索
  let rooms = await client.video.rooms.list({
    status: 'in-progress', 
    uniqueName: ROOM
  });
  if (rooms.length)
    room = rooms[0];
  else {
    // 存在しない場合は作成
    room = await client.video.rooms.create({ 
      uniqueName: ROOM,
      type: 'group'
    });
  }  
}
catch(error) {
  console.error(error);
  callback(error);
}
```
## 1-5. テスト実行

今回のハンズオンでは使用しませんが、クライアント側に作成、あるいは取得したRoomの`SID`を返すように変更もしておきましょう。

```js
// tokenのほかにビデオチャットルーム名、ルームSIDも返す。
response.setBody({ 
  token: accessToken.toJwt(), 
  room: ROOM, 
  roomSid: room.sid });
```

再度下記のコードを利用し実行結果が想定通り得られているか確認します。

```
http://localhost:3000/video-token?user=<任意のユーザー名>
```

これでアクセストークン生成リクエスト時、同時にビデオチャットRoomを作成できるようになりました。


## 関連リソース


## 次のハンズオン

- [ハンズオン: ビデオチャットに参加](/docs/04-Join-Video-Chat/00-Overview.md)