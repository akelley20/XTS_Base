# Custom objects

This example makes use of additional position trigger objects defined outside of `FB_XTS` in a reusable, object-oriented solution. Understanding the use of the [XTS Parameters](../CodeReference/Parameters.md) is necessary for understanding this example.

## The Application

A basic assembly and inspection station consists of the following processes in order:

1. Load part A to mover
1. Inspect part A at a line scan camera
1. Load part B into part A on mover
1. Inspect part A and B at a line scan camera
1. Unload completed part

Stations 1, 3 and 5 are basic stop-wait-release stations. The line scan cameras in steps 2 and 4 require the movers to move at a defined speed and trigger the camera at the appropriate point. This example will use custom position triggers defined inside a reusable line scan camera FB to create these stations.

## Solution discussion

While this application can be solved using only the built in `XTS.Station[]` and `XTS.PositionTrigger[]` objects, care will need to be taken to keep track of which station indexes and position trigger indexes are grouped together and in which order. In this very small example this is not difficult to manage, but in larger applications these types of mappings can become cumbersome.

Here is one way this could be implemented using only `FB_XTS` objects

- `XTS.Station[1]` feeds the line scan camera surrounded by `XTS.PositionTrigger[1]` and `XTS.PositionTrigger[2]`
- `XTS.PositionTrigger[2]` feeds `XTS.Station[2]`
- `XTS.Station[2]` feeds the line scan camera surrounded by `XTS.PositionTrigger[3]` and `XTS.PositionTrigger[4]`
- `XTS.PositionTrigger[4]` feeds `XTS.Station[3]`
- Movers return to station 1

From the above sequence you can see that it very quickly becomes difficult to trace the flow through these processes.

By createing a new line scan camera object to contain the position triggers and skipping some unneeded station indices we can make this much more readable

- XTS.Station[1] feeds LineScan2
- LineScan2 feeds XTS.Station[3]
- XTS.Station[3] feeds LineScan4
- LineScan4 feeds XTS.Station[5]

## The line scan station

Because of the need for speed changes two position triggers will be used before and after the line scan camera to adjust the mover's speed. The first position trigger will also start the camera's scanning.

The FB `LineScanCamera` will be created to implement this solution.

### Declarations

We start by declaring two position triggers inside the `LineScanCamera`.

```javascript
	StartPositionTrigger : PositionTrigger; // slow down and trigger camera
	EndPositionTrigger : PositionTrigger; // speed up and check camera results
```

We'll also need an interface to a camera and a few other intermediate variables

```javascript
VAR INPUT
	CameraSpeed	: LREAL;	// speed to slow to for camera
	ExitSpeed : LREAL;	// speed to next station

	StartPosition : LREAL; // slow down point
	EndPosition : LREAL; // speed up point
END_VAR

VAR_IN_OUT
	Camera : CameraInterface;	// a camera interface with trigger, pass and fail bits
END_VAR

VAR
	TargetMover : iMover; // a temporary variable attached to the mover at the position triggers
	Initialized : bool;	// first-run flag for initializing the position triggers
END_VAR
```

### Initialization

Position triggers need to be registered with the XTS system. This accomplishes two things:

- The PositionTrigger.Cyclic() routine is called automatically for us by the [Mediator](../CodeReference/Objects/Mediator.md)
- The position trigger will automatically be populated with all movers on the system.

We also need to set the locations of the position triggers.

```javascript
IF NOT Initalized THEN
	// set the initialized flag (one-shot this if statement)
	Initialized = true;

	// copy locations to position triggers
	StartPositionTrigger.Position := StartPosition;
	EndPositionTrigger.Position := EndPosition;

	// register the position triggers with the XTS system
	MAIN.XTS.System.AddPositionTrigger(StartPosition);
	MAIN.XTS.System.AddPositionTrigger(EndPosition);
END_IF
```

### Start position trigger

The start position trigger slows down the mover and triggers the camera. 

[Position Trigger](../CodeReference/Objects/PositionTrigger.md) and the Mover's [Payload](../CodeReference/Objects/Mover.md#payload) are used by this code. Details of their use can be found in the links.

```javascript
// check for the position trigger's flag
IF StartPositionTrigger.MoverPassedPosition THEN
	// get the mover that most recently passed this location
	TargetMover = StartPositionTrigger.CurrentMover;

	// there is no need to slow down or scan an empty mover
	// check for valid data and or product on this mover via the Payload parameter
	IF <TargetMover.Payload checks> THEN
		// slow down the mover
		TargetMover.SetVelocity(CameraSpeed);
		// trigger the camera
		Camera.Trigger := TRUE;
	END_If

	// clear the position trigger flag
	StartPositionTrigger.MuteCurrent();
END_IF;
```

### End Position Trigger

The end position trigger checks the camera result and speeds up the mover. It does not directly send the mover to the next station, as the station destination was already set by the preceding load station.

[Position Trigger](../CodeReference/Objects/PositionTrigger.md) and the Mover's [Payload](../CodeReference/Objects/Mover.md#payload) are used by this code. Details of their use can be found in the links.

```javascript
// check for the position trigger's flag
IF EndPositionTrigger.MoverPassedPosition THEN
	// get the mover that most recently passed this location
	TargetMover = EndPositionTrigger.CurrentMover;

	// if the camera was triggered we're working on this mover and need to clean up
	IF (Camera.Trigger) THEN
		// reset the camera trigger
		Camera.Trigger := FALSE;

		// assign the camera result to the mover's payload
		// e.g. TargetMover.Payload.CameraResult = Camera.Result

		// speed up the mover
		TargetMover.SetVelocity(ExitSpeed);
	END_IF

	// clear the position trigger flag
	EndPositionTrigger.MuteCurrent();
END_IF
```

### MAIN Implementation

First declarations on `MAIN` need to be added for the two position triggers

```javscript
	// line scan camera speed and trigger controls
	LineScan2 : LineScanCamera;
	LineScan4 : LineScanCamera;

	// for clarity the camera interfaces are declared here, but the actual camera communication interface is not part of the scope of this example
	Camera2 : CameraInterface;
	Camera4 : CameraInterface;
```

Then within the `MAIN.StationLogic` we'll add code to implement the load, scan, load, scan and unload sequence defined above. More complete examples and discussion of [Station-to-Station](./StationRouting.md) motion is available.

Pay careful attention to the sequence of how movers are sent from one station to the next.

- XTS.Station[1] sends its mover to XTS.Station[3]
- XTS.Station[3] sends its mover to XTS.Station[5]

Since we do not stop at the line-scan camera, we can send the movers directly to the next stop position. The line scan code will adjust the mover's speed accordingly, but will not change the destination.

```javascript

// first station waits for part to be loaded
IF XTS.Station[1].MoverInPosition THEN
	// trigger loading mechanism
	// wait for load complete
	// update mover payload
	// release the mover to the next station
	XTS.Station[1].CurrentMover.MoveToStation(XTS.Station[3])
END_IF

// Call the first camera station (station 2) with appropriate parameters
LineScan2(
	CameraSpeed := 100.0,		// mm/s in front of the camera
	ExitSpeed := 1000.0,		// mm/s when scanning is complete
	StartPosition := 200.0,	// mm
	EndPosition := 300.0,		// mm
	Camera := Camera2,
)

// third station waits for part to be loaded
IF XTS.Station[3].MoverInPosition THEN
	// trigger loading mechanism
	// wait for load complete
	// update mover payload
	// release the mover to the next station
	XTS.Station[3].CurrentMover.MoveToStation(XTS.Station[5])
END_IF

// Call the second camera station (station 4) with appropriate parameters
LineScan4(
	CameraSpeed := 50.0,		// mm/s in front of the camera
	ExitSpeed := 1000.0,		// mm/s when scanning is complete
	StartPosition := 600.0,	// mm
	EndPosition := 700.0,		// mm
	Camera := Camera4,
)

// third station waits for part to be unloaded and sends the empty mover back to the start at station 1
IF XTS.Station[5].MoverInPosition THEN
	// trigger loading mechanism
	// wait for load complete
	// update mover payload
	// release the mover to the next station
	XTS.Station[5].CurrentMover.MoveToStation(XTS.Station[1])
END_IF
```