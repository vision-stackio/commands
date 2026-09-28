<p align="center">
<a>
<img alt="Vision Logo" src="assets//Vision.png" width="132">
</a>
</p>

<p align="center">
<a href="#language-reference"><img alt="Language" src="https://img.shields.io/badge/language-vision%20script-blueviolet?style=flat-square" /></a>
<a href="#robot-commands"><img alt="Commands" src="https://img.shields.io/badge/robot%20commands-50-2ea44f?style=flat-square" /></a>
<a href="#who-can-run-what"><img alt="Access" src="https://img.shields.io/badge/access-admin%20only-E7352C?style=flat-square" /></a>
<a href="#limits"><img alt="Max instructions" src="https://img.shields.io/badge/max%20instructions-500-lightgrey?style=flat-square" /></a>
</p>

<p align="center">
<b>Vision Script</b>: A small, safe, line oriented language for scripting Vision's robot. It is compiled to a flat instruction list, never run as code.
</p>

> **Note:** Vision Script is **not JavaScript** and cannot run arbitrary code. Loops are unrolled, variables are substituted, arithmetic is folded, and `IF` branches are resolved **at compile time**. The robot only ever receives a flat, fully inspectable list of commands, and every one of them still goes through the `CommandRegistry` (schema validation, authorization, safety, execution).

---

## Table of Contents
1. [How it works](#how-it-works)
2. [Quick start](#quick-start)
3. [Language reference](#language-reference)
4. [Robot commands](#robot-commands)
5. [Who can run what](#who-can-run-what)
6. [Limits](#limits)
7. [Examples](#examples)
8. [Errors and diagnostics](#errors-and-diagnostics)
9. [API](#api)
10. [Gotchas](#gotchas)

<br/>

## How it works

```
Source -> Lexer -> Parser -> AST -> Semantic Analysis -> IR -> CommandRegistry -> Safety -> Execution
```

| Stage | Job |
| :---- | :-- |
| **Lexer** | Splits each line into tokens. Comments (`#`) are removed. Reports unterminated strings. |
| **Parser** | Builds the syntax tree. Enforces one statement per line and the nesting limit. |
| **Semantic analysis** | Checks that every command exists, variables are declared, types match, and `WAIT` / `REPEAT` values are in range. |
| **IR generation** | Unrolls loops, picks the taken `IF` branch, folds arithmetic, rolls `RANDOM`, and substitutes variables. Output is a flat list of `EXEC`, `SLEEP` and `LOG` instructions. |
| **Executor** | Runs the list in order. Each `EXEC` goes through `CommandRegistry`. `PRINT` only writes to the server log. |

Because everything is decided before the robot moves, a compiled script is exactly as fixed and inspectable as a hand-written list of commands. Every instruction remembers its source line and column, so a failure is reported as "line 4 failed", not "step 4 of 6 failed".

<br/>

## Quick start

1. Sign in to the dashboard as an **ADMIN** and open **Compiler** (`/dashboard/compiler`).
2. Type a script:
   ```
   EYE_CENTER
   WALK_FORWARD 1000
   AUDIO_SPEAK "Hello"
   ```
3. **Compile** it. This is a dry run that touches no hardware, and it shows diagnostics or the instruction list.
4. To **execute** it, switch the system to **Manual mode** first (same page, admin only). In Autonomous mode the server rejects execution.

The terminal widget on the Compiler page can also run Vision Script one line at a time.

<br/>

## Language reference

One statement per line. `#` starts a comment, unless it is inside a quoted string. Blank lines are ignored. Keywords, command names and variable names are **case-insensitive** (`$i` and `$I` are the same variable).

### Values

| Kind | Written as | Notes |
| :--- | :--------- | :---- |
| Number | `500`, `-20`, `1.5` | Integers and decimals, negative allowed |
| String | `"Hello there"` | Double quotes only |
| Variable | `$angle` | Must be declared before use |

### Statements

| Statement | Syntax | What it does |
| :-------- | :----- | :----------- |
| **Command** | `NAME [value]` | Runs a robot command with at most **one** optional argument. See [Robot commands](#robot-commands). |
| **WAIT** | `WAIT <ms>` | Sleeps. A number or numeric variable, from 0 to 60,000. |
| **SET** | `SET $name <value or expression>` | Declares or redeclares a variable. |
| **RANDOM** | `RANDOM $name <min> <max>` | Random whole number from `min` to `max`, inclusive. |
| **PRINT** | `PRINT <value>` | Writes to the server log. Never touches the robot. |
| **REPEAT** | `REPEAT <count> TIMES [AS $name]` ... `END` | Repeats the block `count` times. |
| **IF** | `IF <a> <op> <b> [THEN]` ... `[ELSE ...]` `END` | Runs one branch. |

### Variables (`SET`)

```
SET $speed 50
SET $name "Sam"
SET $copy $speed
SET $total $speed + 10
```

- Using a variable before it is declared is an error (`UNDECLARED_VARIABLE`).
- Using a text variable where a number is required (`WAIT`, a `REPEAT` count, a numeric argument) is an error (`TYPE_MISMATCH`).

### Arithmetic

`+ - * /`, evaluated strictly **left to right with no operator precedence**:

```
SET $x 2 + 3 * 2      # = 10, not 8
SET $y $x - 4 / 2     # (10 - 4) / 2 = 3
SET $g "Hello " + $name   # + joins two strings
```

- Operators must have **spaces around them**. `$a+1` is read as one variable name.
- Only `SET` accepts an expression. A command argument cannot (`EYE_SET $a + 1` is a parse error). Compute it into a variable first.
- Dividing by zero is a compile error, never a runtime crash.

### RANDOM

```
RANDOM $look 30 150
EYE_SET $look
```

Rolled **once, at compile time**. Compiling the same script twice can give different numbers, but one compiled program is fixed. It is not a new random draw every time the script runs. Both bounds must be whole numbers and `min` must not exceed `max`.

### PRINT

```
PRINT "reached this point"
PRINT $angle
```

Writes `[script:<robotId>] <message>` to the server log with no hardware effect. Handy for seeing what a script computed.

### REPEAT

```
REPEAT 4 TIMES
  WALK_FORWARD 500
  TURN_LEFT 300
END

REPEAT 3 TIMES AS $step
  EYE_SET $step
  WAIT 200
END
```

- The count is a whole number from 1 to 100, or a variable holding one.
- Inside the loop, `$I` (or the name you give with `AS`) is the **0-based** iteration index. Inner loops can use an outer loop's index.
- `REPEAT 3 TIMES` compiles to three copies of its body. No loop exists at run time.

### IF / ELSE

```
IF $angle > 90
  TURN_RIGHT 300
ELSE
  TURN_LEFT 300
END
```

| Operator | Works on |
| :------- | :------- |
| `>` `<` `>=` `<=` | Numbers only |
| `==` `!=` | Numbers or text, but both sides must be the same type |

- `THEN` after the comparison is optional. `ELSE` must be on its own line. `END` closes both `IF` and `REPEAT`.
- One comparison only. There is no `AND` / `OR`.
- The comparison is decided at compile time. Only the taken branch is compiled in, and the other leaves no trace. Both branches must still be valid script, so a typo in the losing branch is still an error.

```
REPEAT 4 TIMES
  IF $I >= 2
    WALK_BACKWARD 500
  ELSE
    WALK_FORWARD 500
  END
END
```

<br/>

## Robot commands

Write a command by name, with **at most one argument**: a number, a string, or a variable.

```
EYE_SET 90
AUDIO_SPEAK "Hello there"
DANCE $duration
```

### How the argument is used

The script passes the same value under every plausible field name, and each command keeps only the field it defines:

| You write | It is passed as |
| :-------- | :-------------- |
| A **number** | `angle`, `speed`, `durationMs`, `step` |
| A **string** | `text`, `name`, `trackId` |

This is why `WALK_FORWARD 1000` means a duration, `LEFT_LEG_FORWARD 50` means a speed, and `FIND_PERSON "Sam"` means a name.

> **Compile time vs. run time.** The compiler checks that a command **exists**, but it does **not** check the command's argument range or whether a required argument is missing. `EYE_SET 500` compiles fine and is rejected when it runs. Execution **stops at the first failed command**.

> **Looking.** The neck servo was removed. `EYE_*` commands keep working, but "looking" is done by turning the whole body with the leg servos, and the camera and ultrasonic sensor are fixed forward.

### Movement and body

| Command | Argument | Range | Manual mode | Notes |
| :------ | :------- | :---- | :---------: | :---- |
| `WALK_FORWARD` | optional ms | 0 to 60,000 | yes | Checks live ultrasonic distance first |
| `WALK_BACKWARD` | optional ms | 0 to 60,000 | yes | Checks ultrasonic first |
| `TURN_LEFT` | optional ms | 0 to 60,000 | yes | Checks ultrasonic first |
| `TURN_RIGHT` | optional ms | 0 to 60,000 | yes | Checks ultrasonic first |
| `WALK_STOP` | none | | no | Always allowed |
| `EYE_CENTER` | none | | no | Always allowed |
| `EYE_SET` | **required** angle | 0 to 180 (90 = center) | yes | |
| `EYE_LEFT` | optional step | 1 to 90 (default 15) | yes | Relative turn |
| `EYE_RIGHT` | optional step | 1 to 90 (default 15) | yes | Relative turn |
| `EYE_TRACK_PERSON` | none | | yes | Use bare. Its `personId` / `targetAngle` cannot be given from a script |

### Individual legs

Everything below exists for both `LEFT_LEG_*` and `RIGHT_LEG_*`.

| Command | Argument | Range | Manual mode | Notes |
| :------ | :------- | :---- | :---------: | :---- |
| `LEFT_LEG_FORWARD` | **required** speed | 0 to 100 | yes | Checks ultrasonic first |
| `LEFT_LEG_REVERSE` | **required** speed | 0 to 100 | yes | Checks ultrasonic first |
| `LEFT_LEG_ROTATE` | **required** speed | -100 to 100 | yes | Negative reverses the rotation |
| `LEFT_LEG_JOINT` | **required** angle | 0 to 180 | yes | Positional joint servo |
| `LEFT_LEG_STOP` | none | | no | Always allowed |
| `LEFT_LEG_CENTER` | none | | no | Always allowed |

### Audio

| Command | Argument | Range | Notes |
| :------ | :------- | :---- | :---- |
| `AUDIO_SPEAK` | **required** text | 1 to 1,000 characters | Text to speech |
| `AUDIO_START_LISTENING` | none | | |
| `AUDIO_STOP_LISTENING` | none | | |
| `AUDIO_STOP` | none | | Stops current speech |

### Missions and performance

| Command | Argument | Range | Notes |
| :------ | :------- | :---- | :---- |
| `FIND_PERSON` | **required** name | 1 to 80 characters | Starts a bounded wander-and-scan patrol. It is a patrol, not real navigation |
| `DANCE` | optional ms, or a string | 2,000 to 120,000 ms; string up to 80 characters | Beat-synced when a real track is playing, fixed tempo otherwise |
| `PLAY_SONG_REQUEST` | **required** text | 1 to 200 characters | Starts the music search, getting-to-know-you chat and beat-synced dance |
| `SMART_DEVICE_TRIGGER` | **required** device name | 1 to 128 characters | Device must be registered by an admin |

`FIND_PERSON` and `DANCE` start a background task and return right away with a `taskId`. The script does not wait for them to finish. Check progress with `GET /api/tasks`, and stop early with `POST /api/tasks/:id/cancel`.

### Sensors and camera

| Command | Argument | Admin only | Notes |
| :------ | :------- | :--------: | :---- |
| `ULTRASONIC_READ` | none | no | |
| `ULTRASONIC_ENABLE` | none | yes | |
| `ULTRASONIC_DISABLE` | none | yes | |
| `ULTRASONIC_CALIBRATE` | none | yes | Its distance settings cannot be given from a script, so bare use does nothing useful |
| `CAMERA_CAPTURE` | none | no | |
| `CAMERA_START_TRACKING` | none | no | |
| `CAMERA_STOP_TRACKING` | none | no | |
| `BRAIN_OBSERVE` | none | no | Use bare. `mediaId` cannot be given from a script |

### Safety

| Command | Argument | Admin only | Notes |
| :------ | :------- | :--------: | :---- |
| `SERVOS_SAFE` | none | yes | Puts servos in a safe state. Always allowed regardless of mode |
| `EMERGENCY_STOP` | none | yes | Immediate stop. Always allowed regardless of mode |

### In the catalog but not usable from a script

These need named fields that a single script argument cannot fill. Use the dashboard or API for them.

| Command | Needs |
| :------ | :---- |
| `EMERGENCY_CLEAR` | `reason` (admin only) |
| `MEMORY_SEARCH` | `query` |
| `MEMORY_LEARN` | `title`, `content` |
| `BRAIN_PROCESS` | `observationId` |
| `TASK_START` / `TASK_CANCEL` | `taskType` / `taskId` |
| `SCRIPT_COMPILE` / `SCRIPT_EXECUTE` | `source` |
| `SET_DANCE_PREFERENCE` | `personId`, `enabled` |
| `RESET_PERSON_FACE` | `personId` |

<br/>

## Who can run what

| Action | Requirement |
| :----- | :---------- |
| Open the Compiler page or call `/api/scripts/*` | **ADMIN** role, enforced by the server |
| **Compile** | ADMIN, and the compiler must be enabled. Works in either control mode and never touches hardware |
| **Execute** | ADMIN, compiler enabled, **and Manual mode on** |
| Stop, center and safe commands | Always allowed once a script runs |
| Movement commands (`EYE_SET`, `WALK_*`, leg motion) | Rejected outside Manual mode |
| Admin-only commands (`EMERGENCY_STOP`, `SERVOS_SAFE`, `ULTRASONIC_*`) | ADMIN |

- **Control mode:** `AUTONOMOUS` (default) means the brain drives and script execution is refused. `MANUAL` means a human drives. Toggle it on the Compiler page (`POST /api/system/control-mode`, ADMIN only).
- **Compiler kill switch:** an admin can disable the compiler entirely. Compile **and** execute are then rejected for everyone, including the dashboard terminal, with `COMPILER_DISABLED`. It is on by default.
- **Delivery:** movement, audio and camera-mode commands are queued and picked up by the robot when it polls `GET /api/commands/pending`. A command can be reported `COMPLETED` by the server slightly before the robot has physically finished it.

<br/>

## Limits

All limits are in `server/src/compiler/limits.ts` and `SemanticAnalyzer.ts`.

| Limit | Value | Error |
| :---- | :---- | :---- |
| `REPEAT` count | 1 to 100, whole number | `REPEAT_COUNT_OUT_OF_RANGE` |
| Nesting depth (`REPEAT` and `IF` share it) | 5 levels | `PARSE_ERROR` |
| Total instructions after unrolling | 500 | `COMPILE_LIMIT_EXCEEDED` |
| `WAIT` | 0 to 60,000 ms | `WAIT_OUT_OF_RANGE` |
| Statements per line | 1 | `PARSE_ERROR` |
| Arguments per command | 1 | `PARSE_ERROR` |

The 500 count includes every `EXEC` command, every `WAIT`, and every `PRINT` after loops are unrolled. Nested loops multiply, so `REPEAT 100` inside `REPEAT 100` is rejected.

<br/>

## Examples

**A square patrol with a message**

```
EYE_CENTER
AUDIO_SPEAK "Starting patrol"
REPEAT 4 TIMES AS $side
  WALK_FORWARD 1000
  TURN_RIGHT 400
  PRINT $side
END
WALK_STOP
```

**A look-around that is different each time you compile**

```
RANDOM $look 30 150
EYE_SET $look
WAIT 500
SET $back 180 - $look
EYE_SET $back
```

**Branching on the loop index**

```
REPEAT 4 TIMES
  IF $I >= 2
    WALK_BACKWARD 500
  ELSE
    WALK_FORWARD 500
  END
END
```

**Variables, text and a mission**

```
SET $who "Sam"
SET $greeting "Hello " + $who
AUDIO_SPEAK $greeting
FIND_PERSON $who
DANCE 20000
```

<br/>

## Errors and diagnostics

Every diagnostic carries the exact line and column. The dashboard shows them the way a compiler would:

```
3:1: error: 'WALK_FORWAD' is not a registered Vision command
  WALK_FORWAD 500
  ^
```

| Code | Meaning | Example cause |
| :--- | :------ | :------------ |
| `LEX_ERROR` | Source cannot be tokenized | `AUDIO_SPEAK "Hello` (no closing quote) |
| `PARSE_ERROR` | Tokens do not form a valid statement | `EYE_CENTER WALK_FORWARD` on one line, a missing `END`, an extra argument |
| `UNKNOWN_COMMAND` | Not a registered command | A typo in a command name |
| `UNDECLARED_VARIABLE` | Variable used before `SET` / `RANDOM` | `EYE_SET $angle` with no `SET $angle` |
| `TYPE_MISMATCH` | Wrong type for the operation | Text in `WAIT`, `>` on text, `-` on a string |
| `WAIT_OUT_OF_RANGE` | `WAIT` outside 0 to 60,000 | `WAIT 90000` |
| `REPEAT_COUNT_OUT_OF_RANGE` | Count outside 1 to 100 | `REPEAT 0 TIMES` |
| `DIVIDE_BY_ZERO` | Division by a value that is zero | `SET $x 5 / 0` |
| `RANDOM_RANGE_INVALID` | `min` is greater than `max` | `RANDOM $a 10 5` |
| `COMPILE_LIMIT_EXCEEDED` | A limit hit only once loops unroll | Over 500 instructions, or a loop-index value that turns out invalid |

The compile result contains `success`, `ast`, `ir` (only when successful) and `diagnostics`. Run-time failures are reported per step with `sourceLine` and `sourceColumn`.

<br/>

## API

All routes require an authenticated **ADMIN**.

| Method | Path | Body | Returns |
| :----- | :--- | :--- | :------ |
| `POST` | `/api/scripts/compile` | `{ "source": "..." }` | `{ success, ast, ir, diagnostics }` |
| `POST` | `/api/scripts/execute` | `{ "source": "...", "robotId": "vision-001" }` | `{ compiled, executionResults }` |

`robotId` is optional and defaults to `vision-001`.

Refusals:

| HTTP | `code` | When |
| :--: | :----- | :--- |
| 403 | `COMPILER_DISABLED` | An admin turned the compiler off |
| 403 | `AUTONOMOUS_MODE_ACTIVE` | Executing while Manual mode is off |

If compilation fails, `execute` returns the diagnostics and an empty `executionResults`, and nothing runs.

<br/>

## Gotchas

* **Variables set inside a block stay inside it.** A `SET` inside `REPEAT` or `IF` lives only in that block, and inside a loop it starts fresh every iteration. A counter like `SET $n $n + 1` inside a loop will **not** accumulate. Use the loop index (`$I`) instead.
* **Redeclaring inside a block shadows.** It does not change the outer variable of the same name.
* **Spaces matter around operators.** Write `$x > 5` and `$a + 1`. `$x>5` is read as a single variable name and fails with `UNDECLARED_VARIABLE`.
* **Compile passes do not guarantee run success.** Command argument ranges and missing required arguments are only checked when the command runs, and the first failure stops the script. Any commands before it have already run.
* **`RANDOM` is fixed per compile.** It is not fresh on each run.
* **Both `IF` branches must be valid.** A mistake in the branch that is not taken is still a compile error.
* **Only one argument, and never an expression.** Compute values into a variable first, then pass the variable.
* **Compile is safe, execute is not.** Compile as often as you like, but a script that walks the robot is real. Stop commands always work.

<br/>