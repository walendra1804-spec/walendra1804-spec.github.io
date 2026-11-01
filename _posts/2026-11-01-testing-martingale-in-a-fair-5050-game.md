---
title: "Testing Martingale in a Fair 50/50 Game: A High Win Rate With an Expected Value of Zero"
description: "A small terminal program for learning probability, extended with the martingale betting strategy. It shows how to split a bankroll across n doubling bets, why every win pays the same profit, and why no betting system can make the expected value of a fair game positive."
date: 2026-11-01 14:00:00 +0700
tags: [probability, martingale, expected-value, python, gambling, math, experiment]
---

*Setup: a small Python program that runs in the terminal, written to learn probability by playing with it. All money in it is imaginary, and this is not financial advice.*

## TL;DR

| Question | Answer |
|---|---|
| Chance of at least one win in n rounds | 1 − (1/2)^n |
| How to fit a bankroll into n doubling bets | First bet = bankroll / (2^n − 1) |
| Profit when you win on round k | Always the first bet, whichever round it is |
| Loss when all n rounds lose | The whole bankroll |
| Expected value of a game | Exactly 0, for any n |
| Can any betting system beat a fair 50/50 game? | No. It changes how the results are spread out, not their average |

---

## 1. Where the idea came from

I wanted to understand probability by experimenting, not by reading formulas. So I started with the simplest game I could describe.

One game has up to n rounds, and I choose n before it starts. Each round comes up **B** or **S**, 50% each, and only appears after I press Enter. B ends the game immediately. S moves on to the next round. If all n rounds are S, the game ends with no B at all.

To make it more interesting, I then added martingale, the strategy of doubling your bet after every loss. I knew what it should look like: with a bankroll of 100,000 and n = 5, the five doubling bets should use up exactly the 100,000. What I didn't know was the formula for the bets, or how much a win should pay. My guess was that the profit would depend on which round the win came in. That guess turned out to be wrong.

## 2. The math behind it

**The chance of a win.** The only way to get no B is to get S every time, which has a chance of (1/2)^n. Everything else contains a B:

```text
P(at least one B) = 1 - (1/2)^n
P(first B on round k) = (1/2)^k
```

**The size of the bets.** The bets go b1, 2·b1, 4·b1, and so on. Their total is b1 × (2^n − 1), and that total has to equal the bankroll:

```text
b1 = bankroll / (2^n - 1)
```

With 100,000 and n = 5, b1 = 100,000 / 31 = 3,225.81:

| Round k | Chance the first B lands here | Bet | Total staked so far |
|---|---:|---:|---:|
| 1 | 50% | 3,225.81 | 3,225.81 |
| 2 | 25% | 6,451.61 | 9,677.42 |
| 3 | 12.5% | 12,903.23 | 22,580.65 |
| 4 | 6.25% | 25,806.45 | 48,387.10 |
| 5 | 3.125% | 51,612.90 | 100,000.00 |

**The profit.** We assumed an even-money payout, so a winning bet of x earns x. A win on round k then earns b1 × 2^(k−1), while the earlier losses add up to b1 × (2^(k−1) − 1). The difference is always exactly b1.

A win on round 4, for example, earns 25,806.45 after losing 22,580.65, which leaves 3,225.81 (the cents are rounded). That's the same as winning on round 1. The doubling is there to cancel earlier losses, not to grow the profit.

**The expected value.** Multiply each outcome by its chance and add them up:

```text
EV = (31/32 × 3,225.81) - (1/32 × 100,000) = 0
```

## 3. How we tested it

The program has two balances, because they answer different questions:

- **The stake** is what's at risk in one game. It stays the same every game, so I can keep staking 100,000.
- **The all-time balance** starts at the bankroll and changes by each game's profit or loss.

The core of a game is short:

```python
def play(n, stake):
    b1 = stake / (2 ** n - 1)
    lost = 0.0
    for k in range(1, n + 1):
        bet = b1 * 2 ** (k - 1)
        input(f"[Round {k}/{n}] bet {bet:,.2f} -> press Enter...")
        if random.random() < 0.5:   # B
            return bet - lost       # always equals b1
        lost += bet                 # S
    return -lost                    # always equals -stake
```

We tested it in two ways:

**1. Running the program.** We fed it scripted Enter presses, with a 100,000 stake and n = 5. It finished five games, winning on rounds 1, 2, 2, 1 and 2. Every win paid +3,225.81, and the all-time balance rose to 116,129.03. The input ran out during the sixth game.

**2. Exact calculation.** We went through every outcome using exact fractions, not random trials, for several values of n. A win always paid exactly b1, and a full losing streak always cost exactly the stake:

| n | First bet | Chance of winning | Chance of losing it all | Wins needed to recover one loss | EV |
|---|---:|---:|---:|---:|---:|
| 1 | 100,000.00 | 50% | 50% | 1 | 0 |
| 3 | 14,285.71 | 87.5% | 12.5% | 7 | 0 |
| 5 | 3,225.81 | 96.875% | 3.125% | 31 | 0 |
| 10 | 97.75 | 99.90234% | 0.09766% | 1,023 | 0 |
| 20 | 0.10 | 99.9999% | 0.0001% | 1,048,575 | 0 |

> **Honest note:** the five played games only show that the program follows its rules. Five games are far too few to show anything about the strategy, and five straight wins is an outcome you'd expect about 85% of the time at n = 5. The conclusion comes from the exact calculation. We did not run a large simulation.

## 4. What the results show

**Five wins in a row looks like a working strategy.** That is the trap. The balance climbed steadily and nothing had gone wrong yet, but the risk hadn't gone away. It just hadn't shown up.

Staying with n = 5, the chance of at least one total loss grows quickly as you keep playing:

| Games played | Chance of at least one total loss |
|---:|---:|
| 5 | 14.7% |
| 22 | 50.3% |
| 32 | 63.8% |
| 100 | 95.8% |

One total loss wipes out the profit from 31 wins.

**A bigger n doesn't help.** Going from n = 5 to n = 20 makes a total loss very rare, but it shrinks each win just as much. The first bet falls to about 0.10, and recovering a single loss takes more than a million wins. The EV stays at 0 in every row.

## 5. Can any strategy beat a fair game?

No. Each round has an EV of 0: a bet of x gains x half the time and loses x half the time. A betting system only chooses how much to bet and when to stop, and it can only base those choices on what has already happened. Past rounds say nothing about the next one, so every bet still has an EV of 0, and so does their total. With a finite bankroll, this is what the [optional stopping theorem](https://en.wikipedia.org/wiki/Optional_stopping_theorem) formalizes.

What a strategy can change is the shape of the results:

| Strategy | Usual result | Rare result | EV |
|---|---|---|---|
| Martingale | Small win | Large loss | 0 |
| Doubling after wins | Small loss | Large win | 0 |
| Same bet every round | Slow drift up and down | Larger drift | 0 |

The EV can only go above 0 if one of the game's assumptions changes:

- **Your chance of winning is really above 50%**, because the game is biased or you know something it doesn't account for.
- **The payout is better than fair**, such as winning 1.1 for every 1 risked at 50% odds.
- **Your bankroll and time are unlimited.** Martingale only "always wins" on paper. Real bankrolls run out, and that limit is the n in this program.

Real casino games go the other way. A red bet on European roulette wins 18 times out of 37. With the same 100,000 stake and n = 5, the EV falls from 0 to about **−460 per game**. The win rate still looks great at 96.4%, but no choice of bets can bring the EV back up to 0.

## 6. What I took from this

1. **A high win rate says little on its own.** You also need to know how much you make when you win and how much you lose when you don't. 96.875% wins can still be worth exactly nothing.
2. **A good run doesn't prove a strategy works.** A strategy that wins small and often will look right for most of its life, right up until the rare loss.
3. **Lowering the chance of a disaster isn't the same as removing its cost.** Martingale makes the big loss rarer and the wins smaller, and the two cancel out exactly.
4. **Check your guesses with the math.** I expected the profit to depend on the winning round. A few lines of algebra showed that it never does.
5. **Look for a real edge, not a clever pattern.** When the odds and payouts are fair, bet sizing can't create an advantage.

---

*Written by Panca Walendra Putra and Claude Opus 5.5 (Anthropic), from a real experiment session.*
