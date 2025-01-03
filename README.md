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

> [!TIP]
> In the Game API all ids are
> [UUIDv4](https://en.wikipedia.org/wiki/Universally_unique_identifier#Version_4_(random))
> values. In this document we will just use strings like `<game_id>` and
> `<player_ada_id>` instead of full UUIDs for readability.

## Action and State Ordering

Each action has an `order` number assigned by the server. This number starts at
1 and increments on every user action. Action 0 is the "setup" action which
creates the initial game state. This action represents a user creating the game
initially.

Every action is either `current` or not (`current` is a boolean field on
actions). Non-current actions mean it was a branch of the action tree that was
undone and replayed. Non-current branches are labeled with a `variant` number
that starts at 1 and increments every time an undo is taken. States can be
uniquely referred to by the action number that created them and an optional
`variant` to refer to an undone action branch. The `current` branch of actions
can also be referred to a `variant` 0.

States can be referred to by the action that created them. State `0` is always
the initial game state. State `α` is the state after the action `α`.

An actions siblings are all actions with the same `order` number and different
`variants`. The end of each variant tree is the `MAX(order)` given a variant
number.

### Undo Algorithm

1. Server has a game with `N` current actions and `M` variants.
2. User requests undo to `order: α`
3. Server changes all actions that are `order: > α` and `variant: 0` (or
   `current: true`) to the `variant: M + 1`
4. Server sets current action to `order: α`. Server now has `M + 1` variants and
   `α` actions that are current.

### Example Undo

Lets look at what the trees look like at the time. When the first undo is going
to happen the tree looks like this (each action is in the form `α:β` which means
an action with `order: α` and `variant: β`):

`0:0 -> 1:0 -> 2:0 -> 3:0 -> 4:0 -> 5:0`

A player then sends a message to the server that they want to undo to action
`4`.

> [!NOTE]
>
> Undos are always against the current (`variant: 0`) tree.

The server then edits all actions after `4:0` to `variant: 1` resulting in:

`0:0 -> 1:0 -> 2:0 -> 3:0 -> 4:0 -> 5:1`

The current move is now set to `4` and they await the next action from that
state. Eventually the server receives this and now out full tree looks like:

``` text
0:0
 |
1:0
 |
2:0
 |
3:0
 |
4:0 --
 |    \
5:0  5:1
```

At this point another player resets to action `2` so the server edits all
actions `2:0` to `variant: 2` resulting in:

``` text
0:0
 |
1:0
 |
2:0
 |
3:2
 |
4:2 --
 |    \
5:2  5:1
```

Once the next three actions are played we end up with this full tree:

``` text
0:0
 |
1:0
 |
2:0 --
 |    \
3:0  3:2
 |    |
4:0  4:2 --
 |    |    \
5:0  5:2  5:1
```

## Example Session

Imagine we have a game of
[Root](https://ledergames.com/products/root-a-game-of-woodland-might-and-right)
with 3 players. We'll name them Ada [^1], Betty [^2], and Carol [^3]. They are
in the middle of a game. This section is meant to explain how the game api works
with examples of what messages might look like.

> [!WARNING]
> This example was written prior to the full API spec being designed. Messages
> and their schemas are not final.

[^1]: [Ada Lovelace](https://en.wikipedia.org/wiki/Ada_Lovelace) was the
    designer of the first algorithm.
[^2]: [Betty Holberton](https://en.wikipedia.org/wiki/Betty_Holberton) invented
    breakpoints in compute debugging and was one of the six original programmers
    of ENIAC.
[^3]: [Carol Shaw](https://en.wikipedia.org/wiki/Carol_Shaw) was one of the
    first female video game designers.

### Game Creation

In this example we'll assume the players are playing on a web client hosted at
`https://client` and using a Game API server hosted at `https://server`.

Ada goes to their client's homepage and creates a new game. They get asked
questions (game dependent) about player count and other game options.

At that point their client will send a POST request to `https://server/games` to
request a new game. The body of this request needs to have some basic server
settings about players:

``` json
{
  "game": {
    "name": "Ada's Fun Root Game",
  },
  "players": [{
    "name": "Ada",
    "faction": "cat",
    "color": "orange"
  },{
    "name": "Betty",
    "faction": "eyrie",
    "color": "blue"
  },{
    "name": "Carol",
    "faction": "woodland",
    "color": "green"
  }]
}
```

The server responds with some data that will help the game client setup initial
state:

``` json
{
  "game": {
    "id": "<game_id>",
    "name": "Ada's Fun Root Game",
    "state": "setup",
    "seed": "<seed>",
    "created_by": "<player_ada_id>",
    "created_at": "2023-06-13T09:41:00+00:00",
    "actions": 0,
    "variants": 0,
    "urls": {
      "events": "https://server/games/<game_id>/events"
    }
  },
  "players": [{
    "id": "<player_ada_id>",
    "name": "Ada",
    "seat": 1,
    "faction": "cat",
    "color": "orange",
    "urls": {
      "actions": "https://server/games/<game_id>/players/<player_ada_id>",
      "events": "https://server/games/<game_id>/players/<player_ada_id>/events"
    }
  },{
    "id": "<player_betty_id>",
    "name": "Betty",
    "seat": 2,
    "faction": "eyrie",
    "color": "blue",
    "urls": {
      "actions": "https://server/games/<game_id>/players/<player_betty_id>",
      "events": "https://server/games/<game_id>/players/<player_betty_id>/events"
    }
  },{
    "id": "<player_carol_id>",
    "name": "Carol",
    "seat": 3,
    "faction": "woodland",
    "color": "green",
    "urls": {
      "actions": "https://server/games/<game_id>/players/<player_carol_id>",
      "events": "https://server/games/<game_id>/players/<player_carol_id>/events"
    }
  }]
}
```

We can see that the game and each player received an ID. The game had some
metadata created including a `seed` value. When creating a game you can specify
a `seed` value as part of the initial game request if you want.

Each player also received an `seat` number. This number is random and the
clients don't have to use it, but it could be useful to help setup some games.

The current state of the game is `setup` which means the server needs an initial
state from the creating player.

### Connecting

Ada's client can now connect to her personal `events` url which will immediately
send her this same json with a `"type": "setup"` field added and url fields
removed. The next step is that Ada's client needs to send a `setup` event with
an initial game state. This also could require getting some random data from the
server.

### Setup

Ada's client first requests some random data:

``` json
{
  "type": "random",
  "request": ["[1-24]", "[1-5)", "8x2d6"]
}
```

and any connected clients to this game will receive the response:

``` json
{
  "type": "random",
  "id": "<random_result_0_id>",
  "action": 0,
  "current": true,
  "variant": 0,
  "request": ["[1-24]", "[1-5)", "8x2d6"]
  "result": [14, 4, [5, 8, 7, 2, 12, 12, 9, 7]],
  "sig": "<random_result_0_sig>",
}
```

Ada's client then uses this result to make the initial game state and sets it:

``` json
{
  "type": "setup",
  "state": {...},
  "state_hash": "<sha_256_state_hash>",
  "log": [{
    "text": "Ada starts as the Marquise de Cat",
    "client": "Ada starts as |cat|",
  },{
    "text": "Betty starts as the Eyrie Dynasty",
    "client": "Betty starts as |eyrie|",
  },{
    "text": "Carol starts as the Woodland Alliance",
    "client": "Carol starts as |woodland|",
  }]
}
```

Messages of `"type": "setup"` can be thought of as creating action `0`. It's
only valid when a game is in the `setup` state and can't be undone. It is the
oldest undo target. Now that initial state is setup the game can begin.

All connected clients get a started message changing the game state to `active`:

``` json
{
  "type": "started",
  "game": {
    "id": "<game_id>",
    "name": "Ada's Fun Root Game",
    "state": "active",
    "seed": "<seed>",
    "created_by": "<player_ada_id>",
    "created_at": "2023-06-13T09:41:00+00:00",
    "actions": 0,
    "variants": 0,
  },
  "state": {...},
  "state_hash": "<sha_256_state_hash>",
  "log": [{
    "text": "Ada starts as the Marquise de Cat",
    "client": "Ada starts as |cat|",
  },{
    "text": "Betty starts as the Eyrie Dynasty",
    "client": "Betty starts as |eyrie|",
  },{
    "text": "Carol starts as the Woodland Alliance",
    "client": "Carol starts as |woodland|",
  }]
  "players": [{
    "id": "<player_ada_id>",
    "name": "Ada",
    "seat": 1,
    "faction": "cat",
    "color": "orange",
  },{
    "id": "<player_betty_id>",
    "name": "Betty",
    "seat": 2,
    "faction": "eyrie",
    "color": "blue",
  },{
    "id": "<player_carol_id>",
    "name": "Carol",
    "seat": 3,
    "faction": "woodland",
    "color": "green",
  }]
}
```

### Ada's Turn

Ada's client alerts her that it's her turn. Let's pretend for demonstration
purposes that her only available action this turn is to do a single move. She
makes the choice in her client and it will send the message by posting it as the
body to the `actions` url for her player:

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
  "log": [{
    "text": "Ada moves three warriors from clearing A to clearing B",
    "client": "Ada moves 3x|cat| from |clearing_1| to |clearing_2|",
  }]
}
```

The `state_hash` field is a client specific hash of the state generated after
this action is applied. The `state` field is the full state for the server to
save.

> [!TIP]
> The Game API doesn't care what method the clients use to hash the state as
> long as the result is a SHA-256 compatible (256-bit as 32-character
> hexidecimal) result.

The `action` field requires a `type` field. The value of `type` and the rest of
the fields are fully game dependent.

The `log` field requires an array of objects with `text` fields and may
optionally include other fields. The `text` field is a pure string
representation of what happened with this action. The optional `client` field
can be used by the game client to store a formatted string so that the client
can display log messages with colors or emojis, or whatever else they want to
support. Other fields are allowed here as well for that same purpose.

Some actions (like we'll see soon) can have multiple log messages since the game
client might do other automated tasks.

### Betty's Turn

Let's pretend that Betty only joins the game at this point by being sent a URL
from Ada. Ada's client generated a url for Betty that was something like:
`https://client/games/<game_id>/players/<player_betty_id>`.

> [!NOTE]
> We don't actually care what this URL looks like since it's on the
> client. Somehow the client needs to learn what the `<game_id>` and
> `<player_betty_id>` are. It doesn't matter how.

When Betty loads this URL her client will first request some information from
the Game API server by sending a GET request to
`https://server/games/<game_id>/players/<players_betty_id>` which will return:

``` json
{
  "game": {
    "id": "<game_id>",
    "name": "Ada's Fun Root Game",
    "state": "active",
    "seed": "<seed>",
    "created_by": "<player_ada_id>",
    "created_at": "2023-06-13T09:41:00+00:00",
    "actions": 1,
    "variants": 0,
    "urls": {
      "events": "https://server/games/<game_id>/events"
    }
  },
  "players": [{
    "id": "<player_ada_id>",
    "name": "Ada",
    "seat": 1,
    "faction": "cat",
    "color": "orange",
  },{
    "id": "<player_betty_id>",
    "name": "Betty",
    "seat": 2,
    "faction": "eyrie",
    "color": "blue",
    "urls": {
      "actions": "https://server/games/<game_id>/players/<player_betty_id>",
      "events": "https://server/games/<game_id>/players/<player_betty_id>/events"
    }
  },{
    "id": "<player_carol_id>",
    "name": "Carol",
    "seat": 3,
    "faction": "woodland",
    "color": "green",
  }]
}
```

She will then connect to the `events` url and get the following message:

``` json
{
  "type": "connected",
  "game": {
    "id": "<game_id>",
    "name": "Ada's Fun Root Game",
    "state": "active",
    "seed": "<seed>",
    "created_by": "<player_ada_id>",
    "created_at": "2023-06-13T09:41:00+00:00",
    "actions": 1,
    "variants": 0,
  },
  "state": {...},
  "state_hash": "<sha_256_state_hash>",
  "log": [{
    "action": { "order": 0, "variant": 0, "current": true },
    "text": "Ada starts as the Marquise de Cat",
    "client": "Ada starts as |cat|",
  },{
    "action": { "order": 0, "variant": 0, "current": true },
    "text": "Betty starts as the Eyrie Dynasty",
    "client": "Betty starts as |eyrie|",
  },{
    "action": { "order": 0, "variant": 0, "current": true },
    "text": "Carol starts as the Woodland Alliance",
    "client": "Carol starts as |woodland|",
  },{
    "action": { "order": 1, "variant": 0, "current": true },
    "text": "Ada moves three warriors from clearing A to clearing B",
    "client": "Ada moves 3x|cat| from |clearing_1| to |clearing_2|",
  }]
  "players": [{
    "id": "<player_ada_id>",
    "name": "Ada",
    "seat": 1,
    "faction": "cat",
    "color": "orange",
  },{
    "id": "<player_betty_id>",
    "name": "Betty",
    "seat": 2,
    "faction": "eyrie",
    "color": "blue",
  },{
    "id": "<player_carol_id>",
    "name": "Carol",
    "seat": 3,
    "faction": "woodland",
    "color": "green",
  }]
}
```

The state tells her client that Betty is up next (how is game dependent). For
sake of argument we're goign to pretend that the only action available to her on
her turn is to make an attack. Her attack needs a random result from the
server. The first thing she does is request it by sending this message:

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
  "id": "<random_result_2_id>",
  "action": {
    "order": 2,
    "variant": 0,
    "current": true
  },
  "request": ["2x1d4"],
  "result": [3, 2],
  "sig": "<random_result_2_sig>",
}
```

This random result is saved on the server related to action `2`. There can only
be one random request per action. It by definition doesn't change the state but
is saved and signed by the server so that all clients can tell that the server
generated it.

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
  "log": [{
    "text": "Betty attacks Carol in clearing C",
    "client": "Betty |eryie| attacks Carol |woodland| in |clearing_3|",
  }]
}
```

All clients now get a new action message:

``` json
{
  "type": "action",
  "player": "<player_betty_id>",
  "order": 2,
  "current" true,
  "variant": 0,
  "state_hash": "<sha_256_state_hash>",
  "action": {
    "type": "ATTACK",
    "in": "clearing_3",
    "player": "<player_carol_id>"
  },
  "log": [{
    "text": "Betty attacks Carol in clearing C",
    "client": "Betty |eryie| attacks Carol |woodland| in |clearing_3|",
  }]
}
```

Since all clients will already have the random information associated with this
action they should all be able to verify their state matches the hash.

### Carol's Turn

In our pretend game it's now Carol's turn (let's assume she's already connected)
and she decides to craft a card in her hand. Let's pretend this finishes the
game (in order to show an action that does more to the state than just a single
action):

``` json
{
  "type": "action",
  "state": {...},
  "state_hash": "<sha_256_state_hash>",
  "game": {
    "state": "finished",
    "winners": ["<player_carol_id>"]
  },
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

You can see that she requests that the game is marked `finished` and the
`winners` field is set to an array of winning player ids.

## Main REST APIs

### Game Meta Data

The metadata for a game can be retrieved at `/games/<game_id>`. This will return
something like:

``` json
{
  "game": {
    "id": "<game_id>",
    "name": "Ada's Fun Root Game",
    "state": "setup",
    "seed": "<seed>",
    "created_by": "<player_ada_id>",
    "created_at": "2023-06-13T09:41:00+00:00",
    "actions": 98,
    "variants": 2,
    "urls": {
      "events": "https://server/games/<game_id>/events"
    }
  },
  "players": [{
    "id": "<player_ada_id>",
    "name": "Ada",
    "seat": 1,
    "faction": "cat",
    "color": "orange",
    "urls": {
      "actions": "https://server/games/<game_id>/players/<player_ada_id>",
      "events": "https://server/games/<game_id>/players/<player_ada_id>/events"
    }
  },{
    "id": "<player_betty_id>",
    "name": "Betty",
    "seat": 2,
    "faction": "eyrie",
    "color": "blue",
    "urls": {
      "actions": "https://server/games/<game_id>/players/<player_betty_id>",
      "events": "https://server/games/<game_id>/players/<player_betty_id>/events"
    }
  },{
    "id": "<player_carol_id>",
    "name": "Carol",
    "seat": 3,
    "faction": "woodland",
    "color": "green",
    "urls": {
      "actions": "https://server/games/<game_id>/players/<player_carol_id>",
      "events": "https://server/games/<game_id>/players/<player_carol_id>/events"
    }
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

### Log

You can get the full game log with `/games/<game_id>/log`:

``` json
{
  "game": {
    "id": "<game_id>",
    "name": "Ada's Fun Root Game",
    "state": "setup",
    "seed": "<seed>",
    "created_by": "<player_ada_id>",
    "created_at": "2023-06-13T09:41:00+00:00",
    "actions": 2,
    "variants": 0
  },
  "log": [{
    "action": { "order": 0, "variant": 0, "current": true },
    "text": "Ada starts as the Marquise de Cat",
    "client": "Ada starts as |cat|",
  },{
    "action": { "order": 0, "variant": 0, "current": true },
    "text": "Betty starts as the Eyrie Dynasty",
    "client": "Betty starts as |eyrie|",
  },{
    "action": { "order": 0, "variant": 0, "current": true },
    "text": "Carol starts as the Woodland Alliance",
    "client": "Carol starts as |woodland|",
  },{
    "action": { "order": 1, "variant": 0, "current": true },
    "text": "Ada moves three warriors from clearing A to clearing B",
    "client": "Ada moves 3x|cat| from |clearing_1| to |clearing_2|",
  },{
    "action": { "order": 2, "variant": 0, "current": true },
    "text": "Carol crafts a Sword",
    "client": "Carol |woodland| crafts a |sword|",
  },{
    "action": { "order": 2, "variant": 0, "current": true },
    "text": "Carol earns 2 points",
    "client": "Carol |woodland| earns 2 |points|",
  },{
    "action": { "order": 2, "variant": 0, "current": true },
    "text": "Carol wins the game with 30 points!",
    "client": "Carol |woodland| wins the game with 30 |points|! |party|",
  }]
```

### State

You can get the full current game state with `/games/<game_id>/states` or the
game state after any action `α` with `/games/<game_id>/states/α`. Remember that
`0` can be used to get the initial state of a game. The `variant` query
parameter can be used to fetch a different variant than current. `variant=0`
will always be the same as not including the `variant` paramter.

``` json
{
  "game": {
    "id": "<game_id>",
    "name": "Ada's Fun Root Game",
    "state": "active",
    "seed": "<seed>",
    "created_by": "<player_ada_id>",
    "created_at": "2023-06-13T09:41:00+00:00",
    "actions": 1,
    "variants": 0,
  },
  "state": {...},
  "state_hash": "<sha_256_state_hash>",
}
```

The `state` field is fully game dependent and supports all valid JSON. The other
fields are all generated by the API.

### Actions

You can get actions at `/games/<game_id>/actions`. This result is paginated and
defaults to recent events first. All variants are included by default.

``` json
"TBD"
```

You can get information about a single action with `order: α` and `variant: β`
at `/games/<game_id>/actions/α?variant=β`:

``` json
{
  "game": "<game_id>",
  "player": "<player_betty_id>",
  "order": 5,
  "variant": 0,
  "current" true,
  "action": {
    "type": "MOVE",
    "from": "clearing_1",
    "to": "clearing_2",
    "warriors": 3,
  },
  "parent": { "order": 4, "variant": 0 },
  "siblings": [{ "order": 5, "variant": 2 }, { "order": 5, "variant": 1 }],
  "log": [{
    "text": "Ada made a great play as the cats",
    "client": "Ada made a great play as |cat|"
  }],
  "created_at": "2024-06-13T09:41:00+00:00",
}
```

You can also view the tree of actions without any details at
`/games/<game_id>/tree` which results in something like:

``` json
{
  "actions": 5,
  "variants": 2,
  "tree": [
    [{ "order": 0, "variant": 0, "current": true }],
    [{ "order": 1, "variant": 0, "current": true }],
    [{ "order": 2, "variant": 0, "current": true }],
    [{ "order": 3, "variant": 0, "current": true }, { "order": 3, "variant": 2 }],
    [{ "order": 4, "variant": 0, "current": true }, { "order": 4, "variant": 2 }],
    [{ "order": 5, "variant": 0, "current": true }, { "order": 5, "variant": 2 }, { "order": 5, "variant": 1 }],
  ]
}
```
