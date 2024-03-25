# another

```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    Note over S,C3: ...
    C1->>S: StartGame
    par
        S->>C1: FirstTwoCards(n1, n2) for the 1st player
    and
        S->>C2: FirstTwoCards(n1, n2) for the 2nd player
    and
        S->>C3: FirstTwoCards(n1, n2) for the 3rd player
    end
    Note over S: Wait for ack from all clients
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>S: Randomly select a player to start moving
    par
        S->>C1: NextTurn(playerIndex)
    and
        S->>C2: NextTurn(playerIndex)
    and
        S->>C3: NextTurn(playerIndex)
    end
    Note over S: Wait for ack from all clients
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the first turn
```

```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    Note over S,C3: ...
    C2->>S: TakeCardFromHeap
    par
        S->>C1: HeapCardTaken(playerIndex)
    and
        S->>C3: HeapCardTaken(playerIndex)
    end
    Note over S: Wait for ack from all clients except active player
    par
        C3->>S: Ack
    and
        C1->>S: Ack
    end
    S->>C2: Card(n)
    C2->>S: ExchangeCard(handCardIndex)
    par
        S->>C1: CardExchanged(playerIndex, handCardIndex)
    and
        S->>C2: CardExchanged(playerIndex, handCardIndex)
    and
        S->>C3: CardExchanged(playerIndex, handCardIndex)
        Note over C3: Based on playerIndex, the client identifies itself as the next player
    end
    Note over S: Wait for ack from all clients
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn
```
