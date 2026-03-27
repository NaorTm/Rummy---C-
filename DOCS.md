# Technical Documentation — Rummy C++

This document provides in-depth technical documentation for every class, data member, and method in the Rummy C++ project. For a high-level overview and game rules, see [README.md](README.md).

---

## Table of Contents

1. [Design Overview](#design-overview)
2. [Class: Card](#class-card)
3. [Class: stackItem](#class-stackitem)
4. [Class: CashierStack](#class-cashierstack)
5. [Memory Management](#memory-management)
6. [Unicode Console Output](#unicode-console-output)
7. [Shuffle Algorithm](#shuffle-algorithm)

---

## Design Overview

The project is split into three tightly coupled classes:

```
Card  ←──────────────────────────────────────────────────┐
  ▲                                                       │
  │ owns (raw pointer)                                    │
stackItem  ──next──►  stackItem  ──next──►  ...  nullptr  │
  ▲                                                       │
  │ head                                                  │
CashierStack   (numInDeck: int)  ──────────────────────────┘
```

| Class | File | Responsibility |
|-------|------|----------------|
| `Card` | `card.h` / `card.cpp` | Stores a single card's suit and value; handles rendering |
| `stackItem` | `stackItem.h` / `stackItem.cpp` | Linked-list node wrapping a `Card*` |
| `CashierStack` | `cashierStack.h` / `cashierStack.cpp` | Stack-based deck with push, pop, shuffle |

---

## Class: Card

**Header:** `card.h`  
**Implementation:** `card.cpp`

### Purpose
Represents a single playing card. A card has two attributes:
- A **suit** (`char`): one of `'H'` (Hearts), `'S'` (Spades), `'C'` (Clubs), `'D'` (Diamonds).
- A **value** (`int`): integer 1–13, where 1 = Ace, 11 = Jack, 12 = Queen, 13 = King.

### Preprocessor Macros (card.h)

```cpp
#define SPADE   L"\u2660"   // ♠
#define CLUB    L"\u2663"   // ♣
#define HEART   L"\u2665"   // ♥
#define DIAMOND L"\u2666"   // ♦
```

These wide-character string literals are used in `printCard()` to render Unicode suit symbols in the Windows console.

### Data Members

| Member | Access | Type | Description |
|--------|--------|------|-------------|
| `sign` | private | `char` | Suit identifier character |
| `value` | private | `int` | Numeric card value (1–13) |

### Method Reference

#### `void setCard(char c, int v)`

**Purpose:** Initialises the card's suit and value.

**Parameters:**
- `c` — Suit character. Valid values: `'H'`, `'S'`, `'C'`, `'D'`.
- `v` — Card value. Valid range: 1–13.

**Side effects:** Directly assigns `sign = c` and `value = v`. No validation is performed; callers are responsible for passing valid values.

**Example:**
```cpp
Card c;
c.setCard('S', 13);  // King of Spades
```

---

#### `void printCard() const`

**Purpose:** Outputs the card's Unicode suit symbol followed by its label to the console.

**How it works (step by step):**
1. Calls `_setmode(_fileno(stdout), _O_U16TEXT)` to put the standard output stream into **UTF-16 wide-character mode** so that `wcout` can emit Unicode characters correctly on Windows.
2. Uses a `switch` on `sign` to select the matching wide-character macro (`HEART`, `SPADE`, `CLUB`, or `DIAMOND`) and outputs it with `wcout`.
3. Calls `_setmode(_fileno(stdout), _O_TEXT)` to restore **narrow text mode** so that subsequent `cout` calls work normally.
4. Uses a second `switch` on `value` to print the display label:
   - `1` → `" A"` (Ace)
   - `11` → `" J"` (Jack)
   - `12` → `" Q"` (Queen)
   - `13` → `" K"` (King)
   - all others → `" "` + integer value

**Output examples:**
```
♥ A   (Ace of Hearts)
♠ 7   (7 of Spades)
♦ Q   (Queen of Diamonds)
♣ 10  (10 of Clubs)
```

**Platform note:** `_setmode`, `_fileno`, `_O_U16TEXT`, and `_O_TEXT` are Windows-specific (from `<io.h>` and `<fcntl.h>`). Porting to Linux/macOS requires replacing this mechanism with a locale-aware alternative.

---

#### `char getSign() const`

**Purpose:** Accessor for the `sign` member.

**Returns:** The suit character (`'H'`, `'S'`, `'C'`, or `'D'`).

---

#### `int getValue() const`

**Purpose:** Accessor for the `value` member.

**Returns:** Integer in the range 1–13.

---

#### `~Card()` *(conditional, DEBUG only)*

**Compiled when:** `DEBUG` macro is defined (`-DDEBUG`).

**Purpose:** Destructor used during debug sessions to trace object lifetime. Prints `"\n Kill "` followed by the card's visual representation by calling `printCard()`.

**Example output:**
```
 Kill ♣ 5
```

---

## Class: stackItem

**Header:** `stackItem.h`  
**Implementation:** `stackItem.cpp`

### Purpose
A **singly-linked-list node** that composes the internal storage of `CashierStack`. Each node holds a raw pointer to a `Card` and a pointer to the next node in the list.

### Data Members

| Member | Access | Type | Description |
|--------|--------|------|-------------|
| `card` | private | `Card*` | Non-owning pointer to a card |
| `next` | private | `stackItem*` | Pointer to the next node; `nullptr` at the tail |

> **Ownership note:** `stackItem` does **not** own the `Card` it points to. Memory management of `Card` objects is the caller's responsibility. `stackItem` owns only itself.

### Method Reference

#### `stackItem(Card* c, stackItem* n = nullptr)`

**Purpose:** Constructor. Creates a new linked-list node.

**Parameters:**
- `c` — Pointer to the `Card` this node holds. Must not be `nullptr` in normal usage.
- `n` — Pointer to the next `stackItem`. Defaults to `nullptr`, making this node a tail node.

**Side effects:** Assigns `card = c` and `next = n`.

**Typical usage inside `CashierStack::push`:**
```cpp
stackItem* newItem = new stackItem(c, head);  // link new node to old head
head = newItem;                                // update head
```

---

#### `Card* getCard() const`

**Purpose:** Returns the card pointer stored in this node.

**Returns:** `Card*` — the pointer set at construction time.

---

#### `stackItem* getNextItem() const`

**Purpose:** Returns the pointer to the next node in the linked list.

**Returns:** `stackItem*` — the next node, or `nullptr` if this is the last node.

---

## Class: CashierStack

**Header:** `cashierStack.h`  
**Implementation:** `cashierStack.cpp`

### Purpose
Implements a **LIFO stack** (last-in, first-out) backed by a singly-linked list of `stackItem` nodes. This models the draw pile or discard pile in a card game. The top of the stack corresponds to `head`.

### Data Members

| Member | Access | Type | Description |
|--------|--------|------|-------------|
| `head` | private | `stackItem*` | Pointer to the top node; `nullptr` for an empty stack |
| `numInDeck` | private | `int` | Number of cards currently in the stack |

### Method Reference

#### `CashierStack()`

**Purpose:** Default constructor. Initialises an empty stack.

**Post-conditions:**
- `head == nullptr`
- `numInDeck == 0`

---

#### `~CashierStack()`

**Purpose:** Destructor. Frees all `stackItem` nodes allocated on the heap to prevent memory leaks.

**Algorithm:**
```
while head is not nullptr:
    save head in tmp
    advance head to head->getNextItem()
    delete tmp
```

Note that only the `stackItem` wrappers are deleted here, not the `Card` objects they point to, because `stackItem` is non-owning with respect to `Card`.

**DEBUG behaviour:** When compiled with `-DDEBUG`, prints `"\n Kill Stack"` before the loop runs.

---

#### `Card* pop()`

**Purpose:** Removes and returns the top card from the stack.

**Returns:** `Card*` pointing to the top card, or `nullptr` if the stack is empty.

**Algorithm:**
1. `tmp = head` (save current head)
2. If `head == nullptr`, return `nullptr` (empty stack guard).
3. `res = head->getCard()` (save the card pointer)
4. `head = head->getNextItem()` (advance head)
5. `delete tmp` (free the stackItem node)
6. `numInDeck--`
7. Return `res`

**Complexity:** O(1)

**Example:**
```cpp
Card* top = deck.pop();
if (top != nullptr) {
    top->printCard();
}
```

---

#### `void push(Card* c)`

**Purpose:** Pushes a card onto the top of the stack.

**Parameters:**
- `c` — Pointer to the card to push.

**Algorithm:**
1. Allocate `new stackItem(c, head)` — the new node points to the current top.
2. Set `head` to the new node.
3. `numInDeck++`

**Complexity:** O(1)

**Example:**
```cpp
Card card;
card.setCard('H', 7);  // 7 of Hearts
deck.push(&card);
```

---

#### `void shuffle()`

**Purpose:** Randomly reorders all cards in the stack.

**How it works:**
1. Seeds the C standard library RNG with the current Unix timestamp: `srand(std::time(nullptr))`.
2. Iterates **200 times**. Each iteration:
   a. Generates a random integer `random_variable = std::rand()` (used implicitly to advance RNG state).
   b. Picks a random depth `n = rand() % numInDeck`.
   c. Calls `shuffleNitem(n)` to move the card at depth `n` to the top of the stack.

Each `shuffleNitem` call performs a deterministic cut of the deck, and repeating this 200 times with random depths produces a well-mixed ordering.

**Side effects:** Mutates the entire linked list ordering.

**Performance:** O(200 × n) = O(n) for a 52-card deck (constant factor).

---

#### `void shuffleNitem(int n)`

**Purpose:** Moves the card at depth `n` (counted from the top) to the top of the stack, leaving all other cards in their original relative order.

**Parameters:**
- `n` — Zero-based depth of the card to move (0 = already at top, 1 = second from top, etc.).

**Algorithm (step by step):**
```
1. Create a temporary CashierStack `tmp`.
2. While n > 1 AND head is not nullptr:
       tmp.push(pop())    ← pop top card onto tmp
       n--
   Result: the first (original_n - 1) cards are now on `tmp`, reversed.
3. c = pop()              ← this is the card originally at depth n
4. While tmp.head is not nullptr:
       push(tmp.pop())    ← restore the cards back (reversing them again = original order)
5. push(c)                ← place the target card on top
```

**Net effect:** The card that was at depth `n` is now at the top of the stack. The relative order of all other cards is preserved.

**Complexity:** O(n) — proportional to the depth of the card being moved.

**Example walkthrough** (deck top → bottom: A B C D E, move card at depth 2):
```
Initial:  A B C D E
Step 2:   Pop A → tmp, Pop B → tmp  (tmp top→bottom: B A)
Step 3:   Pop C → c
Stack:    D E
Step 4:   Push B from tmp, Push A from tmp
Stack:    A B D E
Step 5:   Push C
Stack:    C A B D E
```
Card C (originally at depth 2) is now at the top.

---

#### `void printStackForDeBugOnly() const`

**Purpose:** Diagnostic utility. Prints the index and card representation of every card in the stack, from top to bottom.

**Output format:**
```
---------------
 no 0 - ♥ 7
 no 1 - ♠ K
 no 2 - ♦ 3
 ...
```

**Usage:** Call this after `shuffle()` or `push()` operations during development to verify deck state. Should not be called in production builds.

---

## Memory Management

| Object | Allocated by | Freed by |
|--------|-------------|----------|
| `Card` | Caller | Caller |
| `stackItem` | `CashierStack::push()` | `CashierStack::pop()` or `CashierStack::~CashierStack()` |
| `CashierStack` (internal tmp) | `shuffleNitem()` on the stack | `shuffleNitem()` destructor (automatic) |

Key points:
- `CashierStack` **owns** `stackItem` nodes but **does not own** `Card` objects.
- When `pop()` is called, the `stackItem` node is deleted but the returned `Card*` must be freed by the caller.
- The temporary `CashierStack` used inside `shuffleNitem()` is a stack-allocated local variable; its destructor automatically cleans up any remaining nodes when the function returns.

---

## Unicode Console Output

`printCard()` requires the Windows console to be in **UTF-16 mode** to display suit symbols correctly:

```cpp
_setmode(_fileno(stdout), _O_U16TEXT);   // switch to wide-char mode
wcout << HEART;                           // output Unicode symbol
_setmode(_fileno(stdout), _O_TEXT);       // restore narrow mode
cout << " A";                             // output value label
```

Mixing `cout` (narrow) and `wcout` (wide) on the same stream without a mode switch can cause undefined behaviour on Windows, so the mode must be toggled around each Unicode output.

**Cross-platform alternative (Linux/macOS):**
```cpp
// Replace _setmode calls with locale setup at program start:
setlocale(LC_ALL, "");
// Then simply use wcout throughout, or use UTF-8 with cout:
cout << "\u2665";   // UTF-8 heart symbol
```

---

## Shuffle Algorithm

The shuffle in `CashierStack::shuffle()` is a **random-cut shuffle**: repeatedly move a randomly-selected card to the top. After 200 iterations each drawing a uniform random depth, the expected distribution approaches uniform permutation for a 52-card deck.

Comparison with Fisher-Yates:

| Property | This implementation | Fisher-Yates |
|----------|-------------------|--------------|
| Complexity | O(200 × n) | O(n) |
| Result quality | Good after many iterations | Provably uniform in one pass |
| Seed | `std::time()` (second resolution) | Same |

For a production-quality shuffle, the most common alternative is **Fisher-Yates**. Because this project uses a singly-linked list (which has no O(1) random access), applying Fisher-Yates directly is not practical. You would first need to convert the deck to a `std::vector<Card*>`, shuffle in place, and rebuild the linked list:

```cpp
// Example: Fisher-Yates on a vector representation
#include <vector>
#include <algorithm>
#include <random>

std::vector<Card*> cards;
// ... populate cards ...
std::mt19937 rng(std::random_device{}());
std::shuffle(cards.begin(), cards.end(), rng);
// ... rebuild CashierStack from the shuffled vector ...
```
