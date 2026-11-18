# Command Pattern – Undo / Redo (LLD #31)

<!-- nav -->
⬅ [30 Flyweight Word Processor](30_Flyweight_Word_Processor_Notes.md) · [Index](00_LLD_Index.md) · [32 Structural Patterns Overview](32_Structural_Patterns_Overview_Notes.md) ➡
<!-- nav -->

Sources: [Command PDF](https://drive.google.com/file/d/183IjgJ6pQDHswDQiwWMazetTpdK6SWaV/view?usp=drive_link) · [Code on GitLab](https://gitlab.com/shrayansh8/lld-lowleveldesign/-/tree/main/src/main/java/com/conceptcoding/behavioralpatterns/command)

## In simple words
**Command** turns a **request into an object**. Instead of calling `ac.turnOn()` directly, you create a `TurnOnCommand` object that knows *what to do* and *how to undo it*. Because commands are objects, you can store them, queue them, log them, and undo or redo them.

Think of ordering at a restaurant: the waiter writes your order on a slip (the command). The kitchen later executes it. The slip can be queued, repeated or cancelled.

## The problem it solves
A remote control with buttons for an air conditioner. If each button calls the AC directly, the remote is tied to every device action, and there is no way to **undo** the last action.

## The solution
- **Command interface** – `ICommand` with `execute()` and `undo()`.
- **Concrete commands** – `TurnOnCommand`, `TurnOffCommand`, `SetTemperatureCommand`. Each holds the **receiver** (the AC) and remembers the *previous* state (previous temperature or on/off) so it can undo.
- **Receiver** – `AirConditioner`: does the real work (`turnOn`, `turnOff`, `setTemperature`).
- **Invoker** – `RemoteController`: stores the current command and a **history stack** of executed commands. It doesn't know what the commands do.

**Diagram – Remote control with undo**

```mermaid
classDiagram
  class AirConditioner {
    +boolean isOn
    +int temperature
    +turnOn() void
    +turnOff() void
  }
  class ICommand {
    <<interface>>
    +execute() void
    +undo() void
  }
  class RemoteController {
    +ICommand command
    +Stack~ICommand~ commandHistory
    +pressButton() void
    +undo() void
  }
  class SetTemperatureCommand {
    -AirConditioner ac
    -int newTemperature
    -int previousTemperature
    +execute() void
    +undo() void
  }
  class TurnOffCommand {
    -AirConditioner ac
    -boolean previousState
    +execute() void
    +undo() void
  }
  class TurnOnCommand {
    -AirConditioner ac
    -boolean previousState
    +execute() void
    +undo() void
  }
  ICommand <|.. SetTemperatureCommand
  ICommand <|.. TurnOffCommand
  ICommand <|.. TurnOnCommand
  RemoteController --> ICommand
  SetTemperatureCommand --> AirConditioner
  TurnOffCommand --> AirConditioner
  TurnOnCommand --> AirConditioner
```

**Diagram – execute then undo**

```mermaid
sequenceDiagram
  participant C as Client
  participant R as RemoteController (invoker)
  participant T as SetTemperatureCommand
  participant A as AirConditioner (receiver)
  C->>R: setCommand(new SetTemperatureCommand(ac, 24))
  C->>R: pressButton()
  R->>T: execute()
  T->>T: remember previousTemperature (say 20)
  T->>A: setTemperature(24)
  R->>R: push command onto history
  C->>R: undo()
  R->>R: pop last command
  R->>T: undo()
  T->>A: setTemperature(20)
```

*Invoker with a history stack*

```java
    public void pressButton() {
        command.execute();
        commandHistory.push(command);
    }

    public void undo() {
        if (!commandHistory.isEmpty()) {
            ICommand lastCommand = commandHistory.pop();
            lastCommand.undo();
        }
    }
}
```

## How redo works
The code above keeps one stack (`commandHistory`) for **undo**. To add **redo**, keep a second stack:
- `undo()`: pop from the undo stack, call `undo()`, **push onto the redo stack**.
- `redo()`: pop from the redo stack, call `execute()`, push back onto the undo stack.
- Any *new* command clears the redo stack (the old future is no longer valid).

```mermaid
graph LR
  E["execute(cmd)"] --> US["undo stack: push"]
  US -->|"undo()"| RS["redo stack: push"]
  RS -->|"redo()"| US
  E -. "clears" .-> RS
```

This is how text editors implement Ctrl+Z / Ctrl+Y.

## When to use
- You need undo/redo, history or an audit log.
- You want to queue, schedule or retry operations (job queues, macros, transactions).
- You want to decouple *who asks* from *who does*.

## Pros and cons
- ✅ Decouples sender and receiver; new commands without changing the invoker (Open/Closed).
- ✅ Easy undo/redo, macros (a command made of commands), queues and logging.
- ❌ One small class per action.
- ❌ Undo needs the command to store enough state to reverse itself.

**Related:** **Memento** stores a snapshot of state (useful for undo too); **Strategy** swaps algorithms, while Command encapsulates a *request*.

---

<!-- nav -->
⬅ [30 Flyweight Word Processor](30_Flyweight_Word_Processor_Notes.md) · [Index](00_LLD_Index.md) · [32 Structural Patterns Overview](32_Structural_Patterns_Overview_Notes.md) ➡
<!-- nav -->
