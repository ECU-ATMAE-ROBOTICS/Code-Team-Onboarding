# ControllerCommon Reference

## Class: `XboxController`

```python
XboxController(controllerNum: int = 0, deadZone: float = 0)
```

### Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `controllerNum` | `int` (optional) | `0` | Index of the joystick to open. Passed to `pygame.joystick.Joystick()`. |
| `deadZone` | `float` (optional) | `0` | Axis movement threshold; axis values within `[-deadZone, deadZone]` will be treated as zero, except for axes 4 and 5 (these axes are the left and right trigger). |

---

## Functions / Methods

### `getLayout()`

Print the mapping of detected logical input types to numeric input IDs that the adapter will produce.

**Parameters**
- None.

**Behavior**

For each key in `self.inputIDs` (`"D"`, `"A"`, `"B"`) prints a line like: `"D: [0, 1, 2, 3, 4]"`.

```
D: [0, 1, 2, 3, 4]                                            # All D-Pad button IDs
A: [5, 6, 7, 8, 9, 10]                                        # All Axis IDs
B: [11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 26]  # All Button IDs
```

**Usage notes**
- Use to discover which numeric IDs correspond to D-pad states, axes, and buttons from this controller instance.

---

### `getInputID(input: str)`

Extracts the input ID portion from a message produced by `getControllerInput()`.

**Parameters**
- `input` (`str`): A message string in the format `"[Input ID]:[Input Value]\n"`

**Return value**
- (`str`) The substring before the first `":"` character (the Input ID).

**Examples**
- `getInputID("2:1\n")` → `"2"`
- `getInputID("10:0.6\n")` → `"10"`

**Notes**
- If `":"` is missing, `input.find(":")` returns `-1` and slicing `input[:-1]` will drop the last character. In practice, `getInputID` is intended to be used with `getControllerInput` output, which always contains `":"`.

---

### `getInputValue(input: str)`

Extracts the input value portion from a message produced by `getControllerInput()`.

**Parameters**
- `input` (`str`): A message string in the format `"[Input ID]:[Input Value]\n"`.

**Return value**
- (`str`) The input value. AKA the substring after the `":"` character.

**Examples**
- `getInputValue("2:1\n")` → `"1\n"`
- `getInputValue("10:0.6\n")` → `"0.6\n"`

**Notes**
- The reason for the trailing newline character is so the Arduino can easily parse the instruction.

---

### `getControllerInput()`

Retrieves new controller inputs (buttons, axes, D-pad) since the last pygame event polling and returns them as a list of simple string messages for easy parsing/serial transmission.

**Internal Behavior**

- Calls `pygame.event.get()` and iterates events.
- **For `JOYBUTTONUP` and `JOYBUTTONDOWN`:**
  - Maps `event.button` index to a numeric ID via `inputIDs["B"]` and fetches the button state via `self.controller.get_button(event.button)`.
  - Appends string `f"{id}:{value}\n"` to the message list.
- **For `JOYAXISMOTION`:**
  - Reads axis value: `axisValue = round(self.controller.get_axis(event.axis), 1)`
  - Maps `event.axis` index to an ID via `inputIDs["A"]`
  - If `axisValue > deadZone` or `axisValue < -deadZone` OR `event.axis` is 4 or 5:
    - Append `f"{id}:{axisValue}\n"`
  - Else:
    - Append `f"{id}:0\n"` (zero out small movements).
- **For `JOYHATMOTION` (D-pad):**
  - Reads `dPadVal = self.controller.get_hat(event.hat)` (tuple `(x, y)`)
  - Maps the D-pad states into one of five IDs:

    | D-Pad Tuple | Mapped Output |
    |---|---|
    | `(0, 0)` | `dictVal[0]:0` |
    | `(0, 1)` | `dictVal[1]:1` |
    | `(1, 0)` | `dictVal[2]:2` |
    | `(0, -1)` | `dictVal[3]:3` |
    | `(-1, 0)` | `dictVal[4]:4` |

  - Each mapping appends `f"{id}:{value}\n"` to the message list.

**Return Value**

- `list[str]` — A list of strings, each in the format `"[Input ID]:[Input Value]\n"`.
  - Example return: `["0:0\n", "6:0.5\n", "12:1\n"]` or `[]` if no events occur.
  - IDs are integers encoded as decimal strings. Values are `"0"` or `"1"` for buttons/D-pad, or a float (one decimal place) for axes.

**Edge cases and notes**

- This method only reports events returned by `pygame.event.get()`; if no new events exist, an empty list is returned.
- Axis values are rounded to one decimal place (`round(..., 1)`).
- Axes 4 and 5 are always reported regardless of dead zone; other axes are zeroed when within the dead zone.
  - This is due to the fact that axes 4 and 5 are controller triggers.
- The returned strings include a trailing newline character (`'\n'`) to facilitate serial parsing.

---

## Files with Test Helper Functions

### `Tests/ViewInputMappings.py`

**Purpose**

Connect to the detected controller and print any detected input values live to the console. This script uses the `XboxController` wrapper to map events and then prints the input in the `id:value` format.

- Use this script to test how ControllerCommon maps the IDs of the controller buttons and axes.

### `Tests/testControllerConnection.py`

**Purpose**

Connect to the first detected controller and display current axis/button/D-pad values live in the console.

- This is raw pygame API usage; useful if you want to work directly with `XboxControllerCommon`.