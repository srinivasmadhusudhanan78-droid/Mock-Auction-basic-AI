# Mock-Auction-basic-AI
# Mock-Auction-basic
# Auction Engine Monolith

import random
import re
import json
import os
import requests
from bs4 import BeautifulSoup
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Tuple
from enum import Enum

# -------------------------------
# Util helpers
# -------------------------------
CRORE = 1.0  # represent all amounts in crores as float

def fmt_cr(amount: float) -> str:
    return f"{amount:.2f}Cr"

def round_to_increment(amount: float) -> float:
    if amount <= 9.0:
        tick = 0.25
        return round(amount / tick) * tick
    else:
        tick = 0.5
        return round(amount / tick) * tick

def next_increment(current: float) -> float:
    return 0.25 if current < 9.0 else 0.5

def clamp(value: float, lo: float, hi: float) -> float:
    return max(lo, min(hi, value))

def in_band(value: float, lo: float, hi: float) -> bool:
    return lo <= value <= hi

def safe_money(value: float) -> float:
    return round(value + 1e-9, 2)

# -------------------------------
# Models
# -------------------------------
class Role(str, Enum):
    B = "B"   # Batter
    W = "W"   # Wicket-keeper
    A = "A"   # Allrounder
    P = "P"   # Pace
    S = "S"   # Spin

TIERS = [
    (1, 0.30, 1.50),
    (2, 1.50, 2.75),
    (3, 3.00, 4.75),
    (4, 5.00, 6.50),
    (5, 7.00, 8.50),
    (6, 9.00,10.50),
    (7,11.00,12.50),
    (8,13.00,15.50),
    (9,16.00,18.00),
]

@dataclass
class Player:
    name: str
    slot: str
    base_price: float
    previous_team: Optional[str]
    age: Optional[int]
    role: Optional[Role]
    tier_id: Optional[int] = None
    tier_lo: Optional[float] = None
    tier_hi: Optional[float] = None
    hidden_reserve: Optional[float] = None
    stats: Dict[str, str] = field(default_factory=dict)

@dataclass
class Team:
    name: str
    total_slots: int
    overseas_slots: int
    purse: float
    rtm_available: int
    squad: List[Player] = field(default_factory=list)
    overseas_count: int = 0

    def can_afford(self, price: float) -> bool:
        return self.purse >= price

    def has_slot(self) -> bool:
        return len(self.squad) < self.total_slots

    def add_player(self, player: Player, price: float, is_overseas: bool):
        self.squad.append(player)
        self.purse = round(self.purse - price, 2)
        if is_overseas:
            self.overseas_count += 1

@dataclass
class Constraints:
    min_B: int = 5
    min_PS: int = 5
    min_A: int = 4
    min_W: int = 2

@dataclass
class Bid:
    team: str
    amount: float

@dataclass
class SaleResult:
    player: Player
    sold_to: Optional[str]
    price: Optional[float]
    role: Role
    rtm_used: bool = False
    reason: Optional[str] = None

@dataclass
class AuctionState:
    teams: Dict[str, Team]
    constraints: Constraints
    human_team: str
    sold_log: List[SaleResult] = field(default_factory=list)
    unsold_stack: List[Player] = field(default_factory=list)
    batch_size: int = 8

# -------------------------------
# Rules helpers
# -------------------------------
def base_tier_for_role(role: Role) -> int:
    mapping = {
        Role.A: 5,
        Role.W: 4,
        Role.B: 3,
        Role.P: 4,
        Role.S: 4,
    }
    return mapping.get(role, 3)

def apply_age_adjustments(base_tier: int, age: int) -> int:
    t = base_tier
    if age > 36:
        t -= 2
    elif 25 <= age <= 33:
        t += 1
    return max(1, min(9, t))

def tier_band(tier_id: int) -> Tuple[float,float]:
    for tid, lo, hi in TIERS:
        if tid == tier_id:
            return (lo, hi)
    raise ValueError("Invalid tier")

def snap_to_increment(amount: float) -> float:
    step = 0.25 if amount <= 9.0 else 0.5
    steps = int(amount / step + 1e-9)
    return round(steps * step, 2)

def assign_tier_and_reserve(player: Player, rng: random.Random) -> Player:
    if player.role is None or player.age is None:
        return player
    base = base_tier_for_role(player.role)
    final_tier = apply_age_adjustments(base, player.age)
    lo, hi = tier_band(final_tier)
    anchor = max(player.base_price, lo)
    reserve = rng.uniform(anchor, hi)
    reserve = snap_to_increment(reserve)
    player.tier_id = final_tier
    player.tier_lo = lo
    player.tier_hi = hi
    player.hidden_reserve = reserve
    return player