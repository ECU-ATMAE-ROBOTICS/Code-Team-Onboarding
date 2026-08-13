# Usages and Examples

## Initial Connection Code

```python
import pygame
import time

controller_connected = False
controller = None

# Connect to the Xbox Controller
while not controller_connected:
    try:
        pygame.init()
        controller = XboxController(deadZone=0.2)
        controller_connected = True
    except pygame.error:
        pygame.joystick.quit()
        time.sleep(1)
```

## Robot Control

```python
while True:
    # Select robot mode
    instructions = controller.getControllerInput()
    if instructions != None:
        for instruction in instructions:
            print(instruction)
            inputID = int(controller.getInputID(instruction))
            match inputID:
                # View Button
                case 21:
                    logger.info(f"|{datetime.now().strftime('%H:%M:%S')}|In auto mode")
                    robotmodes.auto(controller, arduino)
                    logger.info(f"|{datetime.now().strftime('%H:%M:%S')}|In neutral mode")
                    break
                # Menu Button
                case 22:
                    logger.info(f"|{datetime.now().strftime('%H:%M:%S')}|In teleop mode")
                    robotmodes.teleop(controller, arduino)
                    logger.info(f"|{datetime.now().strftime('%H:%M:%S')}|In neutral mode")
                    break
    time.sleep(.1)
```

> Snippet found in `ATMAE-2025`

## More Examples

- Driving the 2025 Robot
- Setting the sorting colors for 2025 robot