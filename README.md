# Pico Pong Online

Example of using pico-socket for an online pong game

## How to run

```
npm ci
npm start
```

## What's going on?

This uses [pico-socket](https://github.com/JRJurman/pico-socket) to add online multiplayer to a simple pong game.

This is done by having the game state live in GPIO addresses, which both Pico-8 and pico-socket can read. The following addresses are used in the game:

| Variable   | GPIO Address | Pico-Socket Index |
| ---------- | ------------ | ----------------- |
| PLAYER_ID  | 0x5f80       | 0                 |
| ROOM_ID    | 0x5f81       | 1                 |
| SCORE_1    | 0x5f82       | 2                 |
| SCORE_2    | 0x5f83       | 3                 |
| PLAYER_1_Y | 0x5f84       | 4                 |
| PLAYER_2_Y | 0x5f85       | 5                 |
| BALL_X_POS | 0x5f86       | 6                 |
| BALL_X_SPD | 0x5f87       | 7                 |
| BALL_Y_POS | 0x5f88       | 8                 |
| BALL_Y_SPD | 0x5f89       | 9                 |

- The `PLAYER_ID` is the selected player in a single game instance (either `1` or `2`).
- `ROOM_ID` is a unique value that separates players into isolated sessions
- `SCORE_1` and `SCORE_2` are the values for the two players
- `PLAYER_1_Y` and `PLAYER_2_Y` are each player's paddle position in the game
- `BALL_X_POS`, `BALL_X_SPD`, `BALL_Y_POS`, and `BALL_Y_SPD` are values for describing the moving ball

The `ROOM_ID` and `PLAYER_ID` are unique in that they change the behavior of the server. The `ROOM_ID`
when set will establish a connection to the server so that it can communicate with other clients in the
same `ROOM_ID`. `PLAYER_ID` determines what data we will send to the other clients.

Functionally, we want each player to be responsible for their own data (Player 1 should never read
their position from another player, and similarly, should not be responsible for sending Player 2's position).
We also want a single player to be responsible for general game state (so there is no conflict on who should
resolve the final game state).

So, in pico-socket we have indicies associated with each player that determine what data they should send.
We have `PLAYER_2_Y` associated with the Player 2 - it's the only data we expect from Player 2.
Conversely, for Player 1 we have `PLAYER_1_Y` AND we have all the game data (like the score and ball state).
This means that Player 1 is responsible for telling Player 2 where the ball is, and what the score is.
If player 2 would calculate that in their own game, it gets ignored and overwritten.

So, to implement this configuration, we have the following:
* game logic that exclusively reads and writes to GPIO addresses
* server configuration that controls which data we should be passing back and forth

### Pico-8 Logic

On the pico-8 side, we made a lookup table so we can read and write this data (see the netcode file for complete details):
```lua
-- lookup table with the gpio addresses
lookup = {}
lookup["player_id"]   = 0x5f80
lookup["room_id"]     = 0x5f81
lookup["score_1"]     = 0x5f82
lookup["score_2"]     = 0x5f83
lookup["player_1_y"]  = 0x5f84
lookup["player_2_y"]  = 0x5f85
lookup["ball_x_pos"]  = 0x5f86
lookup["ball_x_spd"]  = 0x5f87
lookup["ball_y_pos"]  = 0x5f88
lookup["ball_y_spd"]  = 0x5f89

-- nset
function nset(key, value)
	poke(lookup[key], value)
end

-- nget
function nget(key)
	return peek(lookup[key])
end
```

Then, throughout the Pico-8 game logic we exclusively use `nget` and `nset`.
You can just use the `poke` and `peek` methods directly, but this lookup table is useful
if you need to change or update the addresses.

### Server Logic

On the server side, we just need to call `createPicoSocketServer` from `pico-socket`,
passing in the different GPIO addresses that we should be looking at.

```js
createPicoSocketServer({
  // where the js and html file are
  assetFilesPath: ".",

  // where the game html file is
  htmlGameFilePath: "./pong.html",

  clientConfig: {
    // index to read to determine the room
    // that the player joined
    roomIdIndex: 1,

    // index to determine the player id
    playerIdIndex: 0,

    // indicies that contain player specific data
    playerDataIndicies: [
      // there is no zeroth player,
      [],
      // first player position, and game data
      [4, 2, 3, 6, 7, 8, 9],
      // second player position
      [5],
    ],
  },
});
```

Now, while both clients are running, they will each write and send their respective values
from GPIO (the first player will send their position and game data, and the second player will
send just their position). This data is sent to the other connected players, and we will write
those values to the Pico-8 game in those GPIO addresses.
