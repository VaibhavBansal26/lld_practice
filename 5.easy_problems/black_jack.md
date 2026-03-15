# 11. Black Jack Game

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from enum import Enum
from typing import Dict, List, Optional, Set
import random


# =========================
# Enums
# =========================

class Suit(str, Enum):
    HEARTS = "HEARTS"
    SPADES = "SPADES"
    CLUBS = "CLUBS"
    DIAMONDS = "DIAMONDS"


class Rank(Enum):
    ACE = (1, 11)
    TWO = (2,)
    THREE = (3,)
    FOUR = (4,)
    FIVE = (5,)
    SIX = (6,)
    SEVEN = (7,)
    EIGHT = (8,)
    NINE = (9,)
    TEN = (10,)
    JACK = (10,)
    QUEEN = (10,)
    KING = (10,)

    def __init__(self, *rank_values: int):
        self.rank_values = rank_values

    def get_rank_values(self) -> List[int]:
        return list(self.rank_values)


class Action(str, Enum):
    HIT = "HIT"
    STAND = "STAND"


class GamePhase(str, Enum):
    STARTED = "STARTED"
    BET_PLACED = "BET_PLACED"
    INITIAL_CARD_DRAWN = "INITIAL_CARD_DRAWN"
    END = "END"


# =========================
# Card
# =========================

@dataclass(frozen=True)
class Card:
    rank: Rank
    suit: Suit

    def get_rank_values(self) -> List[int]:
        return self.rank.get_rank_values()


# =========================
# Deck
# =========================

class Deck:
    def __init__(self):
        self.next_card_index = 0
        self.cards: List[Card] = []
        self.initialize_deck()

    def initialize_deck(self) -> None:
        self.cards = []
        for suit in Suit:
            for rank in Rank:
                self.cards.append(Card(rank, suit))
        self.next_card_index = 0

    def shuffle(self) -> None:
        random.shuffle(self.cards)

    def draw(self) -> Card:
        if self.is_empty() or self.next_card_index >= len(self.cards):
            raise ValueError("No more cards in deck")

        drawn_card = self.cards[self.next_card_index]
        self.next_card_index += 1
        return drawn_card

    def get_remaining_card_count(self) -> int:
        return len(self.cards) - self.next_card_index

    def is_empty(self) -> bool:
        return self.get_remaining_card_count() == 0

    def reset(self) -> None:
        self.next_card_index = 0


# =========================
# Hand
# =========================

class Hand:
    def __init__(self):
        self.hand_cards: List[Card] = []
        self.possible_values: Set[int] = set()

    def add_card(self, card: Card) -> None:
        if card is None:
            raise ValueError("Cannot add null card to hand")

        self.hand_cards.append(card)

        if not self.possible_values:
            for value in card.get_rank_values():
                self.possible_values.add(value)
        else:
            new_possible_values: Set[int] = set()
            for value in self.possible_values:
                for card_value in card.get_rank_values():
                    new_possible_values.add(value + card_value)
            self.possible_values = new_possible_values

    def get_cards(self) -> List[Card]:
        return list(self.hand_cards)

    def get_possible_values(self) -> List[int]:
        return sorted(self.possible_values)

    def clear(self) -> None:
        self.hand_cards.clear()
        self.possible_values.clear()

    def is_bust(self) -> bool:
        if not self.possible_values:
            return False
        return min(self.possible_values) > 21

    def get_best_value(self) -> int:
        if not self.possible_values:
            return 0

        valid_values = [value for value in self.possible_values if value <= 21]
        if valid_values:
            return max(valid_values)
        return min(self.possible_values)


# =========================
# Player Interface
# =========================

class Player(ABC):
    @abstractmethod
    def bet(self, bet: int) -> None:
        pass

    @abstractmethod
    def lose_bet(self) -> None:
        pass

    @abstractmethod
    def return_bet(self) -> None:
        pass

    @abstractmethod
    def payout(self) -> None:
        pass

    @abstractmethod
    def is_bust(self) -> bool:
        pass

    @abstractmethod
    def get_hand(self) -> Hand:
        pass

    @abstractmethod
    def get_balance(self) -> int:
        pass

    @abstractmethod
    def get_name(self) -> str:
        pass

    @abstractmethod
    def get_bet(self) -> int:
        pass


# =========================
# Real Player
# =========================

class RealPlayer(Player):
    def __init__(self, name: str, start_balance: int):
        self.name = name
        self.hand = Hand()
        self.current_bet = 0
        self.balance = start_balance

    def bet(self, bet: int) -> None:
        if bet > self.balance:
            raise ValueError("Bet is greater than balance")
        self.current_bet = bet
        self.balance -= bet

    def lose_bet(self) -> None:
        self.current_bet = 0

    def return_bet(self) -> None:
        self.balance += self.current_bet
        self.current_bet = 0

    def payout(self) -> None:
        self.balance += self.current_bet * 2
        self.current_bet = 0

    def is_bust(self) -> bool:
        return self.hand.is_bust()

    def get_hand(self) -> Hand:
        return self.hand

    def get_balance(self) -> int:
        return self.balance

    def get_name(self) -> str:
        return self.name

    def get_bet(self) -> int:
        return self.current_bet

    def __repr__(self) -> str:
        return f"RealPlayer(name={self.name}, balance={self.balance}, bet={self.current_bet})"


# =========================
# Dealer Player
# =========================

class DealerPlayer(Player):
    def __init__(self):
        self.name = "Dealer"
        self.hand = Hand()

    def bet(self, bet: int) -> None:
        pass

    def lose_bet(self) -> None:
        pass

    def return_bet(self) -> None:
        pass

    def payout(self) -> None:
        pass

    def is_bust(self) -> bool:
        return self.hand.is_bust()

    def get_hand(self) -> Hand:
        return self.hand

    def get_balance(self) -> int:
        return 0

    def get_name(self) -> str:
        return self.name

    def get_bet(self) -> int:
        return 0

    def __repr__(self) -> str:
        return "DealerPlayer()"


# =========================
# Blackjack Game
# =========================

class BlackJackGame:
    def __init__(self, players: List[Player]):
        self.deck = Deck()
        self.players: List[Player] = []
        self.dealer: Player = DealerPlayer()
        self.current_player: Optional[Player] = None
        self.player_turn_status_map: Dict[Player, Optional[Action]] = {}
        self.current_phase = GamePhase.STARTED

        for player in players:
            if player is None:
                raise ValueError("Player cannot be null")
            self.players.append(player)
            self.player_turn_status_map[player] = None

        self.player_turn_status_map[self.dealer] = None
        self.deck.shuffle()

    def get_next_eligible_player(self) -> Optional[Player]:
        if (
            self.current_player is not None
            and self.player_turn_status_map.get(self.current_player) != Action.STAND
            and not self.current_player.is_bust()
        ):
            return self.current_player

        if self.current_player is None:
            for player in self.players:
                if self.player_turn_status_map.get(player) != Action.STAND and not player.is_bust():
                    self.current_player = player
                    return self.current_player

        if self.current_player in self.players:
            current_player_index = self.players.index(self.current_player)
        else:
            current_player_index = -1

        for i in range(current_player_index + 1, len(self.players)):
            player = self.players[i]
            if self.player_turn_status_map.get(player) != Action.STAND and not player.is_bust():
                self.current_player = player
                return self.current_player

        if self.player_turn_status_map.get(self.dealer) != Action.STAND:
            self.current_player = self.dealer
            self.dealer_turn()
            return None

        return None

    def dealer_turn(self) -> None:
        while self.dealer.get_hand().get_best_value() < 17:
            new_draw = self.deck.draw()
            self.dealer.get_hand().add_card(new_draw)

        self.player_turn_status_map[self.dealer] = Action.STAND
        self.check_game_end_condition()

    def start_new_round(self) -> None:
        self.deck.reset()
        self.deck.shuffle()

        for player in self.player_turn_status_map.keys():
            player.get_hand().clear()

        self.player_turn_status_map = {player: None for player in self.players}
        self.player_turn_status_map[self.dealer] = None
        self.current_player = None
        self.current_phase = GamePhase.STARTED

    def deal_initial_cards(self) -> None:
        if self.current_phase != GamePhase.BET_PLACED:
            raise ValueError("All players must bet before dealing")

        for player in self.players:
            player.get_hand().add_card(self.deck.draw())

        self.dealer.get_hand().add_card(self.deck.draw())

        for player in self.players:
            player.get_hand().add_card(self.deck.draw())

        self.dealer.get_hand().add_card(self.deck.draw())

        self.current_phase = GamePhase.INITIAL_CARD_DRAWN

    def bet(self, player: Player, bet: int) -> None:
        if self.current_phase != GamePhase.STARTED:
            raise ValueError("Bets must be placed at the start of the round")

        player.bet(bet)

        all_players_bet = all(
            not isinstance(p, DealerPlayer) and p.get_bet() > 0
            for p in self.players
        )
        if all_players_bet:
            self.current_phase = GamePhase.BET_PLACED

    def hit(self, player: Player) -> None:
        if self.player_turn_status_map.get(player) == Action.STAND:
            raise ValueError("Player has already stood")

        if player.is_bust():
            raise ValueError("Player is already bust")

        drawn_card = self.deck.draw()
        player.get_hand().add_card(drawn_card)
        self.player_turn_status_map[player] = Action.HIT

    def stand(self, player: Player) -> None:
        if self.player_turn_status_map.get(player) == Action.STAND:
            raise ValueError("Player has already stood")

        if player.is_bust():
            raise ValueError("Player is already bust")

        self.player_turn_status_map[player] = Action.STAND

    def check_game_end_condition(self) -> None:
        all_players_done = all(
            self.player_turn_status_map.get(player) == Action.STAND or player.is_bust()
            for player in self.players
        )

        if not all_players_done:
            return

        dealer_value = self.dealer.get_hand().get_best_value()
        dealer_busts = self.dealer.is_bust()

        for player in self.players:
            if player.is_bust():
                player.lose_bet()
            else:
                player_value = player.get_hand().get_best_value()
                if dealer_busts or player_value > dealer_value:
                    player.payout()
                elif player_value == dealer_value:
                    player.return_bet()
                else:
                    player.lose_bet()

        self.current_phase = GamePhase.END

    def show_hand(self, player: Player) -> str:
        cards = [f"{card.rank.name} of {card.suit.name}" for card in player.get_hand().get_cards()]
        values = player.get_hand().get_possible_values()
        return f"{player.get_name()}: {cards}, values={values}, best={player.get_hand().get_best_value()}"


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    player1 = RealPlayer("Alice", 100)
    player2 = RealPlayer("Bob", 100)

    game = BlackJackGame([player1, player2])

    game.bet(player1, 20)
    game.bet(player2, 30)

    game.deal_initial_cards()

    print("Initial hands:")
    print(game.show_hand(player1))
    print(game.show_hand(player2))
    print(game.show_hand(game.dealer))
    print()

    current_player = game.get_next_eligible_player()
    while current_player is not None:
        print(f"Current player: {current_player.get_name()}")

        # simple demo logic: hit if under 16, otherwise stand
        if current_player.get_hand().get_best_value() < 16:
            game.hit(current_player)
            print(f"{current_player.get_name()} hits")
        else:
            game.stand(current_player)
            print(f"{current_player.get_name()} stands")

        print(game.show_hand(current_player))
        print()

        current_player = game.get_next_eligible_player()

    # finalize if dealer already played
    game.check_game_end_condition()

    print("Final hands:")
    print(game.show_hand(player1))
    print(game.show_hand(player2))
    print(game.show_hand(game.dealer))
    print()

    print("Balances after round:")
    print(player1.get_name(), player1.get_balance())
    print(player2.get_name(), player2.get_balance())
    print("Game phase:", game.current_phase.value)
```

DEEP DIVE

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass
from enum import Enum
from typing import Dict, List, Optional, Set
import random


# =========================
# Enums
# =========================

class Suit(str, Enum):
    HEARTS = "HEARTS"
    SPADES = "SPADES"
    CLUBS = "CLUBS"
    DIAMONDS = "DIAMONDS"


class Rank(Enum):
    ACE = (1, 11)
    TWO = (2,)
    THREE = (3,)
    FOUR = (4,)
    FIVE = (5,)
    SIX = (6,)
    SEVEN = (7,)
    EIGHT = (8,)
    NINE = (9,)
    TEN = (10,)
    JACK = (10,)
    QUEEN = (10,)
    KING = (10,)

    def __init__(self, *rank_values: int):
        self.rank_values = rank_values

    def get_rank_values(self) -> List[int]:
        return list(self.rank_values)


class Action(str, Enum):
    HIT = "HIT"
    STAND = "STAND"


class GamePhase(str, Enum):
    STARTED = "STARTED"
    BET_PLACED = "BET_PLACED"
    INITIAL_CARD_DRAWN = "INITIAL_CARD_DRAWN"
    END = "END"


# =========================
# Card
# =========================

@dataclass(frozen=True)
class Card:
    rank: Rank
    suit: Suit

    def get_rank_values(self) -> List[int]:
        return self.rank.get_rank_values()


# =========================
# Deck
# =========================

class Deck:
    def __init__(self):
        self.next_card_index = 0
        self.cards: List[Card] = []
        self.initialize_deck()

    def initialize_deck(self) -> None:
        self.cards = []
        for suit in Suit:
            for rank in Rank:
                self.cards.append(Card(rank, suit))
        self.next_card_index = 0

    def shuffle(self) -> None:
        random.shuffle(self.cards)

    def draw(self) -> Card:
        if self.is_empty() or self.next_card_index >= len(self.cards):
            raise ValueError("No more cards in deck")

        drawn_card = self.cards[self.next_card_index]
        self.next_card_index += 1
        return drawn_card

    def get_remaining_card_count(self) -> int:
        return len(self.cards) - self.next_card_index

    def is_empty(self) -> bool:
        return self.get_remaining_card_count() == 0

    def reset(self) -> None:
        self.next_card_index = 0


# =========================
# Hand
# =========================

class Hand:
    def __init__(self):
        self.hand_cards: List[Card] = []
        self.possible_values: Set[int] = set()

    def add_card(self, card: Card) -> None:
        if card is None:
            raise ValueError("Cannot add null card to hand")

        self.hand_cards.append(card)

        if not self.possible_values:
            for value in card.get_rank_values():
                self.possible_values.add(value)
        else:
            new_possible_values: Set[int] = set()
            for value in self.possible_values:
                for card_value in card.get_rank_values():
                    new_possible_values.add(value + card_value)
            self.possible_values = new_possible_values

    def get_cards(self) -> List[Card]:
        return list(self.hand_cards)

    def get_possible_values(self) -> List[int]:
        return sorted(self.possible_values)

    def clear(self) -> None:
        self.hand_cards.clear()
        self.possible_values.clear()

    def is_bust(self) -> bool:
        if not self.possible_values:
            return False
        return min(self.possible_values) > 21

    def get_best_value(self) -> int:
        if not self.possible_values:
            return 0

        valid_values = [value for value in self.possible_values if value <= 21]
        if valid_values:
            return max(valid_values)
        return min(self.possible_values)


# =========================
# Decision Logic Strategy
# =========================

class PlayerDecisionLogic(ABC):
    @abstractmethod
    def decide_action(self, hand: Hand) -> Action:
        pass


class RealPlayerDecisionLogic(PlayerDecisionLogic):
    def decide_action(self, hand: Hand) -> Action:
        return Action.HIT if hand.get_best_value() < 16 else Action.STAND


class DealerDecisionLogic(PlayerDecisionLogic):
    def decide_action(self, hand: Hand) -> Action:
        return Action.HIT if hand.get_best_value() < 17 else Action.STAND


# =========================
# Player Interface
# =========================

class Player(ABC):
    @abstractmethod
    def bet(self, bet: int) -> None:
        pass

    @abstractmethod
    def lose_bet(self) -> None:
        pass

    @abstractmethod
    def return_bet(self) -> None:
        pass

    @abstractmethod
    def payout(self) -> None:
        pass

    @abstractmethod
    def is_bust(self) -> bool:
        pass

    @abstractmethod
    def get_hand(self) -> Hand:
        pass

    @abstractmethod
    def get_balance(self) -> int:
        pass

    @abstractmethod
    def get_name(self) -> str:
        pass

    @abstractmethod
    def get_bet(self) -> int:
        pass

    @abstractmethod
    def get_decision_logic(self) -> PlayerDecisionLogic:
        pass


# =========================
# Real Player
# =========================

class RealPlayer(Player):
    def __init__(
        self,
        name: str,
        start_balance: int,
        decision_logic: Optional[PlayerDecisionLogic] = None,
    ):
        self.name = name
        self.hand = Hand()
        self.current_bet = 0
        self.balance = start_balance
        self.decision_logic = decision_logic or RealPlayerDecisionLogic()

    def bet(self, bet: int) -> None:
        if bet > self.balance:
            raise ValueError("Bet is greater than balance")
        self.current_bet = bet
        self.balance -= bet

    def lose_bet(self) -> None:
        self.current_bet = 0

    def return_bet(self) -> None:
        self.balance += self.current_bet
        self.current_bet = 0

    def payout(self) -> None:
        self.balance += self.current_bet * 2
        self.current_bet = 0

    def is_bust(self) -> bool:
        return self.hand.is_bust()

    def get_hand(self) -> Hand:
        return self.hand

    def get_balance(self) -> int:
        return self.balance

    def get_name(self) -> str:
        return self.name

    def get_bet(self) -> int:
        return self.current_bet

    def get_decision_logic(self) -> PlayerDecisionLogic:
        return self.decision_logic

    def __repr__(self) -> str:
        return f"RealPlayer(name={self.name}, balance={self.balance}, bet={self.current_bet})"


# =========================
# Dealer Player
# =========================

class DealerPlayer(Player):
    def __init__(self, decision_logic: Optional[PlayerDecisionLogic] = None):
        self.name = "Dealer"
        self.hand = Hand()
        self.decision_logic = decision_logic or DealerDecisionLogic()

    def bet(self, bet: int) -> None:
        pass

    def lose_bet(self) -> None:
        pass

    def return_bet(self) -> None:
        pass

    def payout(self) -> None:
        pass

    def is_bust(self) -> bool:
        return self.hand.is_bust()

    def get_hand(self) -> Hand:
        return self.hand

    def get_balance(self) -> int:
        return 0

    def get_name(self) -> str:
        return self.name

    def get_bet(self) -> int:
        return 0

    def get_decision_logic(self) -> PlayerDecisionLogic:
        return self.decision_logic

    def __repr__(self) -> str:
        return "DealerPlayer()"


# =========================
# Blackjack Game
# =========================

class BlackJackGame:
    def __init__(self, players: List[Player]):
        self.deck = Deck()
        self.players: List[Player] = []
        self.dealer: Player = DealerPlayer()
        self.current_player: Optional[Player] = None
        self.player_turn_status_map: Dict[Player, Optional[Action]] = {}
        self.current_phase = GamePhase.STARTED

        for player in players:
            if player is None:
                raise ValueError("Player cannot be null")
            self.players.append(player)
            self.player_turn_status_map[player] = None

        self.player_turn_status_map[self.dealer] = None
        self.deck.shuffle()

    def get_next_eligible_player(self) -> Optional[Player]:
        if self.current_player is None:
            for player in self.players:
                if self.player_turn_status_map.get(player) != Action.STAND and not player.is_bust():
                    self.current_player = player
                    return self.current_player

            if self.player_turn_status_map.get(self.dealer) != Action.STAND:
                self.current_player = self.dealer
                return self.dealer
        else:
            current_index = self.players.index(self.current_player) if self.current_player in self.players else -1

            for i in range(current_index + 1, len(self.players)):
                player = self.players[i]
                if self.player_turn_status_map.get(player) != Action.STAND and not player.is_bust():
                    self.current_player = player
                    return self.current_player

            if self.current_player != self.dealer and self.player_turn_status_map.get(self.dealer) != Action.STAND:
                self.current_player = self.dealer
                return self.dealer

        return None

    def play_next_turn(self) -> None:
        next_player = self.get_next_eligible_player()
        if next_player is not None:
            self.perform_player_action(next_player)
        else:
            self.check_game_end_condition()

    def perform_player_action(self, player: Player) -> None:
        if player.is_bust():
            return

        action = player.get_decision_logic().decide_action(player.get_hand())

        if action == Action.HIT:
            self.hit(player)
        elif action == Action.STAND:
            self.stand(player)

        if player == self.dealer and self.player_turn_status_map.get(self.dealer) == Action.STAND:
            self.check_game_end_condition()

    def start_new_round(self) -> None:
        self.deck.reset()
        self.deck.shuffle()

        for player in self.player_turn_status_map.keys():
            player.get_hand().clear()

        self.player_turn_status_map = {player: None for player in self.players}
        self.player_turn_status_map[self.dealer] = None
        self.current_player = None
        self.current_phase = GamePhase.STARTED

    def deal_initial_cards(self) -> None:
        if self.current_phase != GamePhase.BET_PLACED:
            raise ValueError("All players must bet before dealing")

        for player in self.players:
            player.get_hand().add_card(self.deck.draw())

        self.dealer.get_hand().add_card(self.deck.draw())

        for player in self.players:
            player.get_hand().add_card(self.deck.draw())

        self.dealer.get_hand().add_card(self.deck.draw())

        self.current_phase = GamePhase.INITIAL_CARD_DRAWN

    def bet(self, player: Player, bet: int) -> None:
        if self.current_phase != GamePhase.STARTED:
            raise ValueError("Bets must be placed at the start of the round")

        player.bet(bet)

        all_players_bet = all(
            not isinstance(p, DealerPlayer) and p.get_bet() > 0
            for p in self.players
        )
        if all_players_bet:
            self.current_phase = GamePhase.BET_PLACED

    def hit(self, player: Player) -> None:
        if self.player_turn_status_map.get(player) == Action.STAND:
            raise ValueError("Player has already stood")

        if player.is_bust():
            raise ValueError("Player is already bust")

        drawn_card = self.deck.draw()
        player.get_hand().add_card(drawn_card)
        self.player_turn_status_map[player] = Action.HIT

        if player.is_bust():
            # player is done once bust
            pass

    def stand(self, player: Player) -> None:
        if self.player_turn_status_map.get(player) == Action.STAND:
            raise ValueError("Player has already stood")

        if player.is_bust():
            raise ValueError("Player is already bust")

        self.player_turn_status_map[player] = Action.STAND

    def check_game_end_condition(self) -> None:
        all_players_done = all(
            self.player_turn_status_map.get(player) == Action.STAND or player.is_bust()
            for player in self.players
        )

        dealer_done = self.player_turn_status_map.get(self.dealer) == Action.STAND or self.dealer.is_bust()

        if not all_players_done or not dealer_done:
            return

        dealer_value = self.dealer.get_hand().get_best_value()
        dealer_busts = self.dealer.is_bust()

        for player in self.players:
            if player.is_bust():
                player.lose_bet()
            else:
                player_value = player.get_hand().get_best_value()
                if dealer_busts or player_value > dealer_value:
                    player.payout()
                elif player_value == dealer_value:
                    player.return_bet()
                else:
                    player.lose_bet()

        self.current_phase = GamePhase.END

    def show_hand(self, player: Player) -> str:
        cards = [f"{card.rank.name} of {card.suit.name}" for card in player.get_hand().get_cards()]
        values = player.get_hand().get_possible_values()
        return f"{player.get_name()}: {cards}, values={values}, best={player.get_hand().get_best_value()}"


# =========================
# Example Usage
# =========================

if __name__ == "__main__":
    player1 = RealPlayer("Alice", 100)
    player2 = RealPlayer("Bob", 100)

    game = BlackJackGame([player1, player2])

    game.bet(player1, 20)
    game.bet(player2, 30)

    game.deal_initial_cards()

    print("Initial hands:")
    print(game.show_hand(player1))
    print(game.show_hand(player2))
    print(game.show_hand(game.dealer))
    print()

    while game.current_phase != GamePhase.END:
        next_player = game.get_next_eligible_player()
        if next_player is None:
            game.check_game_end_condition()
            break

        print(f"Current turn: {next_player.get_name()}")
        game.play_next_turn()
        print(game.show_hand(next_player))
        print()

        if next_player == game.dealer and game.player_turn_status_map.get(game.dealer) == Action.STAND:
            break

    game.check_game_end_condition()

    print("Final hands:")
    print(game.show_hand(player1))
    print(game.show_hand(player2))
    print(game.show_hand(game.dealer))
    print()

    print("Balances after round:")
    print(player1.get_name(), player1.get_balance())
    print(player2.get_name(), player2.get_balance())
    print("Game phase:", game.current_phase.value)
```

## Design Patterns, SOLID Principles, and OOP Concepts Used in the Advanced Blackjack LLD

# 1. Design Patterns Used

## 1.1 Strategy Pattern

**Where used**
- `PlayerDecisionLogic`
- `RealPlayerDecisionLogic`
- `DealerDecisionLogic`

**Why we used it**
Decision making in Blackjack can vary by player type.

Examples:
- a normal player may hit below 16
- the dealer must hit below 17
- an aggressive player may hit below 18
- an AI player may use probability based logic

Instead of hardcoding all decision rules inside `BlackJackGame`, we encapsulate each decision style in its own class.

**Benefits**
- decision making becomes pluggable
- easier to test different playing strategies
- easier to add AI or house rule variations
- `BlackJackGame` stays focused on turn coordination

**Simple reason**  
When behavior can vary independently, Strategy is a strong fit.

---

## 1.2 Facade Pattern

**Where used**
- `BlackJackGame`

**Why we used it**
`BlackJackGame` acts as the central entry point for the whole game flow.

Clients do not directly coordinate:
- `Deck`
- `Hand`
- `Player`
- `DealerPlayer`
- turn status map
- betting resolution

Instead, they use methods like:
- `bet()`
- `deal_initial_cards()`
- `play_next_turn()`
- `hit()`
- `stand()`
- `start_new_round()`

This hides lower level complexity and presents a simpler interface.

**Simple reason**  
Facade makes a multi-component game system easier to use.

---

## 1.3 State-machine-like Turn Flow

**Where used**
- `GamePhase`
- `Action`
- `current_phase`
- `player_turn_status_map`

**Why we used it**
Blackjack progresses through clear stages:

- round started
- bets placed
- initial cards drawn
- player turns
- dealer turn
- result resolution
- round end

This is not a full formal State pattern with separate state classes, but it does use explicit state transitions to control the flow safely.

**Simple reason**  
The game has a structured lifecycle, so explicit phase handling helps keep behavior correct.

---

## 1.4 Composition Over Inheritance

**Where used**
- `BlackJackGame` contains:
  - `Deck`
  - `players`
  - `dealer`
  - turn status map
- `Player` implementations contain:
  - `Hand`
  - decision logic
- `Card` contains:
  - `Rank`
  - `Suit`

**Why we used it**
Instead of building large inheritance chains, the design uses smaller focused objects working together.

Examples:
- a player does not inherit hand logic, it owns a `Hand`
- the game does not inherit deck logic, it owns a `Deck`
- players do not hardcode decisions, they own a strategy object

**Simple reason**  
Composition keeps the system modular and easier to extend.

---

## 1.5 Template-like Shared Player Contract

**Where used**
- `Player` abstraction
- `RealPlayer`
- `DealerPlayer`

**Why we used it**
Both human players and dealer share common operations:
- get hand
- determine bust
- payout related methods
- get name
- expose decision logic

This provides a common contract while still allowing different behaviors.

**Simple reason**  
A shared player interface makes the game logic treat all participants uniformly.

---

# 2. SOLID Principles Used

## 2.1 Single Responsibility Principle (SRP)

**Meaning**
A class should have one reason to change.

**Where used**

### `Card`
Only stores rank and suit.

### `Deck`
Only manages card initialization, shuffle, draw, and reset.

### `Hand`
Only manages cards and possible hand values.

### `RealPlayer`
Only manages player balance, betting, and personal hand state.

### `DealerPlayer`
Only represents dealer specific behavior.

### `PlayerDecisionLogic`
Only handles decision making.

### `BlackJackGame`
Only coordinates the overall game flow.

**Why this is good**
Each class remains focused and easier to understand, test, and maintain.

---

## 2.2 Open Closed Principle (OCP)

**Meaning**
Software should be open for extension but closed for modification.

**Where used**
The design allows new behaviors without major changes to stable logic.

Examples:
- new player decision strategies
- new player types
- new payout rules
- new deck variants
- new game phases or side rules

Adding a new decision policy only requires creating another `PlayerDecisionLogic` implementation.

**Why this is good**
The game becomes extensible without rewriting core orchestration logic.

---

## 2.3 Liskov Substitution Principle (LSP)

**Meaning**
Subclasses or implementations should be replaceable for their base type.

**Where used**
Any implementation of `Player` should work correctly in the game flow.

Examples:
- `RealPlayer`
- `DealerPlayer`

Also, any implementation of `PlayerDecisionLogic` can replace another where decision behavior is expected.

Examples:
- `RealPlayerDecisionLogic`
- `DealerDecisionLogic`

**Why this is good**
The system works correctly with interchangeable valid implementations.

---

## 2.4 Interface Segregation Principle (ISP)

**Meaning**
Clients should not be forced to depend on methods they do not use.

**Where used**
The abstractions are fairly small and focused.

### `PlayerDecisionLogic`
Only defines:
- `decide_action()`

### `Player`
Defines behavior the game needs from any participant.

This keeps contracts focused and practical.

**Why this is good**
Implementations only need to support relevant behavior.

---

## 2.5 Dependency Inversion Principle (DIP)

**Meaning**
High level modules should depend on abstractions, not concrete implementations.

**Where used**
- `BlackJackGame` interacts with `Player` abstraction instead of directly depending only on `RealPlayer`
- player behavior depends on `PlayerDecisionLogic` abstraction instead of hardcoded decision rules

**Where it can improve**
`BlackJackGame` still directly creates `Deck` and `DealerPlayer`.  
A more advanced version could inject:
- deck abstraction
- dealer abstraction
- payout strategy abstraction

**Why this is still good**
The decision-making logic already demonstrates strong DIP by separating high-level game flow from concrete action rules.

---

# 3. OOP Concepts Used

## 3.1 Encapsulation

**Where used**
Each class groups its own data and behavior.

Examples:
- `Hand` encapsulates cards and possible totals
- `Deck` encapsulates drawing and shuffling
- `RealPlayer` encapsulates balance and betting
- `BlackJackGame` encapsulates game flow and phase transitions

**Why this is useful**
Internal logic stays in the right place instead of being spread across the system.

---

## 3.2 Abstraction

**Where used**
- `Player`
- `PlayerDecisionLogic`

These abstractions define required behavior without exposing implementation details.

**Why this is useful**
The game logic can work with general player and decision behavior, not hardcoded concrete classes.

---

## 3.3 Inheritance

**Where used**
Concrete player classes implement `Player`:
- `RealPlayer`
- `DealerPlayer`

Concrete decision logic classes implement `PlayerDecisionLogic`:
- `RealPlayerDecisionLogic`
- `DealerDecisionLogic`

**Why this is useful**
It allows different implementations to share the same contract.

---

## 3.4 Polymorphism

**Where used**
The game calls:
- `player.get_decision_logic().decide_action(player.get_hand())`

The actual action depends on which decision logic object is attached to the player.

Also:
- `RealPlayer` and `DealerPlayer` are both treated as `Player`

**Why this is useful**
The same method call can produce different behavior depending on the concrete implementation.

---

## 3.5 Composition

**Where used**
- `BlackJackGame` has `Deck`, `players`, and `dealer`
- `RealPlayer` has `Hand` and `PlayerDecisionLogic`
- `DealerPlayer` has `Hand` and `PlayerDecisionLogic`
- `Card` has `Rank` and `Suit`

**Why this is useful**
The game is built out of reusable focused objects instead of relying on large inheritance structures.

---

# 4. Additional Good Design Choices

## 4.1 Efficient Deck Draw Handling

**Where used**
- `Deck.next_card_index`

**Why**
Instead of removing cards from the list, the deck advances an index.  
This keeps card drawing O(1) and avoids expensive list shifting.

---

## 4.2 Correct Ace Handling

**Where used**
- `Hand.possible_values`

**Why**
Aces can be worth 1 or 11, so the hand tracks all possible totals.  
This avoids fragile recalculation logic later.

---

## 4.3 Best Value Selection

**Where used**
- `Hand.get_best_value()`

**Why**
The game needs the best valid hand total under 21, not simply the maximum raw total.  
This makes winner resolution more accurate.

---

## 4.4 Centralized Bet Resolution

**Where used**
- `BlackJackGame.check_game_end_condition()`

**Why**
All bet outcomes are resolved in one place, which makes the result logic clearer and easier to maintain.

---

# 5. Final Summary

## Design Patterns Used
- **Strategy Pattern**
- **Facade Pattern**
- **State-machine-like turn flow**
- **Composition over inheritance**
- **Shared player contract abstraction**

## SOLID Principles Used
- **Single Responsibility Principle**
- **Open Closed Principle**
- **Liskov Substitution Principle**
- **Interface Segregation Principle**
- **Dependency Inversion Principle**

## OOP Concepts Used
- **Encapsulation**
- **Abstraction**
- **Inheritance**
- **Polymorphism**
- **Composition**

---

# 6. Interview Ready Answer

In this advanced Blackjack LLD, the most important pattern is the **Strategy pattern**, because player and dealer decisions are no longer hardcoded inside the game.  
Instead, each participant uses a `PlayerDecisionLogic` implementation, such as `RealPlayerDecisionLogic` or `DealerDecisionLogic`, which makes the system much more flexible and extensible.

The design also uses **Facade** through `BlackJackGame`, which coordinates the entire round while hiding deck management, betting, and player turn complexity. It follows strong OOP principles like **encapsulation**, **abstraction**, **inheritance**, **polymorphism**, and **composition**. It aligns well with SOLID, especially **SRP**, **OCP**, and **DIP**, because each class has a clear role, behaviors are extensible, and decision rules depend on abstractions instead of being tightly coupled to the game flow.