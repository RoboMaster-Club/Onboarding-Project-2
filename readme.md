# Onboarding Project - Introduction to Embedded 2

## Background Info

A PWM (Pulse Width Modulation) signal is a simple digital signal often used to emulate an analog signal or control motors and servos. It works by rapidly switching the signal on and off so that the average voltage matches the desired analog value. Here are some key terms to understand when discussing PWM signals:

- **Period**: The duration of one complete cycle of the signal.
- **Duty Cycle**: The percentage of one period in which the signal is "on." For example, a PWM signal with a 25% duty cycle is "on" 25% of the time and "off" 75% of the time.
- **Resolution**: The number of discrete steps in the PWM signal. For example, an 8-bit resolution PWM can have 256 different duty cycle levels (2^8).

The embedded processor we use has a single core, so we will use FreeRTOS to manage priority and execute several tasks simultaneously. Tasks could include sending motor instructions over CAN or handling drivetrain calculations.

![RTOS Scheduling Image](https://open4tech.com/wp-content/uploads/2019/11/preemptive_scheduling.jpg)

[image source](https://open4tech.com/rtos-scheduling-algorithms/)

## Overview

The end goal of this project is to use an STM32 chip to generate and toggle a PWM signal and LED simultaneously, but at different rates. This project will guide you through the step-by-step process of creating a project in CubeMX, assigning pinout, concurrent scheduling using FreeRTOS, and using timers.

## Prerequisites

- A laptop, (either MacOS or Windows) with a usb port.
- Basic knowledge of C
- Completed [Introduction to Embedded 1](https://github.com/RoboMaster-Club/Onboarding-Project-1)

## [!] Start Here

### Part 1: Download Tools and Clone Repository

In order to complete this project you will need to download the following software:

- CubeMX - Used to configure STM32 microcontrollers and generate initialization code. Download the appropriate version from [here](https://www.st.com/en/development-tools/stm32cubemx.html#st-get-software). It may ask you to create an ST account, so just use your school email.
  You will also need the same [tools used in the last project](https://github.com/RoboMaster-Club/Onboarding-Project-1?tab=readme-ov-file#part-1-download-tools).
  Clone the repository using the following command in your terminal:

```
git clone https://github.com/RoboMaster-Club/Onboarding-Project-2.git
```

### Part 2: Create Your First CubeMX Project

1. Open CubeMX and locate the "Home" screen. It may ask you to create an account first.
2. Open the "Create from Board" menu and search for the **NUCLEO-L432KC** board in the "Commercial Part Number" menu.
3. Double-click the correct board to generate the CubeMX project. When it asks if you want to initialize the board to its default mode, press "No" so that the board will be in a clean state. You should see the following if you completed the steps correctly:
   ![New Project Image](/images/New_Project.png)
4. Click on the "Project Manager" tab.
   - Name your project `Onboarding_Project_2`.
   - Set the project location to the parent folder of your cloned repository. For example, if your clone is located at `Clones/Onboarding_Project_2`, select the parent folder `Clones` as the project location.
   - In the "Application Structure" dropdown, select "Basic".
   - From the "Toolchain/IDE" dropdown, select "Makefile".

### Part 3: Enable FreeRTOS

1. Return to the "Pinout & Configuration" tab. Open the "Middlewares and Software Packs" menu from the sidebar on the left. Everything should be greyed out except for "FATFS" and "FREERTOS". Click on "FREERTOS".
2. In the "Interface" dropdown, select "CMSIS_V2".

### Part 4: Configure the PWM Pin Out

1. Click on the "Timers" section on the left side of the "Pinout and Configuration" toolbar.
2. Click on "TIM1" as this will be used to generate the PWM signal.
3. In the new menu that opens, find "Channel 1" and select "PWM Generation CH1" from the dropdown.
4. You should see the "PA8" pin turn green in the Pinout view, indicating that it has been successfully enabled.
5. In the Configuration menu, open the "Counter Settings" and "PWM Generation Channel 1" menus. Configure them according to the following images:
   ![Counter Settings Image](/images/Counter_Settings.png)
   ![PWM Settings Image](/images/PWM_Settings.png)

- Here are a few key settings to understand:
  - **Prescaler** -> `0` (each tick of the clock corresponds to one tick of the timer).
  - **Counter mode** -> `Up` (the timer counts up from 0).
  - **Counter Period** -> `255` (8-bit resolution signal).
  - **Auto-reload preload** -> `Enable` (resets the counter when it hits the maximum period).
  - **Pulse** -> `127` (50% duty cycle).
  - **CH Polarity** -> `High` (sets the channel output to high when active).

### Part 5: Generate and Setup Project

1. Hit the blue "Generate Code" button in the top right. It will give you a warning but ignore it for now.
2. Open your generated project using **VSCode**. If you did everything correctly, you should see the following file structure:
   ![File Structure Image](/images/File_Structure.png)
3. Edit the `Makefile` file and add the following lines to the very end (after where it says `# *** EOF ***`):

```
ECHO_WARNING=echo "\033[33m[Warning]\033[0m"
ECHO_SUCCESS=echo "\033[32m[Success]\033[0m"
flash: $(BUILD_DIR)/$(TARGET).bin
	@$(ECHO_WARNING) "Flashing the binary to the board"
	@openocd -f board/st_nucleo_l4.cfg -c "init; reset halt; \
	flash write_image erase $(BUILD_DIR)/$(TARGET).bin 0x08000000; reset run; shutdown"
	@$(ECHO_SUCCESS) "Flashing Complete"
```

4. Test that everything works as expected by running the **build task**. You should see a new `build/` folder appear once it finishes.

### Part 6: Create FreeRTOS Tasks

1. Open the `main.c` file located in `Src/`.
2. Create a [function prototype](https://www.geeksforgeeks.org/function-prototype-in-c/) under the comment on line 63 that says `USER CODE BEGIN PFP` by adding the following line:

```
void Toggle_LED(void *pvParameters);
```

- `*pvParameters` is a pointer to any data you want to pass to a FreeRTOS task. In this case, we will simply pass `NULL`.

3. We will create the FreeRTOS Task by using `xTaskCreate()` on line 130 under the comment that says `USER CODE BEGIN RTOS_THREADS`.

```
xTaskCreate(Toggle_LED, "Toggle_LED", 128, NULL, 1, NULL);
```

4. Finally, we will define the function outside of `main()` (which ends around line 154).

```
void Toggle_LED(void *pvParameters)
{
	while (1)
	{
		HAL_GPIO_TogglePin(GPIOB, GPIO_PIN_3);
		osDelay(1000);
	}
}
```

- In this function, we define an infinite loop where we call the `HAL_GPIO_TogglePin()` function to toggle GPIO Pin B3, then use `osDelay()` to wait 1000ms before the loop repeats. This is very similar to the function you wrote in the last project which can be found [here](https://github.com/RoboMaster-Club/Onboarding-Project-1?tab=readme-ov-file#part-3-your-first-function).

5. Next, you will repeat the process of steps 2-4 (create function prototype, task, define function) with the function given below:

```
void Toggle_PWM(void *pvParameters)
{
	static uint8_t on = 0;
	while (1)
	{
		on ^= 1;
		if (on)
		{
			HAL_TIM_PWM_Start(&htim1, TIM_CHANNEL_1);
		}
		else
		{
			HAL_TIM_PWM_Stop(&htim1, TIM_CHANNEL_1);
		}
		osDelay(500);
	}
}
```

- `static uint8_t on = 0;` defines an unsigned 8-bit integer which keeps its value between function is calls.
- `on ^= 1;` uses XOR operator to toggle the `on` variable between 1 and 0.
- `HAL_TIM_PWM_Start()` and `HAL_TIM_PWM_Stop()` start/stops the PWM generation on TIM1 CH1 (using the settings we configured in CubeMX in [Part 4](https://github.com/RoboMaster-Club/Onboarding-Project-2?tab=readme-ov-file#part-4-configure-the-pwm-pin-out)).

### Part 7: Flash and Test

When you have completed all the previous parts, come to a meeting where we will provide you with a board and an additional LED to test your code. You will follow the same [procedure from the last project](https://github.com/RoboMaster-Club/Onboarding-Project-1?tab=readme-ov-file#part-5-flash-the-code). Both LEDs should blink simultaneously, with the external LED blinking at twice the rate of the green one.
