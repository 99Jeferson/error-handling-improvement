#  Error Handling and Logging Improvement Project

##  Project Overview

This project is based on a simple Python Calculator CLI application obtained from GitHub. The application allows users to perform basic arithmetic operations such as addition, subtraction, multiplication, and division.

The purpose of this assignment is to analyze poor error handling practices in the code, improve exception handling, introduce meaningful logging, and compare AI-generated suggestions with human reasoning.


##  Identified Problems in the Original Code

The original project had weak error handling practices such as:

### 1. Generic Exception Usage

```python
try:
    num = int(input("Enter a number: "))
except:
    print("Something went wrong")

## Issues:

Catches all exceptions without specificity
Makes debugging difficult
Provides unclear error messages


### 2. No Logging Mechanism

Errors were only printed to the console
No record of system failures
Difficult to trace issues later


##  Improvements Made

### 1. Specific Exception Handling

```python
try:
    num = int(input("Enter a number: "))
except ValueError:
    print("Invalid input. Please enter a valid number.")

## Benefits:

Handles only relevant errors
Improves clarity for users
Makes debugging easier


### 2. Advanced Exception Handling

```python
try:
    result = a / b
except ZeroDivisionError as e:
    print("Cannot divide by zero")

## Benefits:

Prevents application crash
Handles edge cases effectively

##  Logging Implementation

The Python logging module was introduced to replace print statements.

```python
import logging

logging.basicConfig(
    filename='app.log',
    level=logging.ERROR,
    format='%(asctime)s - %(levelname)s - %(message)s'
)

try:
    num = int(input("Enter a number: "))
except ValueError as e:
    logging.error("Invalid input provided", exc_info=True)

###  Advantages of Logging:
Maintains error history in `app.log`
Helps in debugging and monitoring
Provides detailed traceback information


## AI vs Human Reasoning

### AI-Generated Suggestions

Use try-except blocks
Replace print statements with logging
Handle common exceptions like ValueError

### Human Reasoning and Decisions

* Selected **specific exceptions** instead of generic ones
* Added `exc_info=True` to capture full error trace
* Structured logging format for better readability
* Focused on user-friendly error messages

## Key Learning Outcomes

 Importance of proper error handling in software development
 Difference between generic and specific exceptions
 Practical use of Python logging module
 Writing maintainable and debuggable code


## Conclusion

This project demonstrated how poor error handling can reduce software reliability. By implementing structured exception handling and logging,
the application became more robust, user-friendly, and easier to debug.
The comparison between AI suggestions and human reasoning highlighted the importance of thoughtful decision-making in programming.


## References

* Python Documentation (Logging Module)
* GitHub Open Source Projects
