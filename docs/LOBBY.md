# Non-gameplay infrastructure

## Game states

```mermaid
stateDiagram-v2
    state rejoined <<choice>>
    state left <<choice>>
    [*] --> OPEN: CreateGame
    OPEN --> OPEN: JoinGame
    OPEN --> STARTED: StartGame
    STARTED --> SUSPENDED: SuspendGame
    SUSPENDED --> rejoined: RejoinGame
    rejoined --> STARTED: all players in game
    rejoined --> SUSPENDED: not all players in game
    OPEN --> left: LeaveGame
    left --> OPEN: game creator did not leave
    left --> [*]: game creator left
```

## Client-server interaction protocol

Assuming that 3 clients are connected to server.

### Player operations

Register a new player:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: RegisterPlayer
    S->>S: Generate unique player secret
    S->>S: Persist player
    S->>C1: PlayerInfo(name, secret)
    S-->>C2: PlayerStatus(id, name, online)
    S-->>C3: PlayerStatus(id, name, online)
```

Rename a player:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: RenamePlayer(name)
    S->>S: Persist player with the new name
    S->>C1: PlayerInfo(name, secret)
    S-->>C2: PlayerStatus(id, name, online)
    S-->>C3: PlayerStatus(id, name, online)
```

### Listing entities

List online players:
```mermaid
sequenceDiagram
    Server-->>Client: OnlinePlayers(ids, names)
```

List open games:
```mermaid
sequenceDiagram
    Server-->>Client: OpenGames(ids, openTimes, playerNameLists)
```

List suspended games:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: ListSuspendedGames(playerSecret)
    S->>C1: SuspendedGames(ids, startTimes, suspendTimes, playerNameLists, playerOnlineStatuses)
    S-->>C2: PlayerStatus(id, name, online)
    S-->>C3: PlayerStatus(id, name, online)
```

### Game lobby operations

Assuming player 1 creates the game, player 2 joins it, and player 3 is only connected to server.

Create a game:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: CreateGame(playerSecret)
    S->>C1: JoinedGame(id)
    S-->>C2: OpenGames(id, openTime, creatorName)
    S-->>C3: OpenGames(id, openTime, creatorName)
```

Join a game:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: JoinGame(gameId, playerSecret)
    S->>C2: JoinedGame(id)
    S-->>C1: NewPlayer(gameId, playerName)
    S-->>C3: NewPlayer(gameId, playerName)
```

Abandon a game:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: LeaveGame
    S->>C1: GameDeleted(id)
    S-->>C2: GameDeleted(id)
    S-->>C3: GameDeleted(id)
```

Leave an open game:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: LeaveGame
    S-->>C1: PlayerRemoved(gameId, playerIndex)
    S->>C2: PlayerRemoved(gameId, playerIndex)
    S-->>C3: PlayerRemoved(gameId, playerIndex)
```

Start a game:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: StartGame
    S->>C1: GameStarted(id)
    S-->>C2: GameStarted(id)
    S-->>C3: GameDeleted(id)
```

### Game operations

Assuming players 1 and 2 are in the game, and player 3 is only connected to server.

Suspend a started game:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    C1->>S: SuspendGame
    S->>C1: GameSuspended(id, startTime, suspendTime, playerNames, playerOnlineStatuses, suspendingPlayerIndex)
    S-->>C2: GameSuspended(id, startTime, suspendTime, playerNames, playerOnlineStatuses, suspendingPlayerIndex)
```

Leave a suspended game:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: SuspendGame
    S->>C1: PlayerLeft(gameId, playerIndex)
    S-->>C2: PlayerLeft(gameId, playerIndex)
    S-->>C3: PlayerLeft(gameId, playerIndex)
```

Rejoin a game (all other players are already in the game):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    C1->>S: RejoinGame(gameId, playerSecret)
    S->>C1: PlayerRejoined(gameId, playerIndex)
    S-->>C2: PlayerRejoined(gameId, playerIndex)
```

Rejoin a game (some other players did not rejoin yet):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: RejoinGame(gameId, playerSecret)
    S->>C1: PlayerRejoined(gameId, playerIndex)
    S-->>C2: PlayerRejoined(gameId, playerIndex)
    S-->>C3: PlayerRejoined(gameId, playerIndex)
```
