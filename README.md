[lichess.org](http://lichess.org)
---------------------------------

It's a free online chess game focused on [realtime](http://lichess.org/games) and ease of use

It haz a [search engine](http://lichess.org/games/search),
[computer analysis](http://lichess.org/analyse/ief49lif),
[tournaments](http://lichess.org/tournament),
[forums](http://lichess.org/forum),
[teams](http://lichess.org/team),
[puzzles](http://lichess.org/training),
a weird [monitoring console](http://lichess.org/monitor),
and a [world map](http://map.lichess.org).
The UI is available in [80 languages](http://lichess.org/translation/contribute) thanks to the community.

Lichess is written in [Scala 2.11](http://www.scala-lang.org/),
and relies on [Play 2.3](http://www.playframework.com/) for the routing, templating, and JSON.
Pure chess logic is contained in [scalachess](http://github.com/ornicar/scalachess) submodule.
The codebase is fully asynchronous, making heavy use of Scala Futures and [Akka 2 actors](http://akka.io).
Lichess talks to [Stockfish 5](http://stockfishchess.org/) using a [FSM Actor](https://github.com/ornicar/lila/blob/master/modules/ai/src/main/stockfish/ActorFSM.scala) to handle AI moves and analysis.
It uses [MongoDB 2.4](http://mongodb.org) to store more than 30 million games, which are indexed by [elasticsearch](http://elasticsearch.org).
HTTP requests and websocket connections are proxied by [nginx 1.4](http://nginx.org).
New client-side features are written in [ClojureScript](https://github.com/ornicar/lila/tree/master/cljs/puzzle/src).

Join us on #lichess IRC channel on freenode for more info.
See the roadmap on https://etherpad.mozilla.org/ep/pad/view/ro.3bIwxJwTQYW/latest.

Installation
------------

> If you want to add a live chess section to your website, you are welcome to [embed lichess](http://lichess.org/developers). It's very easy.

> This project source code is open for other developers to have an example of non-trivial scala/play2/mongodb application. You're welcome to reuse as much code as you want for your projects, and to get inspired by the solutions I propose to many common web development problems. But please don't just create a public lichess clone. Instead, just [embed lichess using an &lt;iframe&gt;](http://lichess.org/developers).

> Also note that if I provide the source code, I do **not** offer support for your lichess instance. I will probably ignore any question about lichess installation and runtime issues.

## HTTP API

Feel free to use lichess API in your applications and websites.

If the resource you need is not available yet,
drop me an email at thibault.duplessis@gmail.com, and we'll discuss it.

### `GET /api/user/<username>` fetch one user

```
> curl http://en.lichess.org/api/user/thibault
```

```javascript
{
  "username": "thibault",
  "url": "http://lichess.org/@/thibault",   // profile url
  "rating": 1503,                           // global Glicko2 rating
  "progress": 36,                           // rating change over the last ten games
  "online": true,                           // is the player currently using lichess?
  "playing": "http://lichess.org/abcdefgh", // game being played, if any
  "engine": false                           // true if the user is known to use a chess engine
}
```

Example usage with JSONP:

```javascript
$.ajax({
  url:'http://en.lichess.org/api/user/thibault',
  dataType:'jsonp',
  jsonp:'callback',
  success: function(data) {
    // data is a javascript object, do something with it!
    console.debug(JSON.stringify(data));
  }
});
```

### `GET /api/user` fetch many users

All parameters are optional.

name | type | default | description
--- | --- | --- | ---
**team** | string | - | filter users by team
**nb** | int | 10 | maximum number of users to return

```
> curl http://en.lichess.org/api/user?team=coders&nb=100
```

```javascript
{
  "list": [
    {
      "username": "thibault",
      "url": "http://lichess.org/@/thibault",   // profile url
      "rating": 1503,                           // global Glicko2 rating
      "progress": 36,                           // rating change over the last ten games
      "online": true,                           // is the player currently using lichess?
      "engine": false                           // true if the user is known to use a chess engine
    },
    ... // other users
  ]
}
```

Example usage with JSONP:

```javascript
$.ajax({
  url:'http://en.lichess.org/api/user',
  data: {
    team: 'coders',
    nb: 100
  },
  dataType:'jsonp',
  jsonp:'callback',
  success: function(data) {
    // data is a javascript object, do something with it!
    console.debug(JSON.stringify(data.list));
  }
});
```

### `GET /api/game` fetch many games

Games are returned by descendant chronological order.
All parameters are optional.

name | type | default | description
--- | --- | --- | ---
**username** | string | - | filter games by user
**rated** | 1 or 0 | - | filter rated or casual games
**nb** | int | 10 | maximum number of games to return
**token** | string | - | security token (unlocks secret game data)

```
> curl http://en.lichess.org/api/game?username=thibault&rated=1&nb=10
```

```javascript
{
  "list": [
    {
      "id": "x2kpaixn",
      "rated": false,
      "status": "mate",
      "clock":{          // all clock values are expressed in seconds
        "limit": 300,
        "increment": 8,
        "totalTime": 540  // evaluation of the game duration = limit + 30 * increment
      },
      "timestamp": 1389100907239,
      "turns": 44,
      "url": "http://lichess.org/x2kpaixn",
      "winner": "black",
      "players": {
        "white": {
          "userId": "thibault"
          "rating": 1642,
          "analysis": {
            "blunder": 1,
            "inaccuracy": 0,
            "mistake": 2
          }
        },
        "black": ... // other player
      }
    },
    {
      ... // other game
    }
  ]
}
```

(1) All game statuses: https://github.com/ornicar/scalachess/blob/master/src/main/scala/Status.scala#L16-L25

### `GET /api/analysis` fetch many analysis

This API requires a secret token to work.
Analysis are returned by descendant chronological order.
All parameters are optional.

name | type | default | description
--- | --- | --- | ---
**token** | string | - | security token
**nb** | int | 10 | maximum number of analysis to return

```
> curl http://en.lichess.org/api/analysis?nb=10
```

```javascript
{
  "list": [
    {
      "analysis": [
        {
          "eval": -26, // board evaluation in centipawns
          "move": "e4",
          "ply": 1
        },
        {
          "eval": -8,
          "move": "b5",
          "ply": 2
        },
        {
          "comment": "(-0.08 → -0.66) Inaccuracy. The best move was c4.",
          "eval": -66,
          "move": "Nfe3",
          "ply": 3,
          "variation": "c4 bxc4 Nfe3 c5 Qf1 f6 Rxc4 Bb7 b4 Ba6"
        },
        // ... more moves
      ],
      "game": {
        // similar to the game API format, see above
      },
      "uci": "e2e4 e7e5 d2d4 e5d4 g1f3 g8f6" // UCI compatible game moves
    }
  ]
}
```

### `GET /api/puzzle/<id>` fetch one puzzle

```
> curl http://en.lichess.org/api/puzzle/23045
```

```javascript
{
  "id": 16177,
  "url": "http://lichess.org/training/16177",         // URL of the puzzle
  "color": "black",                                   // color of the player
  "position": "6NK/5k2/2r5/2n3PP/8/8/8/8 w - - 7 39", // FEN initial position
  "solution": ["c6h6", "g5h6", "c5e6", "h8h7", "e6g5",
               "h7h8", "f7f8", "h6h7", "g5f7"],       // solution moves
  "rating": 1799                                      // puzzle glicko2 rating
}
```

### `GET /api/puzzle/daily` fetch daily puzzle

```
> curl http://en.lichess.org/api/puzzle/daily
```

```javascript
{
  "id": 16177,
  "url": "http://lichess.org/training/16177",         // URL of the puzzle
  "color": "black",                                   // color of the player
  "position": "6NK/5k2/2r5/2n3PP/8/8/8/8 w - - 7 39", // FEN initial position
  "solution": ["c6h6", "g5h6", "c5e6", "h8h7", "e6g5",
               "h7h8", "f7f8", "h6h7", "g5f7"],       // solution moves
  "rating": 1799                                      // puzzle glicko2 rating
}
```

Credits
-------

See the [lichess Thanks page](http://lichess.org/thanks)
