# Rummy Card Game — C++

A C++ implementation of the core data structures for a **Rummy-style card game**. The project models a standard 52-card deck with Unicode suit symbols, a singly-linked-list stack that acts as the draw/discard pile, and a Fisher-Yates-inspired shuffle algorithm.

---

## Table of Contents

1. [Game Rules (Rummy Overview)](#game-rules-rummy-overview)
2. [Project Structure](#project-structure)
3. [Architecture](#architecture)
4. [Classes & Functions](#classes--functions)
   - [Card](#card)
   - [stackItem](#stackitem)
   - [CashierStack](#cashierstack)
5. [Build Instructions](#build-instructions)
6. [Debug Mode](#debug-mode)
7. [Extending the Project](#extending-the-project)

---

## Game Rules (Rummy Overview)

Rummy is a group of matching-card games that share the same basic gameplay mechanic: forming **melds** (valid card combinations) from the cards in your hand.

### Objective
Be the first player to empty your hand by forming all cards into valid melds and laying them down.

### Card Values
| Card | Value |
|------|-------|
| Ace (A) | 1 |
| 2 – 10 | Face value |
| Jack (J) | 11 |
| Queen (Q) | 12 |
| King (K) | 13 |

### Suits
| Symbol | Name | Code |
|--------|------|------|
| ♠ | Spades | `S` |
| ♣ | Clubs | `C` |
| ♥ | Hearts | `H` |
| ♦ | Diamonds | `D` |

### Valid Melds
- **Set (Group):** Three or four cards of the **same rank** but different suits.  
  *Example: 7♠ 7♥ 7♦*
- **Run (Sequence):** Three or more cards of the **same suit** in consecutive order.  
  *Example: 4♣ 5♣ 6♣*

### Basic Turn Structure
1. **Draw** — Take the top card from the draw pile (deck) or the top card from the discard pile.
2. **Meld / Lay off** *(optional)* — Play a valid meld to the table, or add a card to an existing meld.
3. **Discard** — Place one card face-up on the discard pile to end your turn.

### Winning
A player wins when they have laid down all their cards in valid melds. Remaining cards in opponents' hands are counted as penalty points.

---

## Project Structure

```
Rummy---C-/
├── card.h            # Card class declaration
├── card.cpp          # Card class implementation
├── stackItem.h       # stackItem (linked-list node) declaration
├── stackItem.cpp     # stackItem implementation
├── cashierStack.h    # CashierStack (deck/pile) declaration
├── cashierStack.cpp  # CashierStack implementation
└── README.md         # This file
```

---

## Architecture

```
┌─────────────────────────────────┐
│          CashierStack           │  ← Draw pile / Discard pile
│  head ──► stackItem ──► stackItem ──► ... ──► nullptr
│           │               │
│           ▼               ▼
│          Card            Card
│        (suit, value)   (suit, value)
└─────────────────────────────────┘
```

- **`Card`** is the fundamental data unit — it stores a suit character and a numeric value.
- **`stackItem`** is a singly-linked-list node that wraps a `Card*` and a pointer to the next node.
- **`CashierStack`** is the deck/pile manager. It owns the linked list of `stackItem` nodes and exposes push, pop, and shuffle operations.

---

## Classes & Functions

### Card

Declared in `card.h`, implemented in `card.cpp`.

Represents a single playing card with a **suit** (`char`) and a **value** (`int`).

#### Private Members
| Member | Type | Description |
|--------|------|-------------|
| `sign` | `char` | Suit identifier: `'H'` (Hearts), `'S'` (Spades), `'C'` (Clubs), `'D'` (Diamonds) |
| `value` | `int` | Numeric value 1–13 (1 = Ace, 11 = Jack, 12 = Queen, 13 = King) |

#### Public Methods

---

##### `void setCard(char c, int v)`
Sets the card's suit and value.

| Parameter | Description |
|-----------|-------------|
| `c` | Suit character — one of `'H'`, `'S'`, `'C'`, `'D'` |
| `v` | Numeric value 1–13 |

```cpp
Card myCard;
myCard.setCard('H', 1);  // Ace of Hearts
```

---

##### `void printCard() const`
Prints the card to the console using a **Unicode suit symbol** followed by the card's label.

- Temporarily switches `stdout` to UTF-16 mode (`_O_U16TEXT`) to render the Unicode symbol, then switches back to text mode (`_O_TEXT`) to print the value.
- Values 11, 12, 13, and 1 are printed as `J`, `Q`, `K`, and `A` respectively; all other values print as their integer.

*Example output:* `♥ A`, `♠ 7`, `♦ Q`

---

##### `char getSign() const`
Returns the card's suit character (`'H'`, `'S'`, `'C'`, or `'D'`).

---

##### `int getValue() const`
Returns the card's numeric value (1–13).

---

##### `~Card()` *(DEBUG only)*
When compiled with `-DDEBUG`, the destructor prints `"Kill "` followed by the card's representation to help trace memory deallocation during debugging.

---

### stackItem

Declared in `stackItem.h`, implemented in `stackItem.cpp`.

A **singly-linked-list node** that wraps a pointer to a `Card` and a pointer to the next `stackItem`. It is used internally by `CashierStack` to build the deck linked list.

#### Private Members
| Member | Type | Description |
|--------|------|-------------|
| `card` | `Card*` | Pointer to the card stored in this node |
| `next` | `stackItem*` | Pointer to the next node in the list (`nullptr` at the tail) |

#### Public Methods

---

##### `stackItem(Card* c, stackItem* n = nullptr)`
Constructor. Creates a new node holding card `c` and linking to `n` as the next node.

| Parameter | Default | Description |
|-----------|---------|-------------|
| `c` | — | Pointer to the `Card` this node should hold |
| `n` | `nullptr` | Pointer to the next `stackItem` in the list |

---

##### `Card* getCard() const`
Returns the `Card*` stored in this node.

---

##### `stackItem* getNextItem() const`
Returns the pointer to the next `stackItem` node, or `nullptr` if this is the last node.

---

### CashierStack

Declared in `cashierStack.h`, implemented in `cashierStack.cpp`.

A **stack-based card deck** implemented as a singly-linked list. The top of the stack (`head`) represents the top card of the draw or discard pile. Supports standard stack operations and a shuffle routine.

#### Private Members
| Member | Type | Description |
|--------|------|-------------|
| `head` | `stackItem*` | Pointer to the top node of the stack |
| `numInDeck` | `int` | Running count of cards currently in the stack |

#### Public Methods

---

##### `CashierStack()`
Default constructor. Initialises an empty stack (`head = nullptr`, `numInDeck = 0`).

---

##### `~CashierStack()`
Destructor. Iterates through the linked list and `delete`s every `stackItem` node to prevent memory leaks. When compiled with `-DDEBUG`, it also prints `"Kill Stack"` to `stdout`.

---

##### `Card* pop()`
Removes and returns the top card from the stack.

1. Saves the current `head` node in a temporary pointer.
2. Advances `head` to the next node.
3. Deletes the old `head` node (the `stackItem` wrapper — **not** the `Card` itself).
4. Decrements `numInDeck`.
5. Returns the `Card*` that was on top, or `nullptr` if the stack was empty.

```cpp
Card* drawnCard = deck.pop();
if (drawnCard) {
    drawnCard->printCard();
}
```

---

##### `void push(Card* c)`
Pushes a card onto the top of the stack.

1. Allocates a new `stackItem` wrapping `c` and pointing to the current `head`.
2. Sets `head` to the new node.
3. Increments `numInDeck`.

```cpp
Card myCard;
myCard.setCard('D', 5);  // 5 of Diamonds
deck.push(&myCard);
```

---

##### `void shuffle()`
Shuffles the entire deck using repeated random repositioning.

- Seeds the random number generator with the current time (`std::time(nullptr)`).
- Runs **200 iterations**, each time calling `shuffleNitem(n)` with a random index `n` in `[0, numInDeck)`.
- This simulates multiple random cuts of the deck, producing a well-randomised order.

---

##### `void shuffleNitem(int n)`
Moves the card at position `n` (counting from the top) to the **top** of the stack.

**How it works:**
1. Pops the top `n-1` cards into a temporary `CashierStack` (`tmp`).
2. Pops the card now at the top (position `n`) and saves it as `c`.
3. Pops all cards from `tmp` back onto the main stack (restoring the original order of the first `n-1` cards).
4. Pushes `c` onto the top of the main stack.

The net effect is that the card originally at depth `n` is now at the top of the deck, providing a random-cut shuffle step.

---

##### `void printStackForDeBugOnly() const`
Iterates through the entire stack from `head` to the tail and prints each card's index and representation. Intended for development and debugging only — do not use in production builds.

---

## Build Instructions

This project requires a C++ compiler that supports C++11 or later.

> **Note:** The repository contains only the card/deck library code. There is no `main()` function, so the commands below will fail at the linking stage until you add your own `main.cpp` (or a test driver). The commands illustrate how to compile the provided source files into your project.

### GCC / MinGW (Linux / Windows)

```sh
g++ main.cpp card.cpp stackItem.cpp cashierStack.cpp -o rummy
```

### Clang

```sh
clang++ main.cpp card.cpp stackItem.cpp cashierStack.cpp -o rummy
```

### MSVC (Visual Studio Developer Command Prompt)

```sh
cl main.cpp card.cpp stackItem.cpp cashierStack.cpp /Fe:rummy.exe
```

> **Windows note:** `card.cpp` uses `_setmode`, `_fileno`, `_O_U16TEXT`, and `_O_TEXT` from `<io.h>` and `<fcntl.h>` for Unicode console output. These are Windows-specific headers. On Linux/macOS you can replace the `_setmode` calls with `wcout.imbue(locale(""))` or simply use `cout` with UTF-8 output.

---

## Debug Mode

Compile with `-DDEBUG` to enable extra diagnostic output:

```sh
g++ card.cpp stackItem.cpp cashierStack.cpp -DDEBUG -o rummy_debug
```

In debug mode:
- `Card::~Card()` prints `"Kill <card>"` when a `Card` object is destroyed.
- `CashierStack::~CashierStack()` prints `"Kill Stack"` when the deck is destroyed.

---

## Extending the Project

The current codebase provides the fundamental building blocks. A full Rummy game loop can be built on top of it by adding:

- **Player class** — hand of cards (another stack or vector of `Card*`), score tracking.
- **Game class** — manages the draw pile, discard pile, player turns, and win detection.
- **Meld validation** — functions to check whether a group of cards forms a valid set or run.
- **UI layer** — text-based or graphical front-end that calls `printCard()` for display.
