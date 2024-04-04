Design decisions:
* Server should not send to a client any data that the corresponding player is not supposed to see. This will prevent cheating by means of a modified client.
* Server should explicitly send all data relevant to a client, even if the data can be figured out on the client side. Rationale: avoid game logic duplication with a cost of slightly higher network traffic.
* Every state change on server side happens in two steps: 1) pending notifications for all players are added to the state and persisted; 2) when all players confirm reception of the notifications, server persists the next state and clears all pending notifications. This allows to restore the state gracefully in case any connection gets broken or any participant goes down: it is guaranteed that every player sees every update (not just gets up-to-date state when reconnected).


```mermaid
stateDiagram-v2
    state discarded <<choice>>
    state comparing_cards <<choice>>
    state replacing_cards <<choice>>
    state turn_done <<choice>>
    state round_done <<choice>>
    [*] --> ROUND_STARTING: StartGame
    ROUND_STARTING --> READY_FOR_TURN: Acks
    READY_FOR_TURN --> DECK_CARD_TAKEN: TakeCardFromDeck
    READY_FOR_TURN --> REPLACING_CARD_BY_DISCARDED: ReplaceCardByDiscarded
    READY_FOR_TURN --> ROUND_STOPPING: StopRound
    READY_FOR_TURN --> SHOWING_CARDS: ShowCards
    SHOWING_CARDS --> comparing_cards: Acks
    comparing_cards --> READY_FOR_TURN: some cards are different
    comparing_cards --> replacing_cards: all cards are identical
    replacing_cards --> REPLACING_MULTIPLE_CARDS_FROM_DECK: ReplaceCardsFromDeck
    replacing_cards --> REPLACING_MULTIPLE_CARDS_BY_DISCARDED: ReplaceCardsByDiscarded
    DECK_CARD_TAKEN --> REPLACING_CARD_FROM_DECK: ReplaceCardFromDeck
    REPLACING_CARD_FROM_DECK --> turn_done: Acks
    DECK_CARD_TAKEN --> DISCARDING: Discard
    DISCARDING --> discarded: Acks
    discarded --> turn_done: plain card
    discarded --> PEEKING_OWN_CARD: 7 or 8,<br/>PeekOwnCard
    discarded --> PEEKING_ANOTHERS_CARD: 9 or 10,<br/>PeekAnothersCard
    discarded --> EXCHANGING_CARDS: 11 or 12,<br/>ExchangeCards
    REPLACING_CARD_BY_DISCARDED --> READY_FOR_TURN: Acks
    REPLACING_MULTIPLE_CARDS_BY_DISCARDED --> READY_FOR_TURN: Acks
    ROUND_STOPPING --> READY_FOR_TURN: Acks
    REPLACING_MULTIPLE_CARDS_FROM_DECK --> turn_done: Acks
    PEEKING_OWN_CARD --> turn_done: Acks
    PEEKING_ANOTHERS_CARD --> turn_done: Acks
    EXCHANGING_CARDS --> turn_done: Acks
    turn_done --> READY_FOR_TURN: not last turn
    turn_done --> round_done: last turn
    round_done --> ROUND_FINISHED: nobody's score exceeds 66
    round_done --> GAME_FINISHED: someone's score exceeds 66
    ROUND_FINISHED --> ROUND_STARTING: StartNextRound
    GAME_FINISHED --> [*]
```


### Phase A (round start)

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
        S->>C1: GameState(deckSize, discardedValue, handSizes, playerIndex)
    and
        S->>C2: GameState(deckSize, discardedValue, handSizes, playerIndex)
    and
        S->>C3: GameState(deckSize, discardedValue, handSizes, playerIndex)
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
        S->>C1: FirstTwoCards(value1_1, value1_2)
    and
        S->>C2: FirstTwoCards(value2_1, value2_2)
    and
        S->>C3: FirstTwoCards(value3_1, value3_2)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the first turn of the first round (phase B)
```


### Phase B (turn initiation)

Initiate a turn (select random player for the first one, the 2nd player is selected here):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    S->>S: Select a player to make a move
    S->>S: Update state
    par
        S->>C1: NextTurn(playerIndex)
    and
        S->>C3: NextTurn(playerIndex)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: YourTurn
    C2-->>S: Ack
    Note over S,C3: Proceed to phase C
```


### Phase C (selection, identity testing, or stopping)

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
    S->>C2: TookDeckCard(value, deckSize)
    C2-->>S: Ack
    Note over S,C3: Proceed to phase D
```

Replace a card in hand by the topmost discarded card:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: ReplaceCardByDiscarded(cardIndex)
    S->>S: Update state
    par
        S->>C1: CardReplacedByDiscarded(playerIndex, cardIndex, discardedValue)
    and
        S->>C2: ReplacedCardByDiscarded(cardIndex, discardedValue)
    and
        S->>C3: CardReplacedByDiscarded(playerIndex, cardIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase B)
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
        S->>C1: IdenticalCardsShown(playerIndex, cardIndexes, value)
    and
        S->>C3: IdenticalCardsShown(playerIndex, cardIndexes, value)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: ShowedIdenticalCards(cardIndexes)
    C2-->>S: Ack
    Note over S,C3: Proceed to phase E
```

Claim to have two or more identical cards and fail:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: ShowCards(cardIndexes)
    S->>S: Compare card values (some cards are different)
    S->>S: Update state
    par
        S->>C1: DifferentCardsShown(playerIndex, cardIndexes, values)
    and
        S->>C2: ShowedDifferentCards(cardIndexes)
    and
        S->>C3: DifferentCardsShown(playerIndex, cardIndexes, values)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase B)
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
        S->>C1: StopInitiated(playerIndex)
    and
        S->>C2: InitiatedStop
    and
        S->>C3: StopInitiated(playerIndex)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase B)
```


### Phase D (action)

Replace a card in player's hand by the picked card:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: ReplaceCardFromDeck(cardIndex)
    S->>S: Update state
    par
        S->>C1: CardReplacedFromDeck(playerIndex, cardIndex, discardedValue)
    and
        S->>C2: ReplacedCardFromDeck(cardIndex, discardedValue)
    and
        S->>C3: CardReplacedFromDeck(playerIndex, cardIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase B)
```

Discard the picked card (simple):
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
        S->>C1: CardDiscarded(playerIndex, discardedValue)
    and
        S->>C2: DiscardedCard(discardedValue)
    and
        S->>C3: CardDiscarded(playerIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase B)
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
        S->>C1: CardDiscarded(playerIndex, discardedValue)
    and
        S->>C3: CardDiscarded(playerIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: PeekingOwnCard(discardedValue)
    C2-->>S: Ack
    C2->>S: PeekOwnCard(cardIndex)
    S->>S: Update state
    par
        S->>C1: OwnCardPeeked(playerIndex, cardIndex)
    and
        S->>C2: OwnCard(cardIndex, value)
    and
        S->>C3: OwnCardPeeked(playerIndex, cardIndex)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase B)
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
        S->>C1: CardDiscarded(playerIndex, discardedValue)
    and
        S->>C3: CardDiscarded(playerIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: PeekingAnothersCard(discardedValue)
    C2-->>S: Ack
    C2->>S: PeekAnothersCard(anotherPlayerIndex, cardIndex)
    S->>S: Update state
    par
        S->>C1: AnothersCardPeeked(playerIndex, anotherPlayerIndex, cardIndex)
    and
        S->>C2: AnothersCard(anotherPlayerIndex, cardIndex, value)
    and
        S->>C3: AnothersCardPeeked(playerIndex, anotherPlayerIndex, cardIndex)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase B)
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
        S->>C1: CardDiscarded(playerIndex, discardedValue)
    and
        S->>C3: CardDiscarded(playerIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: ExchangingCards(discardedValue)
    C2-->>S: Ack
    C2->>S: ExchangeCards(cardIndex, anotherPlayerIndex, anotherPlayerCardIndex)
    S->>S: Update state
    par
        S->>C1: CardsExchanged(playerIndex, cardIndex, anotherPlayerIndex, anotherPlayerCardIndex)
    and
        S->>C2: ExchangedCards(cardIndex, anotherPlayerIndex, anotherPlayerCardIndex)
    and
        S->>C3: CardsExchanged(playerIndex, cardIndex, anotherPlayerIndex, anotherPlayerCardIndex)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase B)
```


### Phase E (multiple cards replacement)

Replace multiple cards with a card from deck:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: ReplaceCardsFromDeck
    S->>S: Take a random card (value) from the deck
    S->>S: Update state
    par
        S->>C1: CardsReplacedFromDeck(playerIndex, cardIndexes, discardedValue)
    and
        S->>C2: ReplacedCardsFromDeck(cardIndexes, discardedValue)
    and
        S->>C3: CardsReplacedFromDeck(playerIndex, cardIndexes, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase B)
```

Replace multiple cards with a discarded card:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: ReplaceCardsByDiscarded
    S->>S: Update state
    par
        S->>C1: CardsReplacedByDiscarded(playerIndex, cardIndexes, discardedValue)
    and
        S->>C2: ReplacedCardsByDiscarded(cardIndexes, discardedValue)
    and
        S->>C3: CardsReplacedByDiscarded(playerIndex, cardIndexes, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    Note over S,C3: Proceed to the next turn (phase B)
```

### Final phase

Round is finished, but the game will continue:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    S->>S: Calculate round and total score
    S->>S: Update state
    par
        S->>C1: RoundFinished(turnCount, scores, totalScores, nextPlayerIndex)
    and
        S->>C3: RoundFinished(turnCount, scores, totalScores, nextPlayerIndex)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
    S->>C2: RoundFinished(turnCount, scores, totalScores, nextPlayerIndex)
    C2-->>S: Ack
    C2->S: StartNextRound
    Note over S,C3: Proceed to the first turn of the new round (phase B)
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
        S->>C1: GameFinished(roundCount, scores, totalScores)
    and
        S->>C2: GameFinished(roundCount, scores, totalScores)
    and
        S->>C3: GameFinished(roundCount, scores, totalScores)
    end
    par
        C2-->>S: Ack
    and
        C3-->>S: Ack
    and
        C1-->>S: Ack
    end
```
