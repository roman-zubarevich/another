# another

```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: StartGame
    par
        S->>C1: FirstTwoCards for the 1st player
    and
        S->>C2: FirstTwoCards for the 2nd player
    and
        S->>C3: FirstTwoCards for the 3rd player
    end
    Note over S: Wait for responses from all clients
    par
        C2-->>S: SeenCards
    and
        C3-->>S: SeenCards
    and
        C1-->>S: SeenCards
    end
    S->>S: Randomly select a client to start moving
    S->>C2: YourTurn
```

```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    S->>C2: YourTurn
    C2->>S: TakeCardFromHeap
    S->>C2: Card
    par
        S->>C1,C3: HeapCardTaken
    and
        S->>C3: HeapCardTaken
    end
```