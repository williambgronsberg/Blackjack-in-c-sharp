# Blackjack Online (C#) — Detailed Project Plan

## 1. Project Overview

**Goal:** Build a C# multiplayer Blackjack application, playable online via socket connection, where players join a table using a PIN code. Primary purpose is learning C# — networking, OOP, async programming, UI, and testing.

**Style:** Client-server architecture. One host creates a room and receives a PIN; other players join with that PIN; the server acts as the dealer and enforces all rules.

---

## 2. Requirements

### 2.1 Functional Requirements
- FR1: Host can create a room and receive a unique PIN code
- FR2: Player can join a room by entering PIN + name
- FR3: Server deals two cards to each player and two to the dealer (one hidden)
- FR4: Players can hit, stand (and optionally double down / split as stretch goals)
- FR5: Server enforces turn order — one player acts at a time
- FR6: Bust detection (hand value > 21) ends that player's turn automatically
- FR7: Blackjack detection (Ace + 10-value card) pays out correctly
- FR8: Dealer plays by fixed rules after all players finish (e.g., hit until 17+)
- FR9: Server calculates each player's result (win/lose/push) and updates chips
- FR10: Client UI updates live as cards are dealt and turns progress
- FR11: Support multiple rounds without needing to reconnect
- FR12: Disconnect/reconnect handled without crashing the table

### 2.2 Non-Functional Requirements
- NFR1: Server is the single source of truth — never trust client-calculated hand values
- NFR2: Actions sync across clients within ~200ms on LAN
- NFR3: Clean separation into Server / Client / Shared projects
- NFR4: Shared message contracts used identically by both sides
- NFR5: Malformed or out-of-turn messages are rejected, not crash-inducing

---

## 3. Tech Stack

| Layer | Choice | Notes |
|---|---|---|
| Language | C# (.NET 8) | |
| Networking | `TcpListener` / `TcpClient`, async/await | JSON-line or length-prefixed messages |
| Serialization | `System.Text.Json` | Shared DTOs in `Shared` project |
| UI | **Avalonia UI** (recommended) | XAML + CSS-like styling, cross-platform, closest to web dev experience |
| Alt. UI | Blazor Hybrid | If you'd rather build the UI in HTML/CSS/Razor |
| Version control | Git + GitHub | Feature branches, PRs |

---

## 4. Solution Structure

```
Blackjack.sln
 ├─ Blackjack.Server/
 │   ├─ Networking/
 │   ├─ Rooms/
 │   ├─ Game/            (Deck, Dealer, RoundManager)
 │   └─ Program.cs
 ├─ Blackjack.Client/     (Avalonia app)
 │   ├─ Views/
 │   ├─ ViewModels/
 │   ├─ Networking/
 │   └─ App.axaml
 ├─ Blackjack.Shared/
 │   ├─ Messages/
 │   ├─ Models/           (Card, Player, Hand, GameState)
 │   └─ Enums/            (Suit, Rank, ActionType, RoundPhase)
 └─ Blackjack.Tests/
```

---

## 5. Message Protocol

**Client → Server**
- `create_room { hostName }`
- `join_room { pin, playerName }`
- `start_round {}`
- `player_action { actionType }` — actionType: `hit | stand | double | split`

**Server → Client**
- `room_created { pin }`
- `player_joined { players[] }`
- `round_started { yourHand[], dealerUpCard }`
- `game_update { players[], currentPlayerId, dealerHand }`
- `your_turn { validActions[] }`
- `bust { playerId }`
- `round_result { results[] }` — win/lose/push + payout per player, dealer's full hand revealed
- `player_disconnected { playerId }`
- `error { code, message }`

---

## 6. Core Systems to Build

### 6.1 Networking & Rooms (server)
- Accept connections, async read/write loop per client
- Generate unique PINs, create/track room objects
- Join validation (correct PIN, room not full, game not already in progress)

### 6.2 Game Engine (server)
- Deck: build, shuffle, deal, track remaining cards (reshuffle when low)
- Hand value calculation, correctly handling Aces as 1 or 11
- Turn manager: enforce one active player at a time, auto-advance on bust/stand
- Dealer logic: reveal hidden card, hit until 17+ (decide if dealer hits on soft 17)
- Blackjack payout logic (typically 3:2), push/tie handling
- Round reset for next hand

### 6.3 Client UI
- Screens: landing, join-room, lobby, game table, round-result
- Table layout: dealer hand (one hidden card), each player's hand + chip count, turn indicator
- Action buttons: hit / stand (+ double/split if implemented), disabled off-turn
- Live rendering from `game_update` / `your_turn` messages
- Simple deal animation and win/lose feedback

### 6.4 Shared Contracts
- All DTOs and enums in `Shared`, referenced by both Server and Client — never duplicated

---

## 7. Milestones & Timeline (suggested pace, adjust to your deadline)

| Milestone | Content | Target |
|---|---|---|
| M1 — Foundation | Architecture set, protocol drafted, socket connects, empty UI screens | Week 1 |
| M2 — Rooms | PIN generation, join-by-PIN, lobby shows players | Week 2 |
| M3 — Core Gameplay | Deal cards, turn order, hit/stand sync, bust detection | Week 3 |
| M4 — Dealer & Results | Dealer auto-play, payout logic, round results, chip updates | Week 4 |
| M5 — Multi-round & Polish | Repeat rounds without reconnect, UI polish, disconnect handling | Week 5 |
| M6 — Testing & Docs | Full test pass, README, packaging for submission | End |

---

## 8. Testing Strategy
- **Unit tests**: hand-value calculation (Ace handling, soft/hard totals), deck shuffle/deal, dealer decision logic
- **Integration tests**: server + multiple client instances, play several full rounds
- **Manual QA**: disconnect mid-round, invalid action attempts, joining mid-round, going all-in on chips

## 9. Risks & Mitigations
| Risk | Mitigation |
|---|---|
| Client and server hand-value logic diverge | Only the server calculates results; client only displays what it's told |
| Reshuffle timing bugs (deck runs out mid-round) | Reshuffle only between rounds, never mid-hand |
| UI blocks while waiting on network | Use async/await throughout; never block the UI thread on socket calls |
| Scope creep (split/double before core loop works) | Get plain hit/stand/dealer/payout fully working first, then add extras |

## 10. Stretch Goals
- Double down and split
- Insurance bets
- Multiple decks / shoe with penetration-based reshuffle
- Persistent chip balance across sessions
- Simple chat or emotes
- Sound effects and card-flip animation

---

## 11. Workflow Notes
- Keep `Shared` as the only place message/model definitions live
- Test client-server integration after every milestone, not just at the end
- If working with a partner, split as: one owns server/game engine/dealer logic, the other owns client UI/table rendering — meeting in the middle on the message protocol
