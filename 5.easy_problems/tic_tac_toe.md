# 9. Tic Tac Toe

```python
from __future__ import annotations

from dataclasses import dataclass
from enum import Enum
from typing import Dict, List, Optional


# =========================
# Enums
# =========================

class GameCondition(str, Enum):
    IN_PROGRESS = "IN_PROGRESS"
    ENDED = "ENDED"


# =========================
# Player
# =========================

@dataclass(frozen=True)
class Player:
    name: str
    symbol: str


# =========================
# Move
# =========================

@dataclass(frozen=True)
class Move:
    col_index: int
    row_index: int
    player: Player


# =========================
# Board
# =========================

class Board:
    def __init__(self):
        self.grid: List[List[Optional[Player]]] = [[None for _ in range(3)] for _ in range(3)]

    def update_board(self, col_index: int, row_index: int, player: Player) -> None:
        if self.grid[col_index][row_index] is None:
            self.grid[col_index][row_index] = player

    def get_winner(self) -> Optional[Player]:
        # Check rows
        for i in range(3):
            first = self.grid[i][0]
            if first is not None and all(cell == first for cell in self.grid[i]):
                return first

        # Check columns
        for j in range(3):
            first = self.grid[0][j]
            if first is not None and all(self.grid[i][j] == first for i in range(3)):
                return first

        # Check main diagonal
        top_left = self.grid[0][0]
        if top_left is not None and all(self.grid[i][i] == top_left for i in range(3)):
            return top_left

        # Check anti-diagonal
        top_right = self.grid[0][2]
        if top_right is not None and all(self.grid[i][2 - i] == top_right for i in range(3)):
            return top_right

        return None

    def is_full(self) -> bool:
        return all(cell is not None for row in self.grid for cell in row)

    def reset(self) -> None:
        for i in range(3):
            for j in range(3):
                self.grid[i][j] = None

    def get_player_at(self, col_index: int, row_index: int) -> Optional[Player]:
        return self.grid[col_index][row_index]

    def __repr__(self) -> str:
        rows = []
        for row in self.grid:
            rows.append(" | ".join(cell.symbol if cell else " " for cell in row))
        return "\n---------\n".join(rows)


# =========================
# Score Tracker
# =========================

class ScoreTracker:
    def __init__(self):
        self.player_ratings: Dict[Player, int] = {}

    def report_game_result(
        self,
        player1: Player,
        player2: Player,
        winning_player: Optional[Player],
    ) -> None:
        if winning_player is not None:
            winner = winning_player
            loser = player2 if player1 == winner else player1

            self.player_ratings.setdefault(winner, 0)
            self.player_ratings[winner] += 1

            self.player_ratings.setdefault(loser, 0)
            self.player_ratings[loser] -= 1

    def get_top_players(self) -> Dict[Player, int]:
        sorted_items = sorted(
            self.player_ratings.items(),
            key=lambda entry: entry[1],
            reverse=True,
        )
        return {player: rating for player, rating in sorted_items}

    def get_rank(self, player: Player) -> int:
        sorted_players = [
            p for p, _ in sorted(
                self.player_ratings.items(),
                key=lambda entry: entry[1],
                reverse=True,
            )
        ]
        return sorted_players.index(player) + 1 if player in sorted_players else -1


# =========================
# Game
# =========================

class Game:
    def __init__(self, player_x: Player, player_y: Player):
        self.board = Board()
        self.score_tracker = ScoreTracker()
        self.players: List[Player] = []
        self.current_player_index = 0
        self.moves: List[Move] = []
        self.start_new_game(player_x, player_y)

    def start_new_game(self, player_x: Player, player_y: Player) -> None:
        self.board.reset()
        self.players = [player_x, player_y]
        self.current_player_index = 0
        self.moves = []

    def make_move(self, col_index: int, row_index: int, player: Player) -> None:
        if self.get_game_status() == GameCondition.ENDED:
            raise ValueError("game ended")

        if self.players[self.current_player_index] != player:
            raise ValueError("not the current player")

        if self.board.get_player_at(col_index, row_index) is not None:
            raise ValueError("board position is taken")

        self.board.update_board(col_index, row_index, player)
        new_move = Move(col_index, row_index, player)
        self.moves.append(new_move)

        self.current_player_index = (self.current_player_index + 1) % len(self.players)

        if self.get_game_status() == GameCondition.ENDED:
            self.score_tracker.report_game_result(
                self.players[0],
                self.players[1],
                self.board.get_winner(),
            )

    def get_game_status(self) -> GameCondition:
        winner = self.board.get_winner()
        if winner is not None:
            return GameCondition.ENDED
        return GameCondition.ENDED if self.board.is_full() else GameCondition.IN_PROGRESS

    def get_current_player(self) -> Player:
        return self.players[self.current_player_index]

    def get_score_tracker(self) -> ScoreTracker:
        return self.score_tracker


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    player_x = Player("Alice", "X")
    player_y = Player("Bob", "O")

    game = Game(player_x, player_y)

    print("Current player:", game.get_current_player().name)

    game.make_move(0, 0, player_x)
    game.make_move(1, 0, player_y)
    game.make_move(0, 1, player_x)
    game.make_move(1, 1, player_y)
    game.make_move(0, 2, player_x)

    print("\nBoard:")
    print(game.board)

    print("\nWinner:", game.board.get_winner())
    print("Game status:", game.get_game_status())

    tracker = game.get_score_tracker()
    print("\nRatings:", tracker.get_top_players())
    print("Rank of Alice:", tracker.get_rank(player_x))
    print("Rank of Bob:", tracker.get_rank(player_y))
```

DEEP DIVE

```python
from __future__ import annotations

from collections import deque
from dataclasses import dataclass
from enum import Enum
from typing import Deque, Dict, List, Optional


# =========================
# Enums
# =========================

class GameCondition(str, Enum):
    IN_PROGRESS = "IN_PROGRESS"
    ENDED = "ENDED"


# =========================
# Player
# =========================

@dataclass(frozen=True)
class Player:
    name: str
    symbol: str


# =========================
# Move
# Memento like object
# =========================

@dataclass(frozen=True)
class Move:
    col_index: int
    row_index: int
    player: Player


# =========================
# Move History
# Caretaker
# =========================

class MoveHistory:
    def __init__(self):
        self.history: Deque[Move] = deque()

    def record_move(self, move: Move) -> None:
        self.history.appendleft(move)

    def undo_move(self) -> Move:
        if not self.history:
            raise ValueError("no moves to undo")
        return self.history.popleft()

    def clear_history(self) -> None:
        self.history.clear()

    def is_empty(self) -> bool:
        return len(self.history) == 0


# =========================
# Board
# =========================

class Board:
    def __init__(self):
        self.grid: List[List[Optional[Player]]] = [[None for _ in range(3)] for _ in range(3)]

    def update_board(self, col_index: int, row_index: int, player: Optional[Player]) -> None:
        self.grid[col_index][row_index] = player

    def get_winner(self) -> Optional[Player]:
        # Check rows
        for i in range(3):
            first = self.grid[i][0]
            if first is not None and all(cell == first for cell in self.grid[i]):
                return first

        # Check columns
        for j in range(3):
            first = self.grid[0][j]
            if first is not None and all(self.grid[i][j] == first for i in range(3)):
                return first

        # Check main diagonal
        top_left = self.grid[0][0]
        if top_left is not None and all(self.grid[i][i] == top_left for i in range(3)):
            return top_left

        # Check anti-diagonal
        top_right = self.grid[0][2]
        if top_right is not None and all(self.grid[i][2 - i] == top_right for i in range(3)):
            return top_right

        return None

    def is_full(self) -> bool:
        return all(cell is not None for row in self.grid for cell in row)

    def reset(self) -> None:
        for i in range(3):
            for j in range(3):
                self.grid[i][j] = None

    def get_player_at(self, col_index: int, row_index: int) -> Optional[Player]:
        return self.grid[col_index][row_index]

    def __repr__(self) -> str:
        rows = []
        for row in self.grid:
            rows.append(" | ".join(cell.symbol if cell else " " for cell in row))
        return "\n---------\n".join(rows)


# =========================
# Score Tracker
# =========================

class ScoreTracker:
    def __init__(self):
        self.player_ratings: Dict[Player, int] = {}

    def report_game_result(
        self,
        player1: Player,
        player2: Player,
        winning_player: Optional[Player],
    ) -> None:
        if winning_player is not None:
            winner = winning_player
            loser = player2 if player1 == winner else player1

            self.player_ratings.setdefault(winner, 0)
            self.player_ratings[winner] += 1

            self.player_ratings.setdefault(loser, 0)
            self.player_ratings[loser] -= 1

    def get_top_players(self) -> Dict[Player, int]:
        sorted_items = sorted(
            self.player_ratings.items(),
            key=lambda entry: entry[1],
            reverse=True,
        )
        return {player: rating for player, rating in sorted_items}

    def get_rank(self, player: Player) -> int:
        sorted_players = [
            p for p, _ in sorted(
                self.player_ratings.items(),
                key=lambda entry: entry[1],
                reverse=True,
            )
        ]
        return sorted_players.index(player) + 1 if player in sorted_players else -1


# =========================
# Game
# Originator
# =========================

class Game:
    def __init__(self, player_x: Player, player_y: Player):
        self.board = Board()
        self.score_tracker = ScoreTracker()
        self.move_history = MoveHistory()
        self.players: List[Player] = []
        self.current_player_index = 0
        self.start_new_game(player_x, player_y)

    def start_new_game(self, player_x: Player, player_y: Player) -> None:
        self.board.reset()
        self.players = [player_x, player_y]
        self.current_player_index = 0
        self.move_history.clear_history()

    def make_move(self, col_index: int, row_index: int, player: Player) -> None:
        if self.get_game_status() == GameCondition.ENDED:
            raise ValueError("game ended")

        if self.players[self.current_player_index] != player:
            raise ValueError("not the current player")

        if self.board.get_player_at(col_index, row_index) is not None:
            raise ValueError("board position is taken")

        self.board.update_board(col_index, row_index, player)

        new_move = Move(col_index, row_index, player)
        self.move_history.record_move(new_move)

        self.current_player_index = (self.current_player_index + 1) % len(self.players)

        if self.get_game_status() == GameCondition.ENDED:
            self.score_tracker.report_game_result(
                self.players[0],
                self.players[1],
                self.board.get_winner(),
            )

    def undo_move(self) -> None:
        if self.move_history.is_empty():
            raise ValueError("no moves to undo")

        if self.current_player_index == 0:
            self.current_player_index = len(self.players) - 1
        else:
            self.current_player_index -= 1

        last_move = self.move_history.undo_move()
        self.board.update_board(last_move.col_index, last_move.row_index, None)

    def get_game_status(self) -> GameCondition:
        winner = self.board.get_winner()
        if winner is not None:
            return GameCondition.ENDED
        return GameCondition.ENDED if self.board.is_full() else GameCondition.IN_PROGRESS

    def get_current_player(self) -> Player:
        return self.players[self.current_player_index]

    def get_score_tracker(self) -> ScoreTracker:
        return self.score_tracker


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    player_x = Player("Alice", "X")
    player_y = Player("Bob", "O")

    game = Game(player_x, player_y)

    print("Current player:", game.get_current_player().name)

    game.make_move(0, 0, player_x)
    game.make_move(1, 0, player_y)
    game.make_move(0, 1, player_x)

    print("\nBoard after 3 moves:")
    print(game.board)

    game.undo_move()

    print("\nBoard after undo:")
    print(game.board)
    print("Current player after undo:", game.get_current_player().name)

    game.make_move(0, 1, player_x)
    game.make_move(1, 1, player_y)
    game.make_move(0, 2, player_x)

    print("\nFinal Board:")
    print(game.board)

    print("\nWinner:", game.board.get_winner())
    print("Game status:", game.get_game_status())

    tracker = game.get_score_tracker()
    print("\nRatings:", tracker.get_top_players())
    print("Rank of Alice:", tracker.get_rank(player_x))
    print("Rank of Bob:", tracker.get_rank(player_y))
```

## Design Patterns, SOLID Principles, and OOP Concepts Used in the Advanced Tic-Tac-Toe LLD

# 1. Design Patterns Used

## 1.1 Memento Pattern

**Where used**
- `Move`
- `MoveHistory`
- `Game`

**Why we used it**
We wanted to support **undo functionality**.

Each move stores:
- row or column position
- player who made the move

This move acts like a small snapshot of game state for reversal.

### Roles in Memento pattern
- **Memento** → `Move`
- **Caretaker** → `MoveHistory`
- **Originator** → `Game`

### How it works
- when a move is made, `Game` creates a `Move`
- `MoveHistory` stores it in stack order
- when undo is requested, `Game` gets the last move back and restores the board state

**Benefits**
- supports undo cleanly
- keeps history management separate
- does not expose board internals too much

**Simple reason**  
Memento is useful when we want to save and restore previous state.

---

## 1.2 Facade-like Game Coordinator

**Where used**
- `Game`

**Why we used it**
`Game` acts as the main controller for the Tic-Tac-Toe system.

Clients do not directly coordinate:
- `Board`
- `MoveHistory`
- `ScoreTracker`

Instead, they use:
- `make_move()`
- `undo_move()`
- `get_game_status()`
- `get_current_player()`
- `get_score_tracker()`

This gives a simple interface for playing the game.

**Simple reason**  
A facade style entry point makes the system easier to use.

---

## 1.3 State-machine-like Turn Flow

**Where used**
- `Game.current_player_index`
- `GameCondition`

**Why we used it**
The game follows a simple turn-based state flow:
- one player moves
- turn switches
- game continues until win or draw
- then game ends

This is not a full formal State pattern with separate state classes, but it uses a state-machine-like flow.

**Simple reason**  
The game has explicit turn progression and end conditions.

---

## 1.4 Composition Over Inheritance

**Where used**
- `Game` contains:
  - `Board`
  - `ScoreTracker`
  - `MoveHistory`
  - players
- `MoveHistory` contains `Move`

**Why we used it**
The system is built by combining focused objects rather than using inheritance trees.

Examples:
- `Game` does not inherit board logic
- `Game` does not inherit score logic
- `Game` does not inherit history logic

**Simple reason**  
Composition keeps the design modular and easier to maintain.

---

# 2. SOLID Principles Used

## 2.1 Single Responsibility Principle (SRP)

**Meaning**
A class should have one reason to change.

**Where used**

### `Player`
Only stores player identity and symbol.

### `Move`
Only stores move details.

### `MoveHistory`
Only stores and manages move history.

### `Board`
Only manages board state and winner checking.

### `ScoreTracker`
Only tracks ratings and ranks.

### `Game`
Only coordinates gameplay flow and turn handling.

**Why this is good**
Each class has a focused responsibility, which makes the design easier to understand and extend.

---

## 2.2 Open Closed Principle (OCP)

**Meaning**
Software should be open for extension but closed for modification.

**Where used**
The design can be extended later without major rewrites.

Examples:
- replace score logic with Elo rating
- extend board size to 4x4 or NxN
- add AI player support
- add redo support in move history
- add tournament management over multiple games

**Why this is good**
The design is small but still extensible.

---

## 2.3 Liskov Substitution Principle (LSP)

**Meaning**
Subclasses should be replaceable for their base type.

**Where used**
This design uses little inheritance, so LSP is less central here.  
Still, if future extensions introduce different player types or board strategies, they should remain substitutable.

**Why this is still worth mentioning**
The design is simple enough that inheritance is minimal, which avoids many LSP issues.

---

## 2.4 Interface Segregation Principle (ISP)

**Meaning**
Clients should not be forced to depend on methods they do not use.

**Where used**
There are no large interfaces in this design.  
The classes remain focused and small, so the design naturally stays close to ISP.

Examples:
- `MoveHistory` only exposes move history behavior
- `Board` only exposes board-related behavior
- `ScoreTracker` only exposes score-related behavior

**Why this is good**
The system stays clean without bloated contracts.

---

## 2.5 Dependency Inversion Principle (DIP)

**Meaning**
High level modules should depend on abstractions, not low level details.

**Where used**
This is only partially followed.

`Game` directly creates:
- `Board`
- `ScoreTracker`
- `MoveHistory`

A more advanced version could inject these dependencies through the constructor.

For example:
- inject `Board`
- inject `ScoreTracker`
- inject `MoveHistory`

**Why this is still acceptable**
For a small game system, direct composition keeps the design simple.  
But in a bigger system, dependency injection would improve flexibility and testing.

---

# 3. OOP Concepts Used

## 3.1 Encapsulation

**Where used**
Each class bundles related data and behavior together.

Examples:
- `Board` encapsulates grid updates, full board check, and winner check
- `MoveHistory` encapsulates stack behavior
- `ScoreTracker` encapsulates rating updates and ranking
- `Game` encapsulates turn flow and move orchestration

**Why this is useful**
The internal details stay inside the appropriate class.

---

## 3.2 Abstraction

**Where used**
There are no explicit interfaces in this version, but classes still abstract different responsibilities.

Examples:
- `Board` abstracts board management
- `ScoreTracker` abstracts player ranking
- `MoveHistory` abstracts undo support

**Why this is useful**
The rest of the system can work with these components without caring about internal implementation details.

---

## 3.3 Inheritance

**Where used**
Very little inheritance is used in this design.

**Why**
The system is simple and works better with composition.  
This is actually a good design choice for a small game model.

---

## 3.4 Polymorphism

**Where used**
Polymorphism is limited in this version because the design is intentionally simple.

Possible future places:
- human player vs AI player
- different score tracker strategies
- different board validation strategies

**Why this is still acceptable**
Not every good OO design needs heavy polymorphism. For small systems, keeping the model simple is often better.

---

## 3.5 Composition

**Where used**
- `Game` has `Board`, `ScoreTracker`, and `MoveHistory`
- `MoveHistory` has a stack of `Move`

**Why this is useful**
This is one of the strongest OOP aspects of the design because it keeps classes small and reusable.

---

# 4. Additional Good Design Choices

## 4.1 Board as 2D structure

**Where used**
- `Board.grid`

**Why**
A 2D list directly matches the spatial structure of Tic-Tac-Toe and gives O(1) access for updates and reads.

---

## 4.2 Immutable value objects

**Where used**
- `Player`
- `Move`

**Why**
These are simple data holders and should not change once created.

This reduces bugs and keeps the design safer.

---

## 4.3 Centralized move validation

**Where used**
- `Game.make_move()`

**Why**
Move rules are checked in one place:
- game not ended
- correct player turn
- target cell empty

This keeps the rule enforcement consistent.

---

## 4.4 Centralized undo tracking

**Where used**
- `MoveHistory`

**Why**
Undo logic is separated from the board itself, which makes the design cleaner and easier to extend later.

---

# 5. Final Summary

## Design Patterns Used
- **Memento Pattern**
- **Facade-like game coordinator**
- **State-machine-like turn flow**
- **Composition over inheritance**

## SOLID Principles Used
- **Single Responsibility Principle**
- **Open Closed Principle**
- **Interface Segregation Principle** loosely through focused classes
- **Dependency Inversion Principle** partially, with room for constructor injection
- **Liskov Substitution Principle** less central due to minimal inheritance

## OOP Concepts Used
- **Encapsulation**
- **Abstraction**
- **Composition**
- limited **Inheritance**
- limited **Polymorphism**

---

# 6. Interview Ready Answer

In this advanced Tic-Tac-Toe LLD, the most important pattern is the **Memento pattern**, which is used to implement undo functionality.  
`Move` acts as the memento, `MoveHistory` acts as the caretaker, and `Game` acts as the originator that creates and restores moves.

The design also uses strong separation of responsibilities: `Board` manages board state, `MoveHistory` manages undo history, `ScoreTracker` manages rankings, and `Game` coordinates the overall game flow. This aligns well with **SRP** and uses core OOP ideas like **encapsulation**, **abstraction**, and especially **composition**, which is the strongest design choice in this implementation.