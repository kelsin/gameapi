# Game API

[![license](https://img.shields.io/github/license/kelsin/gameapi?logo=data%3Aimage%2Fsvg%2Bxml%3Bbase64%2CPHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSIyNCIgaGVpZ2h0PSIyNCIgdmlld0JveD0iMCAwIDI0IDI0IiBmaWxsPSJub25lIiBzdHJva2U9IiNmZmYiIHN0cm9rZS13aWR0aD0iMiIgc3Ryb2tlLWxpbmVjYXA9InJvdW5kIiBzdHJva2UtbGluZWpvaW49InJvdW5kIj48cGF0aCBzdHJva2U9Im5vbmUiIGQ9Ik0wIDBoMjR2MjRIMHoiIGZpbGw9Im5vbmUiLz48cGF0aCBkPSJNNyAyMGwxMCAwIi8%2BPHBhdGggZD0iTTYgNmw2IC0xbDYgMSIvPjxwYXRoIGQ9Ik0xMiAzbDAgMTciLz48cGF0aCBkPSJNOSAxMmwtMyAtNmwtMyA2YTMgMyAwIDAgMCA2IDAiLz48cGF0aCBkPSJNMjEgMTJsLTMgLTZsLTMgNmEzIDMgMCAwIDAgNiAwIi8%2BPC9zdmc%2B&logoColor=%23fff&color=%23750014)](https://github.com/kelsin/gameapi?tab=MIT-1-ov-file#readme)

A Game-Agnostic Web API for Turn Based Games.

## Definition

This repo is meant to define and contain a reference implementation of an API
for turn based game development.

The API supports HTTP REST requests with SSE for realtime events, and
Websockets. Each method uses the same JSON messages and schemas for
communication.

This API supports any game clients but only supports all players playing a
single game to be using the same client, on the same version.

## Example Session

Imagine we have a game of
[Root](https://ledergames.com/products/root-a-game-of-woodland-might-and-right)
with 3 players. We'll name them Ada [^1], Betty [^2], and Carol [^3]. They are
in the middle of a game. This section is meant to explain how the game api works
with examples of what messages might look like.

> [!NOTE]
> This example was written prior to the full API spec being designed. Messages
> and their schemas are not final.

[^1]: [Ada Lovelace](https://en.wikipedia.org/wiki/Ada_Lovelace) was the
    designer of the first algorithm.
[^2]: [Betty Holberton](https://en.wikipedia.org/wiki/Betty_Holberton) invented
    breakpoints in compute debugging and was one of the six original programmers
    of ENIAC.
[^3]: [Carol Shaw](https://en.wikipedia.org/wiki/Carol_Shaw) was one of the
    first female video game designers.

### Connecting

Ada opens her browser and loads up the URL to the game client she is using. The
client should eventually hit the player URL of the api:
`/games/<game_id>/players/<player_ada_id>` which returns some simple metadata
about what to do next:

``` json
{
  "server": {
    "post": "/games/<game_id>/players/<player_ada_id>",
    "sse": "/games/<game_id>/players/<player_ada_id>/events",
    "ws": "/games/<game_id>/players/<player_ada_id>/ws",
  }
}
```

If the server didn't support
[SSE](https://en.wikipedia.org/wiki/Server-sent_events) than the `post` and
`sse` fields wouldn't be there. If the server didn't support
[Websockets](https://en.wikipedia.org/wiki/WebSocket) than the `ws` field won't
exist. The server has to support one of the two. In this case the client can use
either.

When using websockets the user connects at the `ws` url and can send and receive
Game API JSON messages over the socket. In the case of using SSE, the user
should POST Game API JSON messages to the `post` url and receive the server
messages over the `sse` url.

### Ada's Turn

Once Ada's client connects she immediately gets a connect message from the server
with the full current game, players, and state information:

``` json
{
  "type": "connect",
  "game": {
    "id": "<game_id>",
    "created_at": "2023-06-13T09:41:00+00:00",
    "current_turn": 100,
    "current_action": "<action_100_id>",
    "current_player": "<player_ada_id>"
  },
  "players": [{
    "name": "Ada",
    "id": "<player_ada_id>",
    "index": 0,
    "faction": "cat",
    "color": "orange"
  },{
    "name": "Betty",
    "id": "<player_betty_id>",
    "index": 1,
    "faction": "eyrie",
    "color": "blue"
  },{
    "name": "Carol",
    "id": "<player_carol_id>",
    "index": 2,
    "faction": "woodland",
    "color": "green"
  }]
  "state": {
    "scores": {
      "<player_ada_id>": 13,
      "<player_betty_id>": 13,
      "<player_carol_id>": 13
    }
  },
}
```

Her client sets up their display and alerts her that it's her turn. Let's
pretend for demonstration purposes that her only available action this turn is
to do a single move. She makes the choice in her client and it will send the
message (either by POST'ing this JSON to the `post` url or by sending this
message over the websocket):

``` json
{
  "type": "action",
  "state": {...},
  "state_hash": "<sha_256_state_hash>",
  "action": {
    "type": "MOVE",
    "from": "clearing_1",
    "to": "clearing_2",
    "warriors": 3,
  },
  "log": [
    {
      "text": "Ada moves three warriors from clearing A to clearing B",
      "client": "Ada moves 3x|cat| from |clearing_1| to |clearing_2|",
    }
  ]
}
```

The `state_hash` field is a client specific hash of the state generated after
this action is applied. The `state` field is the full state for the server to
save.

> [!TIP]
> The Game API doesn't care what method the clients use to hash the state as
> long as the result is a SHA-256 compatible (256-bit as 32-character
> hexidecimal) result.

The `action` field requires a `type` but value of `type` and the rest of the
fields are fully game dependent. The `log` field requires an array of objects
with `text` fields and may optionally include other fields. The `text` field is
a pure string representation of what happened with this action. The optional
`client` field can be used by the game client to store a formatted string so
that the client can display log messages with colors or emojis, or whatever else
they want to support. Other fields are allowed here as well for that same
purpose.

Some actions (like we'll see soon) can have multiple log messages since the game
client might do other automated tasks.

### Betty's Turn

On Betty's computer, her client receives an action message telling her that Ada
made a move:

``` json
{
  "type": "action",
  "player": "<player_ada_id>",
  "state_hash": "<sha_256_state_hash>",
  "action": {...},
  "log": {...},
}
```

Her client gets this action and applies it to the game state it already has. It
then hashes the game state (using whatever specific hashing technique the client
has defined) and makes sure that the hash is identical.

The new state tells her client that Betty is up next. For sake of argument we're
goign to pretend that the only action available to her on her turn is to make an
attack. Her attack needs a random result from the server. The first thing she
does is request it by sending this message:

``` json
{
  "type": "random",
  "request": ["2x1d4"]
}
```

A client should always request ALL of the random data it needs at one time (for
a single action) in one request. The request field supports an array of
requests. One simple format of a request is the one shown above: `<α>x<β>d<γ>`
which means "Give me α different results of rolling a γ sided-die β times and
summing the results". This should be suitable for many different games. Other
request types (which might mean using full JSON objects instead of a simple
string) may be supported in the future.

The server then sends a random result to all clients:

``` json
{
  "type": "random",
  "id": "<random_result_1_id>",
  "request": ["2x1d4"],
  "result": [3, 2],
  "sig": "oiwejfwe",
}
```

This random result is saved on the server as an action like any others. It by
definition doesn't change the state but is saved and signed by the server so
that all clients can tell that the server generated it.

Betty's client then uses that information to create her action and send it:

``` json
{
  "type": "action",
  "state": {...},
  "state_hash": "<sha_256_state_hash>",
  "action": {
    "type": "ATTACK",
    "in": "clearing_3",
    "player": "<player_carol_id>"
  },
  "random_id": "<random_result_1_id>",
  "log": [
    {
      "text": "Betty attacks Carol in clearing C",
      "client": "Betty |eryie| attacks Carol |woodland| in |clearing_3|",
    }
  ]
}
```

It notes that it uses the random result that was just sent (clients are allowed
to use previous random results as well but there should only be one random
result between each action). Other clients can use this information to compute
the state deterministically.

### Carol's Turn

In our pretend game it's now Carol's turn and she decides to craft a card in her
hand. Let's pretend this finishes the game (in order to show an action that does
more to the state than just a single action):

``` json
{
  "type": "action",
  "state": {...},
  "state_hash": "<sha_256_state_hash>",
  "action": {
    "type": "CRAFT",
    "card": 45
  },
  "log": [
    {
      "text": "Carol crafts a Sword",
      "client": "Carol |woodland| crafts a |sword|",
    },
    {
      "text": "Carol earns 2 points",
      "client": "Carol |woodland| earns 2 |points|",
    },
    {
      "text": "Carol wins the game with 30 points!",
      "client": "Carol |woodland| wins the game with 30 |points|! |party|",
    }
  ]
}
```

## Main REST APIs

### Game Meta Data

> [!TIP]
> In the Game API all ids are
> [UUIDv4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random))
> values. In this document we will just use strings like `<game_id>` and
> `<player_ada_id>` instead of full UUIDs for readability.

The metadata for a game can be retrieved at `/games/<game_id>`. This will return
something like:

``` json
{
  "game": {
    "id": "<game_id>",
    "created_at": "2023-06-13T09:41:00+00:00",
    "current_turn": 100,
    "current_action": "<action_100_id>",
    "current_player": "<player_ada_id>"
  },
  "players": [{
    "name": "Ada",
    "id": "<player_ada_id>",
    "index": 0,
    "faction": "cat",
    "color": "orange"
  },{
    "name": "Betty",
    "id": "<player_betty_id>",
    "index": 1,
    "faction": "eyrie",
    "color": "blue"
  },{
    "name": "Carol",
    "id": "<player_carol_id>",
    "index": 2,
    "faction": "woodland",
    "color": "green"
  }]
}
```

In this example all root fields in the `game` object are required and managed by
the API itself. In the player objects only `id` is required. `index` is created
by the API (a random player ordering that can be used by the games if
needed). `name` is optional and most clients should set it. Fields like
`faction` and `color` are just created by the game client for their game. A lot
of objects in the Game API support any extra fields on objects to support game
features.

### State

You can get the full current game state with `/games/<game_id>/states/current`
or the initial game state with `/games/<game_id>/states/initial`.

``` json
{
  "game": {
    "id": "<game_id>",
    "created_at": "2023-06-13T09:41:00+00:00",
    "current_turn": 100,
    "current_action": "<action_100_id>",
    "current_player": "<player_ada_id>"
  },
  "state": {
    "scores": {
      "<player_ada_id>": 13,
      "<player_betty_id>": 13,
      "<player_carol_id>": 13
    }
  },
}
```

The `state` field is fully game dependent and supports all valid JSON. The other
fields are all generated by the API.

### Actions

You can get actions at `/games/<game_id>/actions`. This result is paginated and
defaults to recent events first.

You can get information about a single action at
`/games/<game_id>/actions/<action_id>`:

``` json
{
  "game": "<game_id>",
  "id": "<action_100_id>",
  "action": {
    "type": "MOVE",
    "from": "clearing_1",
    "to": "clearing_2",
    "warriors": 3,
  },
  "player": "<player_betty_id>",
  "parent": "<action_99_id>",
  "siblings": ["<action_100.1_id>", "<action_100.2_id>"],
  "children": ["<action_101_id>"],
  "log": "Human readable string of what happened",
  "created_at": "2024-06-13T09:41:00+00:00",
}
```

Just like the `state` field in the state APIs the `action` field here is fully
game dependent. All other fields in this example are generated by the API except
`log` which is a required part of creating an action (we'll see that later).
