# Ideas for an Expert Automated Player (Self-Play) for Club Fight

## 1. Game Analysis & The Agent's Perspective

"Club Fight" is a single-player stochastic puzzle game with hidden information. To build an expert-level automated player, we must formally define the environment:

*   **State Space:**
    *   **Visible State:** The 3x3 Grid (up to 9 cards), the Club Hand (up to 4 cards).
    *   **Hidden State:** The order of the remaining cards in the Draw Pile (non-clubs + jokers) and the Club Pile.
    *   **Agent's Memory (Card Counting):** The agent will maintain an exact distribution of the remaining unseen cards. Since it knows the starting deck (54 cards), it can perfectly deduce the exact quantities of each rank and suit remaining in both the Draw Pile and the Club Pile.
*   **Action Space:**
    *   A single turn consists of picking a `Club Card` from the hand, choosing an `Action` (Match, Sum, Any, Run, Up/Down, Line), and selecting `Target Cards` from the grid.
    *   The action space is highly combinatorial but heavily constrained by the grid's current state and suit blockers (e.g., 3+ Hearts block "Run").
*   **Reward Function:**
    *   +1 for clearing a grid card (dense reward).
    *   +100 for winning (clearing the entire draw pile and grid).
    *   -100 for losing (running out of clubs with grid cards remaining).

---

## 2. Approach 1: Perfect Information Monte Carlo (PIMC) / MCTS

Given the stochastic nature of drawing cards, standard minimax search doesn't work. Monte Carlo Tree Search (MCTS) is a premier algorithm for this, specifically a variant designed for hidden information.

### How it works:
1.  **Determinization:** Before making a move, the agent randomly shuffles the known remaining cards in the Draw Pile and Club Pile to create a "determinized" (fully observable) simulated game state.
2.  **MCTS Execution:** The agent runs standard MCTS on this simulated perfect-information state. It explores the tree of possible actions, simulating the game forward.
3.  **Ensemble Voting:** The agent repeats this process for dozens or hundreds of different determinizations.
4.  **Action Selection:** The agent aggregates the results across all determinized MCTS runs and selects the action with the highest expected win rate or expected cards cleared.

### Why it fits Club Fight:
*   **Perfect Memory Utilization:** Determinization leverages the agent's perfect memory. As the game progresses and the deck thins, the determinizations become increasingly accurate, allowing the agent to calculate exact probabilities for the endgame.
*   **Tactical Depth:** MCTS naturally discovers deep tactical sacrifices, such as using an "Any" action on a Heart just to unblock the highly efficient "Run" action for the next turn.

---

## 3. Approach 2: Deep Reinforcement Learning (AlphaZero Architecture)

A dedicated Neural Network trained via self-play Reinforcement Learning (RL) can learn to evaluate the board and propose actions without relying entirely on brute-force search.

### State Representation (Input Tensor):
The neural network needs a structured input representing the game state:
*   **Grid:** A 3x3x54 one-hot encoded tensor representing the exact card in each grid slot.
*   **Hand:** A 4x13 one-hot tensor for the available clubs.
*   **Memory Vector:** A 54-dimensional vector containing the normalized counts of remaining cards in the deck, providing the network with perfect card-counting memory.

### Network Outputs:
*   **Policy Head ($p$):** Outputs a probability distribution over all legal composite actions (Club -> Action Type -> Targets).
*   **Value Head ($v$):** Outputs a scalar between [-1, 1] predicting the probability of winning from the current state.

### Training via PPO or AlphaZero:
*   The network plays millions of games against itself.
*   It can be combined with MCTS (like AlphaZero). The Neural Network evaluates leaf nodes in the MCTS tree, drastically reducing the need for random rollouts and focusing the search only on highly promising moves.

### Why it fits Club Fight:
*   **Pattern Recognition:** The network will implicitly learn heuristics humans struggle with, such as the statistical value of keeping a low-value club vs. a face card based on the remaining draw pile distribution.
*   **Fast Inference:** Once trained, neural network inference is much faster than running thousands of PIMC simulations per turn, making it ideal for a real-time web opponent.

---

## 4. Approach 3: Expectimax Search with a Crafted Heuristic

If training a neural network is too complex, a classical Expectimax algorithm augmented by a hand-crafted evaluation function is a strong baseline.

### The Heuristic Function:
Instead of searching to the end of the game, the search tree is truncated at a shallow depth (e.g., 2-3 turns). The leaf nodes are evaluated by a formula that scores the "health" of the board:
*   $+$ `Weight_A` * (Number of Club Cards Remaining)
*   $-$ `Weight_B` * (Number of Grid Cards Remaining)
*   $-$ `Weight_C` * (Penalty for blocked actions: +1 if Hearts $\ge$ 3, etc.)
*   $+$ `Weight_D` * (Grid flexibility: high variance in grid ranks is better for "Sum" and "Run").

### Expectimax Implementation:
The search tree branches on two node types:
1.  **Max Nodes (Agent's Turn):** The agent picks the action that maximizes the expected heuristic score.
2.  **Chance Nodes (Drawing Cards):** The environment transitions are averaged based on the exact probabilities of drawing specific cards, calculated directly from the agent's perfect memory of the remaining deck.

---

## 5. Strategic Concepts the Agent Will Exploit

Regardless of the underlying architecture, a strong AI will inherently discover and exploit these high-level strategies:

1.  **Suit Management (Unblocking):** The AI will track the suits in the grid carefully. If Spades are at 2, and the Draw Pile is heavy on Spades, the AI will prioritize removing a Spade with a "Match" or "Any" action to prevent the "Up/Down" action from being blocked on the next draw.
2.  **Joker Hoarding:** Jokers are incredibly powerful for the "Line" action. The AI will calculate the probability of drawing a Joker and save specific club ranks to maximize a future "Line" clear.
3.  **Expected Value of "Sum":** The AI will know exactly how many low cards (A-5) vs high cards (9-K) remain. If the deck is heavy on low cards, it will hold onto high-value clubs (K, Q, J) to clear 3-4 grid cards simultaneously via the "Sum" action.

## Summary Recommendation

For the strongest possible expert player, an **AlphaZero-style architecture (MCTS guided by a Deep Neural Network)** is the most robust choice. It combines the rigorous tactical foresight of tree search with the intuitive, probability-aware pattern recognition of a neural network.

If computational resources for training are limited, **Perfect Information Monte Carlo (PIMC)** is the best purely algorithmic alternative, as it perfectly leverages the agent's card-counting memory without requiring an explicitly hand-crafted heuristic.