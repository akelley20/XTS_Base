
# Project Tour

## Basics

This project is roughly organized into *Provided Objects* and *Implementation Placeholders*.

### Provided Objects

A suite of ready-to-use function blocks / objects are made available by this project and located in the "XTS" directory within the PLC project. These objects are:

- **Mover**
- **MoverList**
- **PositionTrigger**
- **Station**
- **Track**
- **Zone**

Additionally there are three "administrative" object definitions which handle some of the background tasks & quality-of-life enhancements. These include:

- **Mediator**
- **ErrorMover**
- **Objective**

Finally, we provide a single function block / object which houses the rest:

- **FB_XTS**

Detailed documentation for all of these objects is available in the *Code Reference* tab, above.


!!! tip "Intent"
    All of these objects are intended to be instantiated and used *as-is* without modifying their internal implementations- simply by calling the public methods and properties. However, for advanced solutions these objects are all provided **completely open source** for you to customize and extend as needed.

### Implementation Placeholders

A Main State Machine is provided in the **MAIN (PRG)**. Underneath this POU are ACTIONS (*Initializing*, *Recoverying*, *RecoverOneshot*, and *StationLogic*), which can be places to add your application-specific logic. Some basic motion code is already included which makes this project ready-to-run, but the intent is for you to overwrite these sections according to your needs.

The calling architecture itself is also an example, but can be manipulated to meet your organization's preferred coding methodology.

### XTS Folder Structure

This project includes a folder called `XTS (Do Not Edit)`. The goal of this project is to contain system-level code changes to within this folder, and have user customized code exist outside of this folder. By using this methodology, it should be straight-forward to perform a merge in the future to update system-level code without affecting station-to-station logic and other end user code.

This does not mean `XTS (Do Not Edit)` should never be edited. You're welcome to make changes to any part of this code as needed to fit your application. It just serves as a reminder that extra care will need to be taken if you eventually pull updated code from this repo into your project. TwinCAT and/or Git's merge tools should help with this process.