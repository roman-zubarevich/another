### Phase A (round start)

Start a new round:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: StartRound
    S->>S: Prepare initial state of the round, randomly pick players' cards
    par
        S->>C1: FirstTwoCards(value1_1, value1_2)
    and
        S->>C2: FirstTwoCards(value2_1, value2_2)
    and
        S->>C3: FirstTwoCards(value3_1, value3_2)
    end
    Note over S: Wait for ack from all clients
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>S: Randomly select a player (playerIndex) to start moving
    Note over S: Notify other players first
    par
        S->>C1: FirstTurn(playerIndex)
    and
        S->>C3: FirstTurn(playerIndex)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: FirstTurn(playerIndex)
    Note over S,C3: Proceed to the first turn (phase B)
```

### Phase B (selection, identity testing, or stopping)

Pick card from heap:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: TakeCardFromHeap
    par
        S->>C2: HeapCardTaken(playerIndex)
    and
        S->>C3: HeapCardTaken(playerIndex)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3->>S: Ack
    and
        C2->>S: Ack
    end
    S->>S: Take a random card (value) from the heap
    S->>C1: Card(value)
    Note over S,C3: Proceed to phase C
```

Pick topmost discarded card (for exchange):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: TakeDiscardedCard
    par
        S->>C2: DiscardedCardTaken(playerIndex, value)
    and
        S->>C3: DiscardedCardTaken(playerIndex, value)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3->>S: Ack
    and
        C2->>S: Ack
    end
    S->>C1: Card(value)
    Note over S,C3: Proceed to the phase C
```

Claim to have two or more identical cards and succeed:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: ShowCards(handCardIndexes)
    S->>S: Compare card values (all appear equal)
    par
        S->>C2: CardsShown(playerIndex, handCardIndexes, values)
    and
        S->>C3: CardsShown(playerIndex, handCardIndexes, values)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C2-->>S: Ack
    end
    S->>C1: CardsIdentical(discardedValue)
    Note over S,C3: Proceed to phase D
```

Claim to have two or more identical cards and fail:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: ShowCards(handCardIndexes)
    S->>S: Compare card values (some cards are different)
    par
        S->>C1: CardsDifferent
    and
        S->>C3: CardsShown(playerIndex, handCardIndexes, values)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: CardsShown(playerIndex, handCardIndexes, values)
    Note over S,C3: All clients recognize Client 2 as the next player
    Note over S,C3: Proceed to the next turn (phase B)
```

### Phase C (action)

Exchange the picked card with a card in player's hand:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: ExchangeCard(handCardIndex)
    par
        S->>C1: CardExchanged(playerIndex, handCardIndex, discardedValue)
    and
        S->>C3: CardExchanged(playerIndex, handCardIndex, discardedValue)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>S: Update 1st player's hand and discarded card (here??)
    S->>C2: CardExchanged(playerIndex, handCardIndex, discardedValue)
    Note over S,C3: All clients recognize Client 2 as the next player
    Note over S,C3: Proceed to the next turn (phase B)
```

Discard the picked card (simple):

Discard the picked card (7 or 8):

Discard the picked card (9 or 10):

Discard the picked card (11 or 12):

### Phase D (multiple cards replacement)

Replace multiple cards with a card from heap:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: TakeCardFromHeap
    S->>S: Take a random card (value) from the heap
    par
        S->>C1: Card(value)
    and
        S->>C3: HeapCardTaken(playerIndex)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3->>S: Ack
    and
        C1->>S: Ack
    end
    S->>S: Update 1st player's hand and discarded card (here??)
    S->>C2: HeapCardTaken(playerIndex)
    Note over S,C3: All clients recognize Client 2 as the next player
    Note over S,C3: Proceed to the next turn (phase B)
```

Replace multiple cards with a discarded card:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: TakeDiscardedCard
    par
        S->>C1: Card(value)
    and
        S->>C3: DiscardedCardTaken(playerIndex, value)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3->>S: Ack
    and
        C1->>S: Ack
    end
    S->>S: Update 1st player's hand and discarded card (here??)
    S->>C2: DiscardedCardTaken(playerIndex, value)
    Note over S,C3: All clients recognize Client 2 as the next player
    Note over S,C3: Proceed to the next turn (phase B)
```
