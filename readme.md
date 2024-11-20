# Lab 01
## Step one - Toggle the state of the led
### CubeMX part
1. Create the project using CubeMX
2. Configure the pin of the button to generate an interruption on a rising pulse.
3. Activate the button interruption on the NVIC
4. Set up the project path and enable the generation of the .c/.h files of the peripherals
5. Using the "stm32pio" tool, generate the source files using the command:
```sh
stm32pio new -d ./ -b nucleo_f401re 
``` 

### VSCode part
Now with the code generated, we have to create the fuction that is going to be called each time the button is pressed/released (depending on the trigger of the interruption)

For this, we add in the "USER CODE 4" section of the main.c file, the following code
```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin){ 
    if(GPIO_Pin == B1_Pin){ 
        HAL_GPIO_TogglePin(LD2_GPIO_Port, LD2_Pin); 
    } 
} 
```
When an interrupt is made by te button, the function "HAL_GPIO_EXTI_Callback" is called, where the parametter "GPIO_Pin" is the number of the pin that generate the interruption.

Inside the function, the pin number is compared with the pin number of the button, to check that the interruption was made by the button. Finally, the function "HAL_GPIO_TogglePin" is called to change the state of the led. Fot that, is required to pass as parameter the the pin number of the led, and also the GPIO port where is connected.

### PlatformIO + Renode config
To setup Renode with PlatformIO so the program can be tested we need to modify the platform.ini file

```ini
[env]
platform = ststm32
board = nucleo_f401re
framework = stm32cube
board_build.stm32cube.custom_config_header = yes

[env:renode]
upload_protocol = custom
upload_command = renode $UPLOAD_FLAGS
upload_flags =
    -e include @./nucleo-f401re.resc
    -e sysbus LoadELF @$PROG_PATH
    -e start

[platformio]
include_dir = Inc
src_dir = Src
```
To finish the setup we need to copy some files into the working directory:
- nucleo-f401re.repl
- nucleo-f401re.resc
- STM32DMAv2.cs
- stm32f401.repl

Now, each time we "upload" the program. Renode is going to be launched simulating the board.

### Problems
While configuring Renode, an unexpected problem was found. The way the CRC was called/declared was changed in a version of renode, so the file "stm32f401.repl" required a change/update: 

```
crc: CRC.STM32_CRC @ sysbus 0x40023000
    series: STM32Series.F4
```

## Step two - Blink Led using timer
In this project, we need to enable and setup the timer 10 to generate an interruption each "x" seconds. This value is by default 1, but whe the button is pressed, changes to 2, and then to 10 after pressing it again. If, the button is pressed again, it returns to 1.
Finally, each interruption of the timer should make the led toggle.
### CubeMX Part
1. Create the project
2. Enable the timer 10 and its interruption
3. Enable interruption of the button
4. Generate the code as in the [Step one](#cubemx-part)

### VsCode Part
With the code generated, we have to set up the parameters "Autoreload register (ARR)" and the "Prescaler register (PSC)" from the timer 10 using the following formula:


$$
Period=\frac{(ARR+1)\times(PSC+1)}{TimerClockFreq}
$$
Important: For this case, the $TimerClockFreq=84MHz$

The selected value for the PSC is 65000, and the ARR is goning to be variable. Depending on the period that is required (1s, 2s, 10s)
So for:
- 1s  $ARR=1292$
- 2s  $ARR=2584$
- 10s $ARR=12920$

Now that we have all the parameters we create the following function on the "USER CODE 4" section of the main.c file:

```c
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin){ 
  if(GPIO_Pin == B1_Pin){ 
    HAL_TIM_Base_Stop_IT(&htim10); //Stop timer
    time_selector = (time_selector!=2) ? time_selector + 1 : 0; //Update time selector
    switch (time_selector){
      case 0:
        __HAL_TIM_SET_AUTORELOAD(&htim10, 1292); //Set timer to 1s
      break;

      case 1:
        __HAL_TIM_SET_AUTORELOAD(&htim10, 2584); //Set timer to 2s
      break;

      case 2:
        __HAL_TIM_SET_AUTORELOAD(&htim10, 12920); //Set timer to 10s
      break;
      
      default:
        __HAL_TIM_SET_AUTORELOAD(&htim10, 1297); //Set timer to 1s
        time_selector = 0;
      break;
    }
    __HAL_TIM_SET_COUNTER(&htim10, 0); //Reset timer
    HAL_TIM_Base_Start_IT(&htim10); //Start timer
  }
} 
```

The function is called on each button 1 press, same as in the [step one](#vscode-part). This stops the timer, changes the value of the ARR register in a round-robin fashing, resets the timer and, finally, restarts it.

Another important code fragment to add is the
```c
HAL_TIM_Base_Start_IT(&htim10);
```
in the "USER CODE 2" section of the same file (main.c).