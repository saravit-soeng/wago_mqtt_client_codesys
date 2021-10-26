# Wago MQTT Client on E!Cockpit(Codesys-based)

Configuration Cloud Connectivity (MQTT Anycloud) on Wago WBM as below:
![image](https://user-images.githubusercontent.com/19525030/138884622-c57d5a61-50fa-4fce-a311-4cf9df2f4b2e.png)

Program structure Main Program (PLC_PRG)
```
PROGRAM PLC_PRG
VAR
END_VAR
________________________________________________________________________________________________
Publish_NativeMQTT();
Status_NativeMQTT();
```
Function for convert from bytes array to decimal (bigendian_real)
```
FUNCTION bigendian_real : REAL
VAR_INPUT
	pArrayInput: POINTER TO BYTE;
END_VAR
VAR
	aTemp: ARRAY[0..3] OF BYTE;
	aTemp1: ARRAY[0..3] OF BYTE;
END_VAR
________________________________________________________________________________________________
memcopy(ADR(aTemp), pArrayInput, 4);

aTemp1[3] := aTemp[0];
aTemp1[2] := aTemp[1];
aTemp1[1] := aTemp[2];
aTemp1[0] := aTemp[3];

memcopy(ADR(bigendian_real), ADR(aTemp1[0]), 4);
```
Program for publish message to MQTT Broker (Publish_NativeMQTT)
```
PROGRAM Publish_NativeMQTT
VAR
	// variable for storing sensor data
	inputData: ARRAY[0..23] OF BYTE;
	aPeaktoPeakX: REAL;
	aPeaktoPeakY: REAL;
	aPeaktoPeakZ: REAL;
	temperature: REAL;

	MyInterval: TIME := T#1S;
	Timer: TON;
	
	aBuffer: ARRAY[0..255] OF BYTE;
	dwByteCount: DWORD;
	
	oFbPublish: WagoAppCloud.FbPublishMQTT;
	xMyTrigger: BOOL := FALSE;
	dwBusyCounter: DWORD := 0;
	dwErrorCounter: DWORD := 0;
	
	// JSON Writer
	sTemplate: STRING(JSON_MAX_STRING) := '{"a_peak_to_peak_x":#Parameter,"a_peak_to_peak_y":#Parameter,"a_peak_to_peak_z":#Parameter,"temperature":#Parameter,"sensor_id":"1"}'; // Define sensor_id in advanced - make sure it not duplicated
	asParameters: ARRAY[0..3] OF STRING;
	myWriter: Fb_JSON_Writer_01;
	sPayload: STRING(JSON_MAX_STRING);
	xTriggerJSON: BOOL;
	xWriter_Done: BOOL;
	xWriter_Error: BOOL;
	myStatus: WagoSysErrorBase.FbResult;
	
END_VAR
________________________________________________________________________________________________
Timer(IN := TRUE, PT := MyInterval);
IF Timer.Q THEN
	xTriggerJSON := TRUE;
	Timer(IN := FALSE);
	
	IF NOT oFbPublish.xBusy THEN
		// Copy JSON to buffer
		dwByteCount := Length(sPayload);
		MemCopy(pDest := ADR(aBuffer), pSource := ADR(sPayload), udiSize := dwByteCount);
		
		// Trigger the transmission
		xMyTrigger := TRUE;
	ELSE
		// Busy statistics counter
		dwBusyCounter := dwBusyCounter + 1;		
	END_IF
	
	IF oFbPublish.xError THEN
		// Error statistics counter
		dwErrorCounter := dwErrorCounter + 1;
	END_IF
	
END_IF

BuildPayload();

// Trigger MQTT Publish
oFbPublish(
	sTopic := 'test',
	eQualityOfService := 1,
	dwSize := dwByteCount,
	aData := aBuffer,
	xTrigger := xMyTrigger
);
```
Define action in Publish_NativeMQTT program (BuildPayload)
```
// Get data from sensor
inputData := IoConfig_Globals_Mapping.dataInput;
aPeaktoPeakX := bigendian_real(ADR(inputData[4]));
aPeaktoPeakY := bigendian_real(ADR(inputData[8]));
aPeaktoPeakZ := bigendian_real(ADR(inputData[12]));
temperature := bigendian_real(ADR(inputData[16]));

asParameters[0] := REAL_TO_STRING(aPeaktoPeakX);
asParameters[1] := REAL_TO_STRING(aPeaktoPeakY);
asParameters[2] := REAL_TO_STRING(aPeaktoPeakZ);
asParameters[3] := REAL_TO_STRING(temperature);

myWriter(
	sJSON_BaseFrame := sTemplate,
	oStatus => myStatus,
	xError => xWriter_Error,
	xDone => xWriter_Done,
	aParameterValues := asParameters,
	xTrigger := xTriggerJSON,
	sOutput := sPayload
);
```
Program for check MQTT connection status(Status_NativeMQTT)
```
PROGRAM Status_NativeMQTT
VAR
	oFbStatus: WagoAppCloud.FbStatus_NativeMQTT;
	xMyError: BOOL := FALSE;
	xConnectedToMyCloud: BOOL := FALSE;
	rMyFillLevel: REAL := 0;
	uliMyOutgoingBlocks: ULINT := 0;
END_VAR
________________________________________________________________________________________________
// Will detect new values with default interval of every 5 second
oFbStatus(
	xEnabled := TRUE,
	xError => xMyError,
	xCloudConnected => xConnectedToMyCloud,
	rCacheFillLevel => rMyFillLevel,
	uliOutgoingDataBlocks => uliMyOutgoingBlocks
);
```
