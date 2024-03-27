Design decisions:
* Server should not send to a client any data that the corresponding player is not supposed to see. This will prevent cheating by means of a modified client.
* Server should explicitly send all data relevant to a client, even if the data can be figured out on the client side. Rationale: avoid game logic duplication with a cost of slightly higher network traffic.


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
        S->>C1: TurnStart(playerIndex)
    and
        S->>C3: TurnStart(playerIndex)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: TurnStart(playerIndex)
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

Replace a card in hand by the topmost discarded card:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: ReplaceCardByDiscarded(handCardIndex)
    par
        S->>C1: CardReplaced(playerIndex, handCardIndex, discardedValue)
    and
        S->>C3: CardReplaced(playerIndex, handCardIndex, discardedValue)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
        S->>S: Update 1st player's hand, discarded card, and whose turn it is
    and
        C1-->>S: Ack
    end
    S->>C2: CardReplaced(playerIndex, handCardIndex, discardedValue)
    Note over S,C3: All clients recognize Client 2 as the next player
    Note over S,C3: Proceed to the next turn (phase B)
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

Stop the round:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: StopRound
    par
        S->>C1: Stopping(playerIndex)
    and
        S->>C3: Stopping(playerIndex)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3->>S: Ack
    and
        C1->>S: Ack
    end
    S->>S: Initiate stop counter
    S->>C2: Stopping(playerIndex)
    Note over S,C3: All clients recognize Client 2 as the next player
    Note over S,C3: Proceed to the next turn (phase B)
```


### Phase C (action)

Replace a card in player's hand by the picked card:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: ReplaceCard(handCardIndex)
    par
        S->>C1: CardReplaced(playerIndex, handCardIndex, discardedValue)
    and
        S->>C3: CardReplaced(playerIndex, handCardIndex, discardedValue)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>S: Update 1st player's hand and discarded card (here??)
    S->>C2: CardReplaced(playerIndex, handCardIndex, discardedValue)
    Note over S,C3: All clients recognize Client 2 as the next player
    Note over S,C3: Proceed to the next turn (phase B)
```

Discard the picked card (simple):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: Discard
    S->>S: Check discarded card (appears to be plain)
    par
        S->>C1: Discarded(playerIndex, discardedValue)
    and
        S->>C3: Discarded(playerIndex, discardedValue)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>S: Update discarded card (here??)
    S->>C2: Discarded(playerIndex, discardedValue)
    Note over S,C3: All clients recognize Client 2 as the next player
    Note over S,C3: Proceed to the next turn (phase B)
```

Discard the picked card (7 or 8):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: Discard
    S->>S: Check discarded card (appears to be 7 or 8)
    par
        S->>C2: Discarded(playerIndex, discardedValue)
    and
        S->>C3: Discarded(playerIndex, discardedValue)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C2-->>S: Ack
    end
    S->>S: Update discarded card (here??)
    S->>C1: Discarded(playerIndex, discardedValue)
    C1->>S: PeekOwnCard(handCardIndex)
    par
        S->>C1: Card(value)
    and
        S->>C3: OwnCardPeeked(playerIndex, handCardIndex)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: OwnCardPeeked(playerIndex, handCardIndex)
    Note over S,C3: All clients recognize Client 2 as the next player
    Note over S,C3: Proceed to the next turn (phase B)
```

Discard the picked card (9 or 10):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: Discard
    S->>S: Check discarded card (appears to be 9 or 10)
    par
        S->>C2: Discarded(playerIndex, discardedValue)
    and
        S->>C3: Discarded(playerIndex, discardedValue)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C2-->>S: Ack
    end
    S->>S: Update discarded card (here??)
    S->>C1: Discarded(playerIndex, discardedValue)
    C1->>S: PeekCard(anotherPlayerIndex, handCardIndex)
    par
        S->>C1: Card(value)
    and
        S->>C3: CardPeeked(playerIndex, anotherPlayerIndex, handCardIndex)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: CardPeeked(playerIndex, anotherPlayerIndex, handCardIndex)
    Note over S,C3: All clients recognize Client 2 as the next player
    Note over S,C3: Proceed to the next turn (phase B)
```

Discard the picked card (11 or 12):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: Discard
    S->>S: Check discarded card (appears to be 11 or 12)
    par
        S->>C2: Discarded(playerIndex, discardedValue)
    and
        S->>C3: Discarded(playerIndex, discardedValue)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C2-->>S: Ack
    end
    S->>S: Update discarded card (here??)
    S->>C1: Discarded(playerIndex, discardedValue)
    C1->>S: ExchangeCards(handCardIndex, anotherPlayerIndex, anotherPlayerHandCardIndex)
    par
        S->>C1: CardsExchanged(playerIndex, handCardIndex, anotherPlayerIndex, anotherPlayerHandCardIndex)
    and
        S->>C3: CardsExchanged(playerIndex, handCardIndex, anotherPlayerIndex, anotherPlayerHandCardIndex)
    end
    Note over S: Wait for ack from all notified clients
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>S: Update players' hands after exchange (here??)
    S->>C2: CardsExchanged(playerIndex, handCardIndex, anotherPlayerIndex, anotherPlayerHandCardIndex)
    Note over S,C3: All clients recognize Client 2 as the next player
    Note over S,C3: Proceed to the next turn (phase B)
```


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
