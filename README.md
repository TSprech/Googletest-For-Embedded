# Googletest For Embedded
![GitHub](https://img.shields.io/github/license/TSprech/Pull-Based-Pipeline?color=%23D22128&logo=apache) ![CPlusPlus](https://img.shields.io/badge/Version-C%2B%2B17-blue?logo=c%2B%2B&style=flat)

Guide / resources on how to use googletest on Platformio and soon other embedded platforms.

- [Googletest For Embedded](#googletest-for-embedded)
  - [Why Unit Testing On Embedded Is Important](#why-unit-testing-on-embedded-is-important)
    - [Benefits of Embedded Unit Testing on Native / Desktop Environment](#benefits-of-embedded-unit-testing-on-native--desktop-environment)
    - [Drawbacks of Embedded Unit Testing on Native / Desktop Environment](#drawbacks-of-embedded-unit-testing-on-native--desktop-environment)
  - [Two Goals For Full Embedded Unit Testing](#two-goals-for-full-embedded-unit-testing)
  - [Setting It Up](#setting-it-up)
    - [Prerequisites](#prerequisites)
    - [PlatformIO](#platformio)
  - [Setup](#setup)
    - [Googletest Main File](#googletest-main-file)
      - [Writing Your First Test](#writing-your-first-test)
        - [Testing Classes / Methods](#testing-classes--methods)
        - [Testing Free Function](#testing-free-function)
  - [Handy Notes For Writing Testable Code](#handy-notes-for-writing-testable-code)

## Why Unit Testing On Embedded Is Important
Unit testing is extremely common on desktop environments as a way to consistently make sure code behaves the way it is supposed to in all circumstances, even ones that are difficult to reach during normal operation.

### Benefits of Embedded Unit Testing on Native / Desktop Environment
- The desktop environment is much easier to work with when it comes to finding the root of bugs
- Compiling and testing on desktop is much faster than cross compiling and flashing on hardware
- Testing can be much more in depth and extensive on desktop due to less resource constraints (gigabytes vs kilo/megabytes)
- There is more support for desktop unit testing frameworks than embedded (at least free frameworks)

### Drawbacks of Embedded Unit Testing on Native / Desktop Environment
- (Most obvious) The test is not running on the hardware the final code will end up on
- Timing constraints and speed requirements are not represented in any accurate way
- Emulation of hardware peripherals results could be different than final hardware (but there are work arounds)
- A successful unit test makes no promises about the hardware working other than that the logic is correct

## Two Goals For Full Embedded Unit Testing
1. [X] Write unit tests for the logic and functions / methods that run on desktop
2. [ ] Write unit test that run on the embedded system that actually interface with the hardware and peripherals

## Setting It Up
### Prerequisites
- Install MinGW and add gcc/g++ to the path
- Download googletest from it's [repo](https://github.com/google/googletest)
- Create a PlatformIO project

### PlatformIO
1. Take the `googletest → include → gtest` folder and copy it into the `include` folder of your project
2. Create a new folder in the `test` folder called `native`
3. Create a new `whateveryouwanttonameit.cpp` file and `#include "gtest/gtest.h"`
4. Insert the following block as your main function:
```cpp
int main(int ac, char* av[]) {
  testing::InitGoogleTest(&ac, av);
  return RUN_ALL_TESTS();
}
```
5. Add the following to your `platformio.ini` file:
```ini
[env:native]
platform = native
lib_compat_mode = off
build_unflags = -std=gnu++11
build_flags = -std=c++17
```
6. Run the command `pio test --environment native` and it should say success *Note: You may need to using the command `pio platform install "native"` to have PlatformIO recognize the native platform*

## Setup
### Googletest Main File
The main file of your `whateveryouwanttonameit.cpp` should contain the following main function definition from the setup above:
```cpp
int main(int ac, char* av[]) {
  testing::InitGoogleTest(&ac, av);
  return RUN_ALL_TESTS();
}
```

#### Writing Your First Test
##### Testing Classes / Methods
*I bring these up first as most of the time you will be testing with objects (since this is C++ after all).* Testing these use the `TEST_F` macro. I would recommend referring to the googletest documentation for details about the intricacies of it as it requires a separate class for managing the object.

##### Testing Free Function
Free function are tested using the `TEST` macro. These are a good way to start out testing and get familiar with the framework.

## Handy Notes For Writing Testable Code
If you're using the Arduino library (which is usually the case with PlatformIO) wrap any Arduino function calls in wrapper functions. For example instead of writing:
```cpp
void MyFunction() {
  ...
  Wire.beginTransmission(address);
  Wire.write(value);
  Wire.endTransmission();
  ...
}
```
Write:
```cpp
void MyFunction() {
  ...
  I2CBeginTransmission(address);
  I2CWrite(value);
  I2CEndTransmission();
  ...
}
```
and have a header file similar to this:
```cpp
#define UNITTESTINGENABLED // Define when testing on native
#ifndef UNITTESTINGENABLED
  #include "Wire.h"

  uint8_t I2CBeginTransmission(uint16_t address) {
    return Wire.beginTransmission(address);
  }
  void I2CWrite(uint8_t value) {
    Wire.write(value);
  }
  void I2CEndTransmission() {
    Wire.endTransmission();
  }
  ...
  // <rest of the Wire functions>
#else
  uint8_t i2c_return_value;
  uint8_t i2c_set_value;

  // This function will set the value that will be returned when I2CRead is called
  void I2CReturnValue(uint8_t value) {
    i2c_return_value = value;
  }

  // This function will retrieve the value that your I2C methods write to the I2C bus when I2CWrite is called
  uint8_t I2CSetValue() {
    return i2c_set_value;
  }

  uint8_t I2CBeginTransmission(uint16_t address) {
    // Doesn't do anything on the desktop environment
  }
  void I2CWrite(uint8_t value) {
    i2c_set_value = value;
  }
  void I2CEndTransmission() {
    // Doesn't do anything on the desktop environment
  }
  ...
  // <rest of the I2C desktop functions>
#endif
```

While the above is by no means exhaustive (eg only dealing with 8 bit read/write), it give one way to deal with emulating hardware and capturing a representation of how functions are interacting with the hardware. Note that:
- No hardware specific header files are included when you are testing (eg `Wire.h`)
  - This is because the computer will most likely not have any idea what to do with hardware manipulation commands
- Functions that interact with the hardware manipulate variables instead of hardware
  - Ex: The I2C function that reads a value from hardware instead just returns a value that is set earlier in a test
  - Ex: If you wanted to set a pin to a certain state, have the hardware calls set a boolean variable and check the state of that variable in your test
  - If your function deals with sending many values, it may be easiest to have a FIFO buffer or similar which is filled and cleared as needed