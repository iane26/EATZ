<<<<<<< HEAD
# EATZ
=======
# EATZ

A terminal pager for food trucks. The cook types each order in, and the program prints a
simulated text for the customer with their exact place in line, for example
`[TEXT to 555-555-0142] You're #4, about 12 min away.`

Language: Java (tested with JDK 25; anything 17+ works). No build tool: just `javac`.

## Layout

```
src/main/java/eatz/model/       MODEL       LinkedDeque (Queue + Node), Order, OrderStatus,
                                            OrderInfo, OrderValidator, OrderLine
src/main/java/eatz/view/        VIEW        ConsoleView
src/main/java/eatz/controller/  CONTROLLER  EatzController, EatzApp (main)
src/main/java/eatz/ModelDriver.java         tiny driver: model only, no view
src/test/java/eatz/model/       JUnit tests for the model (no view, no controller)
src/test/java/eatz/controller/  scripted whole-program test (fake keyboard)
lib/                            JUnit console launcher jar (so no Maven/Gradle needed)
build.ps1                       compile / test / run helper
```

## How to run (PowerShell, from this folder)

| Command | What it does |
|---|---|
| `.\build.ps1 driver` | **Model only.** Runs `ModelDriver`: adds, cancels, serves and remakes orders and shows rejected input. No view or controller is used. |
| `.\build.ps1 test` | Compiles and runs all JUnit tests (68 tests; all pass). |
| `.\build.ps1 run` | **Thin view.** Runs the real program: `EatzApp` -> `EatzController` -> `ConsoleView` -> `OrderLine`. |

Without the script (what it does under the hood):

```
javac -d out/main (all files under src/main)
java -cp out/main eatz.ModelDriver
java -cp out/main eatz.controller.EatzApp
```

Using the program: pick `1` Add, `2` Mark ready, `3` Cancel, `4` View line, `5` Remake.
Items are typed on one line like `2 taco, churro` (quantity optional). Menu: taco, burrito,
quesadilla, churro, horchata. End of input quits with "Goodbye." (Ctrl+D on Mac/Linux, Ctrl+Z
then Enter on Windows). Ctrl+C also prints "Goodbye.".

## Design summary

- **Model** (`eatz.model`) imports only `java.*` and itself. It never prints or reads.
  `OrderLine` owns the Queue (`LinkedDeque<Order> waiting`) plus two hash maps; `nodeById`
  points at queue nodes so Cancel unlinks in O(1). `OrderInfo` is the read-only snapshot the
  model hands out.
- **View** (`ConsoleView`) knows only `OrderInfo` and strings. It makes no decisions.
- **Controller** (`EatzController`) is the only class touching both sides. It sends typed text
  to `OrderValidator`, calls `OrderLine`, passes the answers to the view, and turns
  `IllegalArgumentException` / `IllegalStateException` into messages.

## Reflection

**What changed from our specification?**

| Area | Spec said | We built | Why |
|---|---|---|---|
| Classes | Class table listed 9 classes | Added `OrderStatus` (enum) and `ModelDriver` (tiny driver) | The spec used `OrderStatus` in Section 4 but never gave it a row; `ModelDriver` satisfies "exercisable from a tiny driver" without a view. |
| ConsoleView methods | Spec named only `readLine`, `confirm`, `showError`, `showPositionText` | Also `showMenu`, `showMessage`, `showLine`, `showReadyText`, `showCancelText` | The spec said the view prints menus, the line and texts but never named those methods. Mark ready and Cancel each need a customer text. |
| `Order` constructor | Re-checks id, name, phone, items, quantity >= 1, nulls | Also enforces the caps (10 per item, 20 per order) | Section 6 says caps "must hold even if a caller skips the controller". `OrderLine.addOrder` gets items as a Map, not text, so it cannot call `parseItems`; the `Order` constructor is the backstop. |
| `parseItems` details | Listed the four error messages | Also: merged repeats are checked against the per-item cap (`10 taco, 5 taco` -> quantity error); empty entries like `taco,,churro` are ignored; `2x taco` -> quantity error; an entry that does not start with a digit or sign is treated as a whole item name (so `veggie burrito` says "isn't on the menu" instead of a confusing quantity error) | The spec left these cases open. |
| `cleanName` | Strip control characters, collapse spaces, trim, cut to 30 | Same, but if the cut lands in the middle of an emoji (surrogate pair) the half is dropped | Avoids storing a broken character. |
| `OrderInfo` from `markReady` / `cancel` | "position 0" | Position 0 **and** wait 0 | Wait is position x 3, so 0 follows from the spec's rule. |
| `EatzApp` Ctrl+C hook | "registers a Ctrl+C hook that prints Goodbye." | The hook only prints if `run()` did not already finish | A JVM shutdown hook runs on every exit, so on Ctrl+D we would have printed "Goodbye." twice. |
| Tests | Section 7 listed model tests only, no test creates a controller or view | All Section 7 model tests are written, plus a few extras (phone cap freeing up after a serve, remake not blocked by the cap, rejected add does not use an ID, remade order can be cancelled). We also added `EatzControllerTest`, a scripted-keyboard run | Section 8 asks for a scripted run (bad menu choice, bad phone, Mark ready on empty, end of input). It is a separate file in the controller package, so the model tests still never touch a view or controller. |
| Validation location | Model decides, controller catches, view prints | Unchanged | The design held up. |
| Method signatures | Frozen in Sections 4-5 | No public signature changed; only additions listed above | Frozen contract worked. |

TODO (team): add the "what prompted it" detail for any change you remember from the
Wednesday join discussion.

**Did the model really stand alone?**
Yes. Every file in `eatz.model` imports only `java.*` and other `eatz.model` classes (checked
with a search of the imports). The model is exercised by `ModelDriver` and by the JUnit
tests with no `Scanner`, `PrintStream`, `ConsoleView` or `EatzController`. Nothing had to move across
the line. The one judgement call: the duplicate-order question. The *model* decides what a
duplicate is (`isDuplicateOfLast`), and the *controller + view* ask the cook (`confirm`), as
the spec said.

**What happened to our unknowns?**
From what the spec's GenAI note records:
- *How `OrderLine` finds a queue node from an order ID without exposing links*: **figured out.** `enqueue` returns an opaque `Node<T>` handle, `OrderLine.nodeById` stores it, and `cancel` calls `remove(node)`. O(1), and the controller and view never see a link.
- *What Remake should accept*: **figured out.** Only READY orders; cancelled or still-waiting ones are rejected.
- *How the cook types several items*: **figured out.** One line through `parseItems`.
- *Which bad inputs to handle, and the exact limits*: **figured out and implemented** (see below).
- TODO (team): check this list against the "things we don't know how to do" list in the original scope document and mark each one figured out / still open.

**What doesn't work yet?**
- MVP features: all five menu choices work. Nothing is saved: the line lives in memory only, as the MVP says.
- Bad-input cases from Section 6: not all are handled.
- Need to include a show menu option for user convienence.
- Need to display order ID from the model side on the view side for the remake option to work.
- Not automatically tested: the Ctrl+C shutdown hook in `EatzApp` (it needs a real terminal), and `ConsoleView` formatting on its own (it is only checked through the scripted run).
- No test checks the Section 4 invariants (`nodeById.size() == waiting.size()` etc.) directly, because those fields are private. They are only checked indirectly through `size()` / `getLine()`.
- `addOrder` counts orders per phone by walking the whole line (O(n)); fine for a food truck.
- Windows end-of-input is Ctrl+Z, not Ctrl+D.
- No failing tests.

## Next Steps

**What is left to finish the MVP (in order)**
1. Run the program by hand in a real terminal and try Ctrl+C and end of input (the hook is not covered by an automated test).
2. Go through the Section 6 table once more with the real program and tick off every row by hand.
3. Tidy `ConsoleView` output (alignment of the line display, clearer prompts).
4. Merge to `main` only when `.\build.ps1 test` is green.

**Our change feature (Deliverable 4): Skip / requeue the front order**
When the customer at the front of the line does not show up, the cook picks "Skip". The front
order moves to the back of the line and everyone moves up. An order can be skipped at most
2 times; a third skip is rejected with "That order was already skipped twice." (it has to be
served or cancelled). This adds new model state (a skip count per order) and new logic.

*Prediction: what it touches*

| Layer | Class and method | Change |
|---|---|---|
| Model | `OrderLine.skipFront()` | **new**: dequeue front, check skip count, `enqueue` at the back, replace its entry in `nodeById`, return an `OrderInfo` with the new position |
| Model | `Order` | **new** field `skipCount` with `getSkipCount()` and a package-private `addSkip()` (the spec's Section 4 table says Order has no changing field except `status`, so that table is updated) |
| View | `ConsoleView.showMenu()` | one new line, `6) Skip front order` |
| View | `ConsoleView.showSkipText(OrderInfo)` | **new**: prints "[TEXT to ...] We called you, now you're #n" |
| Controller | `EatzController.parseMenuChoice(String)` | range changes from 1-5 to 1-6, and its error message |
| Controller | `EatzController.run()` | one new `case 6` |
| Controller | `EatzController.handleSkip()` | **new**, same shape as `handleReady` |

*Prediction: what should NOT change*: `LinkedDeque` and `Node` (`dequeue`, `enqueue` and `remove` already do everything needed),
`OrderValidator`, `OrderInfo` (position and wait already carry what the view needs), `EatzApp`, and `OrderLine.addOrder / markReady / cancel / remake / getLine`.
We will compare this to what actually happens. The risk we see: `OrderLine.makeInfo` and
`getLine` should stay the same, but if the "skipped twice" rule needs to show up in the line
display, `OrderInfo` would have to change, which would cross a frozen contract.

**Stretch goals**
- Committing to: the Skip feature above (it is our change feature).
- Dropping: the live countdown. The spec says it needs a model listener interface, which is a bigger change than we want to defend.
- TODO (team): confirm against the stretch goals listed in the original scope document.

**New tests we will need**
- `OrderLineTest`: `skipFront` on [1,2,3] gives ids [2,3,1] with positions 1..3 and waits 3, 6, 9; skip with one order in line leaves it at position 1; skip on an empty line throws `IllegalStateException("Nobody's waiting.")`; the same order skipped a third time throws `IllegalArgumentException`; line unchanged after the rejected skip; a skipped order can still be cancelled (this checks `nodeById` was updated); a skipped order can still be served and remade.
- `OrderTest`: new orders have skip count 0; `addSkip` increases it; `skipCount` is not changed by `setStatus`.
- `EatzControllerTest`: choice `6` on an empty line shows the error once; choice `7` still shows "Please choose a number from 1 to 6."; skip then View line shows the new order.
- `LinkedDequeTest`: none expected. If one is needed, our boundary was wrong.

**Who does what (proposed, from the spec's Section 8 owners)**

| Item | Owner |
|---|---|
| `OrderLine.skipFront()` and its tests | Hannah |
| `Order.skipCount` / `addSkip` and its tests | Rohit |
| `ConsoleView.showMenu`, `showSkipText` | Bagu |
| `EatzController` changes, scripted-run test, rerun before every merge, merges to `main` | Ian |
| Hand test of Section 6 table and Ctrl+C | whole group |

TODO (team): confirm owners.

## Attribution

Owners below are taken from the Section 8 plan in the specification. TODO (team): confirm
each row and fill in the "how" (wrote / revised / reviewed / tested / GenAI-assisted).

| File | Who worked on it | Validator |
|---|---|---|
| `model/LinkedDeque.java` (includes `Node`) | Hannah | Ian |
| `model/OrderLine.java` | Hannah | Bagu |
| `model/Order.java` | Rohit | Hannah |
| `model/OrderStatus.java` | Rohit | Ian |
| `model/OrderInfo.java` | Rohit | Bagu |
| `model/OrderValidator.java` | Rohit | Ian |
| `view/ConsoleView.java` | Bagu | Hannah |
| `controller/EatzController.java` | Ian | Hannah |
| `controller/EatzApp.java` | Ian | Rohit |
| `ModelDriver.java` | Bagu | Hannah |
| `build.ps1`, `.gitignore` | Bagu | Ian |
| `test/model/LinkedDequeTest.java` | Hannah | Rohit |
| `test/model/OrderLineTest.java` | Hannah | Ian |
| `test/model/OrderTest.java` | Bagu | Rohit |
| `test/model/OrderInfoTest.java` | Rohit | Hannah |
| `test/model/OrderValidatorTest.java` | Rohit | Ian |
| `test/controller/EatzControllerTest.java` | Ian | Bagu |

### GenAI note

***During the preparation of this assignment, we used ChatGPT, in the development of EATZ for help and guidance in creating the files for the program when we were completely stuck. After using this tool, we reviewed as a team and edited the content as needed to ensure its accuracy and take full responsibility for the content in relation to grading.***
