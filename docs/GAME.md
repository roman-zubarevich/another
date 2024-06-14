# Gameplay implementation

## Gameplay states

```mermaid
stateDiagram-v2
    state checking_card <<choice>>
    state comparing_cards <<choice>>
    state turn_done <<choice>>
    state round_done <<choice>>
    [*] --> ROUND_STARTING: StartGame
    ROUND_STARTING --> READY_FOR_TURN: Acks
    READY_FOR_TURN --> DECK_CARD_TAKEN: TakeCardFromDeck
    READY_FOR_TURN --> DISCARDED_CARD_TAKEN: TakeDiscardedCard
    READY_FOR_TURN --> ROUND_STOPPING: StopRound

    DECK_CARD_TAKEN --> REPLACING_CARDS: PickOwnCard
    DISCARDED_CARD_TAKEN --> REPLACING_CARDS: PickOwnCard
    REPLACING_CARDS --> turn_done: Acks

    DECK_CARD_TAKEN --> checking_card: Discard
    checking_card --> DISCARDING_PLAIN: plain card
    checking_card --> DISCARDING_7_8: 7 or 8
    checking_card --> DISCARDING_9_10: 9 or 10
    checking_card --> DISCARDING_11_12: 11 or 12
    DISCARDING_PLAIN --> turn_done: Acks
    DISCARDING_7_8 --> SEEING_OWN_CARD: PickOwnCard
    SEEING_OWN_CARD --> turn_done: Acks
    DISCARDING_9_10 --> SEEING_ANOTHERS_CARD: PickAnothersCard
    SEEING_ANOTHERS_CARD --> turn_done: Acks
    DISCARDING_11_12 --> PICKING_ANOTHERS_CARD_FOR_EXCHANGE: PickOwnCard
    PICKING_ANOTHERS_CARD_FOR_EXCHANGE --> EXCHANGING_CARD: PickAnothersCard
    EXCHANGING_CARD --> turn_done: Acks

    DECK_CARD_TAKEN --> SHOWING_CARDS: ShowCards
    DISCARDED_CARD_TAKEN --> SHOWING_CARDS: ShowCards
    SHOWING_CARDS --> comparing_cards: Acks
    comparing_cards --> REPLACING_CARDS: all cards are identical
    comparing_cards --> READY_FOR_TURN: some cards are different

    ROUND_STOPPING --> READY_FOR_TURN: Acks
    turn_done --> READY_FOR_TURN: not last turn
    turn_done --> ROUND_FINISHED: last turn
    ROUND_FINISHED --> round_done: Acks
    round_done --> ROUND_STARTING: nobody's score exceeds 66,<br>StartNextRound
    round_done --> [*]: someone's score exceeds 66
```

## Client-server interaction protocol

Assuming that there are 3 players in the game, so 3 clients interact with server.

### Phase 1 (round start)

Start a new round:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: StartNextRound or StartGame
    S->>S: Prepare initial state of the round
    S->>S: Randomly pick players' cards
    S->>S: Update state
    par
        S->>C1: BoardInitialized(activePlayerIndex, deckSize, discardedValue, handSizes, stopperIndex, round, turn)
    and
        S->>C2: BoardInitialized(activePlayerIndex, deckSize, discardedValue, handSizes, stopperIndex, round, turn)
    and
        S->>C3: BoardInitialized(activePlayerIndex, deckSize, discardedValue, handSizes, stopperIndex, round, turn)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    par
        S->>C1: CardsRevealed(lookingPlayerIndex, targetPlayerIndex, firstTwoCardIndexes, firstTwoCardValues1)
    and
        S->>C2: CardsRevealed(lookingPlayerIndex, targetPlayerIndex, firstTwoCardIndexes, firstTwoCardValues2)
    and
        S->>C3: CardsRevealed(lookingPlayerIndex, targetPlayerIndex, firstTwoCardIndexes, firstTwoCardValues3)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the first turn of the first round (phase 2)
```

### Phase 2 (turn initiation)

Initiate a turn (select random player for the first turn, the 2nd player is selected here):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    S->>S: Select a player to make a move
    S->>S: Update state
    par
        S->>C1: TurnStarted(activePlayerIndex, turn)
    and
        S->>C3: TurnStarted(activePlayerIndex, turn)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: TurnStarted(activePlayerIndex, turn, actions)
    C2-->>S: Ack
    Note over S,C3: Proceed to phase 3
```

### Phase 3 (initial card selection or stopping round)

Pick card from deck:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: TakeCardFromDeck
    S->>S: Pick a random card (value) from the deck
    S->>S: Update state
    S->>C2: BoardUpdated(deckSize, deckCard, actions)
    C2-->>S: Ack
    Note over S,C3: Proceed to phase 4a or 4b
```

Pick the discarded card:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: TakeDiscardedCard
    S->>S: Update state
    S->>C2: BoardUpdated(actions)
    C2-->>S: Ack
    Note over S,C3: Proceed to phase 4a
```

Stop the round:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: StopRound
    S->>S: Update state
    par
        S->>C1: StopRequested(stopperIndex)
    and
        S->>C2: StopRequested(stopperIndex)
    and
        S->>C3: StopRequested(stopperIndex)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase 2)
```

### Phase 4a (replacing or identity testing)

Replace a card in player's hand by the picked card:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: PickOwnCard(cardIndex)
    S->>S: Update state
    par
        S->>C1: CardsReplaced(playerIndex, cardIndexes, fromDeck, discardedValue, deckSize)
    and
        S->>C2: CardsReplaced(playerIndex, cardIndexes, fromDeck, discardedValue, deckSize)
    and
        S->>C3: CardsReplaced(playerIndex, cardIndexes, fromDeck, discardedValue, deckSize)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase 2)
```

Claim to have two or more identical cards and succeed:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: ShowCards(cardIndexes)
    S->>S: Compare card values (all appear equal)
    S->>S: Update state
    par
        S->>C1: CardsRevealed(lookingPlayerIndexes, targetPlayerIndex, cardIndexes, cardValues)
    and
        S->>C2: CardsRevealed(lookingPlayerIndexes, targetPlayerIndex, cardIndexes)
    and
        S->>C3: CardsRevealed(lookingPlayerIndexes, targetPlayerIndex, cardIndexes, cardValues)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>S: Update state
    par
        S->>C1: CardsReplaced(playerIndex, cardIndexes, fromDeck, discardedValue, deckSize)
    and
        S->>C2: CardsReplaced(playerIndex, cardIndexes, fromDeck, discardedValue, deckSize)
    and
        S->>C3: CardsReplaced(playerIndex, cardIndexes, fromDeck, discardedValue, deckSize)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase 2)
```

Claim to have two or more identical cards and fail:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: ShowCards(cardIndexes)
    S->>S: Compare card values (all appear equal)
    S->>S: Update state
    par
        S->>C1: CardsRevealed(lookingPlayerIndexes, targetPlayerIndex, cardIndexes, cardValues)
    and
        S->>C2: CardsRevealed(lookingPlayerIndexes, targetPlayerIndex, cardIndexes)
    and
        S->>C3: CardsRevealed(lookingPlayerIndexes, targetPlayerIndex, cardIndexes, cardValues)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase 2)
```

### Phase 4b (discarding)

Discard the picked card (plain):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: Discard
    S->>S: Check discarded card (appears to be plain)
    S->>S: Update state
    par
        S->>C1: BoardUpdated(deckSize, discardedValue)
    and
        S->>C2: BoardUpdated(deckSize, discardedValue)
    and
        S->>C3: BoardUpdated(deckSize, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase 2)
```

Discard the picked card (7 or 8):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: Discard
    S->>S: Check discarded card (appears to be 7 or 8)
    S->>S: Update state
    par
        S->>C1: BoardUpdated(deckSize, discardedValue)
    and
        S->>C3: BoardUpdated(deckSize, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: BoardUpdated(deckSize, discardedValue, actions)
    C2-->>S: Ack
    C2->>S: PeekOwnCard(cardIndex)
    S->>S: Update state
    par
        S->>C1: CardsRevealed(lookingPlayerIndex, targetPlayerIndex, cardIndex)
    and
        S->>C2: CardsRevealed(lookingPlayerIndex, targetPlayerIndex, cardIndex, cardValue)
    and
        S->>C3: CardsRevealed(lookingPlayerIndex, targetPlayerIndex, cardIndex)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase 2)
```

Discard the picked card (9 or 10):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: Discard
    S->>S: Check discarded card (appears to be 9 or 10)
    S->>S: Update state
    par
        S->>C1: BoardUpdated(deckSize, discardedValue)
    and
        S->>C3: BoardUpdated(deckSize, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: BoardUpdated(deckSize, discardedValue, actions)
    C2-->>S: Ack
    C2->>S: PeekAnothersCard(playerIndex, cardIndex)
    S->>S: Update state
    par
        S->>C1: CardsRevealed(lookingPlayerIndex, targetPlayerIndex, cardIndex)
    and
        S->>C2: CardsRevealed(lookingPlayerIndex, targetPlayerIndex, cardIndex, cardValue)
    and
        S->>C3: CardsRevealed(lookingPlayerIndex, targetPlayerIndex, cardIndex)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase 2)
```

Discard the picked card (11 or 12):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: Discard
    S->>S: Check discarded card (appears to be 11 or 12)
    S->>S: Update state
    par
        S->>C1: BoardUpdated(deckSize, discardedValue)
    and
        S->>C3: BoardUpdated(deckSize, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: BoardUpdated(deckSize, discardedValue, actions)
    C2-->>S: Ack
    C2->>S: PeekOwnCard(cardIndex)
    S->>S: Update state
    S-->>C2: BoardUpdated(actions)
    C2-->>S: Ack
    C2->>S: PeekAnothersCard(playerIndex, cardIndex)
    S->>S: Update state
    par
        S->>C1: CardExchanged(playerIndex, cardIndex, anotherPlayerIndex, anotherPlayerCardIndex)
    and
        S->>C2: CardExchanged(playerIndex, cardIndex, anotherPlayerIndex, anotherPlayerCardIndex)
    and
        S->>C3: CardExchanged(playerIndex, cardIndex, anotherPlayerIndex, anotherPlayerCardIndex)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase 2)
```

### Phase 5 (finish round)

Round is finished, but the game will continue (assuming it is player 1's turn):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    S->>S: Calculate round and total score
    S->>S: Update state
    par
        S->>C1: RoundFinished(cards, scores, totalScores, isGameFinished)
    and
        S->>C2: RoundFinished(cards, scores, totalScores, isGameFinished)
    and
        S->>C3: RoundFinished(cards, scores, totalScores, isGameFinished)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C1: BoardUpdated(actions)
    C1-->>S: Ack
    C1->S: StartNextRound
    Note over S,C3: Proceed to the first turn of the new round (phase 2)
```

Game is finished:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    S->>S: Calculate round and total score
    S->>S: Update state
    par
        S->>C1: RoundFinished(cards, scores, totalScores, isGameFinished)
    and
        S->>C2: RoundFinished(cards, scores, totalScores, isGameFinished)
    and
        S->>C3: RoundFinished(cards, scores, totalScores, isGameFinished)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>S: Delete game
```
