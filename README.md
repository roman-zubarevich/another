Design decisions:
* Server should not send to a client any data that the corresponding player is not supposed to see. This will prevent cheating by means of a modified client.
* Server should explicitly send all data relevant to a client, even if the data can be figured out on the client side. Rationale: avoid game logic duplication with a cost of slightly higher network traffic.
* Every state change on server side happens in two steps: 1) pending notifications for all players are added to the state and persisted; 2) when all players confirm reception of the notifications, server persists the next state and clears all pending notifications. This allows to restore the state gracefully in case any connection gets broken or any participant goes down: it is guaranteed that every player sees every update (not just gets up-to-date state when reconnected).


```mermaid
stateDiagram-v2
    state discarded <<choice>>
    state showing_cards <<choice>>
    state turn_done <<choice>>
    [*] --> STARTING
    STARTING --> READY_FOR_TURN
    READY_FOR_TURN --> DECK_CARD_TAKEN: TakeCardFromDeck
    READY_FOR_TURN --> READY_FOR_TURN: ReplaceCardByDiscarded, StopRound
    READY_FOR_TURN --> showing_cards: ShowCards
    showing_cards --> REPLACING_MULTIPLE_CARDS: all cards are identical
    showing_cards --> READY_FOR_TURN: some cards are different
    DECK_CARD_TAKEN --> turn_done: ReplaceCard
    DECK_CARD_TAKEN --> discarded: Discard
    discarded --> turn_done: plain card
    discarded --> PEEKING_OWN_CARD: 7 or 8
    discarded --> PEEKING_ANOTHERS_CARD: 9 or 10
    discarded --> EXCHANGING_CARDS: 11 or 12
    REPLACING_MULTIPLE_CARDS --> READY_FOR_TURN: TakeDiscardedCard
    REPLACING_MULTIPLE_CARDS --> turn_done: TakeCardFromDeck
    PEEKING_OWN_CARD --> turn_done: PeekOwnCard
    PEEKING_ANOTHERS_CARD --> turn_done: PeekAnothersCard
    EXCHANGING_CARDS --> turn_done: ExchangeCards
    turn_done --> READY_FOR_TURN: not last turn
    turn_done --> FINISHED: last turn
    FINISHED --> [*]
```


### Phase A (round start)

Start a new round:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C1->>S: StartRound
    S->>S: Prepare initial state of the round
    S->>S: Randomly pick players' cards
    S->>S: Save pending notifications
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
        S->>S: Clear notification 2
    and
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1 and update state
    end
    Note over S,C3: Proceed to the first turn initiation (phase B)
```


### Phase B (turn initiation)

Initiate a turn (select random player for the first one, the 2nd player is selected here):
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    S->>S: Select a player (playerIndex) to make a move
    S->>S: Save pending notifications
    par
        S->>C1: NextTurn(playerIndex)
    and
        S->>C3: NextTurn(playerIndex)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: YourTurn
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
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
    S->>S: Save pending notifications
    par
        S->>C1: DeckCardTaken(playerIndex)
    and
        S->>C3: DeckCardTaken(playerIndex)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: TookDeckCard(value)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
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
    S->>S: Save pending notifications
    par
        S->>C1: CardReplaced(playerIndex, cardIndex, discardedValue)
    and
        S->>C3: CardReplaced(playerIndex, cardIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: ReplacedCardByDiscarded(cardIndex, discardedValue)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
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
    S->>S: Save pending notifications
    par
        S->>C1: IdenticalCardsShown(playerIndex, cardIndexes, value)
    and
        S->>C3: IdenticalCardsShown(playerIndex, cardIndexes, value)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: ShowedIdenticalCards(discardedValue)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
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
    S->>S: Save pending notifications
    par
        S->>C1: DifferentCardsShown(playerIndex, cardIndexes, values)
    and
        S->>C3: DifferentCardsShown(playerIndex, cardIndexes, values)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: ShowedDifferentCards(cardIndexes)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
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
    S->>S: Save pending notifications
    par
        S->>C1: StopInitiated(playerIndex)
    and
        S->>C3: StopInitiated(playerIndex)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: InitiatedStop
    C2-->>S: Ack
    S->>S: Clear notification 2 and initiate stop counter
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
    C2->>S: ReplaceCard(cardIndex)
    S->>S: Save pending notifications
    par
        S->>C1: CardReplaced(playerIndex, cardIndex, discardedValue)
    and
        S->>C3: CardReplaced(playerIndex, cardIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: ReplacedCardFromDeck(cardIndex, discardedValue)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
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
    S->>S: Save pending notifications
    par
        S->>C1: CardDiscarded(playerIndex, discardedValue)
    and
        S->>C3: CardDiscarded(playerIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: DiscardedCard(discardedValue)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
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
    S->>S: Save pending notifications
    par
        S->>C1: CardDiscarded(playerIndex, discardedValue)
    and
        S->>C3: CardDiscarded(playerIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: PeekingOwnCard(discardedValue)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
    C2->>S: PeekOwnCard(cardIndex)
    S->>S: Save pending notifications
    par
        S->>C1: OwnCardPeeked(playerIndex, cardIndex)
    and
        S->>C3: OwnCardPeeked(playerIndex, cardIndex)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: OwnCard(cardIndex, value)
    C2-->>S: Ack
    S->>S: Clear notification 2
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
    S->>S: Save pending notifications
    par
        S->>C1: CardDiscarded(playerIndex, discardedValue)
    and
        S->>C3: CardDiscarded(playerIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: PeekingAnothersCard(discardedValue)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
    C2->>S: PeekAnothersCard(anotherPlayerIndex, cardIndex)
    S->>S: Save pending notifications
    par
        S->>C1: AnothersCardPeeked(playerIndex, anotherPlayerIndex, cardIndex)
    and
        S->>C3: AnothersCardPeeked(playerIndex, anotherPlayerIndex, cardIndex)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: AnothersCard(anotherPlayerIndex, cardIndex, value)
    C2-->>S: Ack
    S->>S: Clear notification 2
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
    S->>S: Save pending notifications
    par
        S->>C1: CardDiscarded(playerIndex, discardedValue)
    and
        S->>C3: CardDiscarded(playerIndex, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: ExchangingCards(discardedValue)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
    C2->>S: ExchangeCards(cardIndex, anotherPlayerIndex, anotherPlayerCardIndex)
    S->>S: Save pending notifications
    par
        S->>C1: CardsExchanged(playerIndex, cardIndex, anotherPlayerIndex, anotherPlayerCardIndex)
    and
        S->>C3: CardsExchanged(playerIndex, cardIndex, anotherPlayerIndex, anotherPlayerCardIndex)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: ExchangedCards(cardIndex, anotherPlayerIndex, anotherPlayerCardIndex)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
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
    C2->>S: TakeCardFromDeck
    S->>S: Take a random card (value) from the deck
    S->>S: Save pending notifications
    par
        S->>C1: CardsReplacedFromDeck(playerIndex, cardIndexes, discardedValue)
    and
        S->>C3: CardsReplacedFromDeck(playerIndex, cardIndexes, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: ReplacedCardsFromDeck(cardIndexes, discardedValue)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
    Note over S,C3: Proceed to the next turn (phase B)
```

Replace multiple cards with a discarded card:
```mermaid
sequenceDiagram
    participant S as Server
    participant C1 as Client 1
    participant C2 as Client 2
    participant C3 as Client 3
    C2->>S: TakeDiscardedCard
    S->>S: Save pending notifications
    par
        S->>C1: CardsReplacedByDiscarded(playerIndex, cardIndexes, discardedValue)
    and
        S->>C3: CardsReplacedByDiscarded(playerIndex, cardIndexes, discardedValue)
    end
    Note over S: Wait for all acks
    par
        C3-->>S: Ack
        S->>S: Clear notification 3
    and
        C1-->>S: Ack
        S->>S: Clear notification 1
    end
    S->>C2: ReplacedCardsByDiscarded(cardIndexes, discardedValue)
    C2-->>S: Ack
    S->>S: Clear notification 2 and update state
    Note over S,C3: Proceed to the next turn (phase B)
```
