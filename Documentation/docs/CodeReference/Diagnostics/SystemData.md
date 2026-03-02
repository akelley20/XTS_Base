
# XTS System Data

The outputs from the `XTS` (FB_XTS) object in the `MAIN` POU consolidates some commonly used mover and system information into one space to help simplify basic troubleshooting and visualization on higher level systems or HMIs.

## Fieldbus

The data in MAIN.XTS outputs has deliberately been chosen to use arrays of elementary data types that can be sent through most field buses in their default configuration. This can help facilitate status and error annunciation a higher level control system if one is used. 

## MoverData

The information available for each individual mover is as follows. This data is available as outputs from the `FB_XTS` function block.

- MoverPositions - a REAL representing the current position of the mover on the track.
- MoverErrors - a BOOL when true indicates that there is an error with this mover.
- MoverErrorIDs - a UDINT with an error code produced this mover. Error codes can be looked up in Beckhoff Infosys.
- MoverStates - a DINT which indicates the state of each mover such as idle, enabling, run, or error. The list of states is available in the code at `XTS\DUTs\MoverState_enum`.

![GVL.MoverData data view](../../Images/GVLMoverData.png)


## State data

The output `.XtsState`, the state information for the overall XTS state machine including disabled, enabled stopping or error. The list of states is available in the code at `XTS\DUTs\XTSStateMachine_enum`. This value is helpful when visualizing or debugging the xts from a higher level system.

