# Application Mover

The most common need to customize system provided objects is to add functionality to the [`Mover`](../CodeReference/Objects/Mover.md) object. In order to make this as simple as possible, while adhering to the folder and code maintenance structure discussed in [XTS Folder Structure](../GettingStarted/4_ProjectTour.md#xts-folder-structure), a stub extension of the mover object has been provided as `ApplicationMover (FB)`.

By placing custom code within the Application Mover FB you can add properties, actions and methods to the mover and have some assurance that they will not be reverted by updates to the underlying Mover object within this repo.

## Extending
The provided stub code handles the most basic needs of extending the mover and includes a simple example property addition. ApplicationMover.Cyclic will be called automatically by FB_XTS, however the included call to `SUPER^.Cyclic(GroupReference)` must be maintained in the application mover .Cyclic method.

## Accessing the ApplicationMover
Chainable functions in XTS such as Mover.MoveToStation() or MoverList.GetMoverByLocation() return standard XTS movers. In order to access extended methods and properties an indirect reference to the XTS mover array must be made from the [`.MoverIndex`](../CodeReference/Objects/Mover.md#moverindex) property.

```javascript
// get the lead mover's index from a mover list
LeadMoverIndex := MoverList.GetMoverByLocation( 1, 900, MC_Positive_Direction ).MoverIndex;

// access the application mover specific property .PositionLag
LeadMoverLag := XTS.Mover[LeadMoverIndex].PositionLag
```
# Example Method
This example code provides a method to move a mover based on a desired ramp time and maximum velocity, instead of the pre-configured `MotionParameters` already assigned to the mover. It uses [MC_CalcDynamicsByRampTime](https://infosys.beckhoff.com/content/1033/tcplclib_tc2_mc2/13611300875.html?id=1298283191705576852) from the Tc2_MC2 library.

Start by creating a method on the application called MoveToStationByRampTime with a return type of ApplicationMover.

```javascript
// method declarations
VAR_INPUT
	DestinationStation	: REFERENCE TO Station;	
    Velocity     : LREAL;
    RampTime     : LREAL;
    Stiffness    : LREAL;
	Gap			 : LREAL;
END_VAR
VAR
	MotionParameters: MotionParameters_typ;
END_VAR
VAR_INST
	fbMC_CalcDynamicsByRampTime: MC_CalcDynamicsByRampTime := (Velocity := 1, RampTime := 1, Stiffness := 0.5);
END_VAR
```

```javascript
// method code

// use the motion library to calculate move parameters
fbMC_CalcDynamicsByRampTime(
	Velocity:= Velocity, 
	RampTime:= RampTime, 
	Stiffness:= Stiffness, 
	Error=> , 
	ErrorID=> , 
	Acceleration=> , 
	Deceleration=> , 
	Jerk=> );

// update the motion parameters of the mover
MotionParameters.Velocity			:= Velocity;
MotionParameters.Acceleration		:= fbMC_CalcDynamicsByRampTime.Acceleration;
MotionParameters.Deceleration		:= fbMC_CalcDynamicsByRampTime.Deceleration;
MotionParameters.Jerk				:= fbMC_CalcDynamicsByRampTime.Jerk;
MotionParameters.Gap				:= Gap;
MotionParameters.Direction			:= Tc3_Mc3Definitions.MC_Direction.mcDirectionPositive;

// test for error
IF NOT fbMC_CalcDynamicsByRampTime.Error THEN
	// no error, update parameters call underlying MoveToStation
	SUPER^.MotionParameters:= MotionParameters;
	SUPER^.MoveToStation(DestinationStation);
ELSE
	// report the error
	ErrorId		:= fbMC_CalcDynamicsByRampTime.ErrorID;
	ErrorOrigin	:= CONCAT( InstancePath, 'MoveToStationByRampTime' );
END_IF

// return the application mover to make it chainable
MoveToStationByRampTime := THIS^;
```

The new method can now be called by using the indirect reference method described in [Accessing the Application Mover](#accessing-the-applicationmover).
```javascript
If XTS.Station[1].MoverInPosition Then
	// get the mover index
	MoverIndex := XTS.Station[1].CurrentMover.MoverIndex;
	// call the "by ramp time" method
	XTS.Mover[MoverIndex].MoveToStationByRampTime(
		DestinationStation:= XTS.Station[1],	// destination station
		Velocity:= 500, 											// max velocity
		RampTime:= 1.5, 											// desired ramp time in seconds
		Stiffness:= 0.5, 											// stiffness of move 0..1, larger values equate to higher jerk
		Gap:= 85);														// colission avoidance gap
END_IF;
```