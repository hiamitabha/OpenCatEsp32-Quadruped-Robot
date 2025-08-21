# Gait Control Implementation Guide

## Overview

This document describes the enhanced gait control functionality implemented in the OpenCat ESP32 firmware. The system now supports more precise control over gait execution by allowing arguments to be passed with gait commands.

## Features

### 1. **Timed Straight Gaits**
Execute a straight gait for a specific duration:
```
k wkF 5000    # Walk forward for 5000ms (5 seconds)
k bkF 3000    # Walk backward for 3000ms (3 seconds)
```

### 2. **Angle-Controlled Turning Gaits**
Execute a turning gait until a specific angle is reached:
```
k wkL 90      # Turn left (clockwise) 90 degrees
k wkR 45      # Turn right (counterclockwise) 45 degrees
```

### 3. **Polar Coordinate System**
The system uses polar coordinate convention where:
- Positive angles are counterclockwise (right turn)
- Negative angles are clockwise (left turn)
- This matches standard mathematical conventions

## Technical Implementation

### 1. Yaw Coordinate System Correction

**Problem**: Original ypr[0] (yaw angle) direction was inconsistent with polar coordinate definition:
- Original: positive angle = clockwise direction
- Expected: positive angle = counterclockwise direction (polar coordinate convention)

**Solution**: Negate ypr[0] at the IMU data source rather than at usage time.

**Modifications**:
- `src/imu.h - readIMU()` function: Add `ypr[0] = -ypr[0];` for both ICM42670 and MPU6050
- `src/imu.h - print6Axis()` function: Use `-icm.ypr[0]` and `-mpu.ypr[0]` for display
- Remove negation operations in usage locations

### 2. Turning Control Variables

```cpp
// In src/imu.h
bool turningQ = false;           // Turning control switch
float targetYawAngle = 0.0;     // Target angle to reach
float initialYawAngle = 0.0;    // Initial angle when turning started
bool needTurning = false;        // Flag to prevent turning exception from being skipped
```

### 3. Enhanced T_SKILL Command Processing

**Argument Parsing**:
- Parse skill commands with space-separated arguments
- Extract skill name and timing/angle parameter
- Support both formats: `k wkF 5000` or `k wkL 90`

**Straight Gaits (F suffix)**:
- Add task to queue with specified timing
- Execute gait for specified duration

**Turning Gaits (L/R suffix)**:
- Set up turning control parameters
- Record initial yaw angle
- Calculate target angle based on direction and offset
- Normalize angles to -180 to 180 degree range

### 4. IMU Exception Handling

**New Exception Type**:
```cpp
#define IMU_EXCEPTION_TURNING -7
```

**Detection Logic**:
- Monitor current yaw difference from initial angle
- Check if target angle is reached or exceeded
- Set `imuException = IMU_EXCEPTION_TURNING` when target reached
- Disable turning control: `turningQ = false`

**Exception Processing**:
- Handle `IMU_EXCEPTION_TURNING` in `dealWithExceptions()`
- Add "up" task to stop robot and make it stand
- Print debug information

## Serial Command Usage

### Command Format

The enhanced T_SKILL command supports arguments for precise gait control:

```
k <skill_name> <argument>
```

Where:
- `k` is the T_SKILL token
- `<skill_name>` is the gait skill name (e.g., wkF, wkL, wkR, bkF)
- `<argument>` is the timing (for straight gaits) or angle (for turning gaits)

### Straight Gait Commands

For straight gaits (skills ending with 'F'), the argument represents execution time in milliseconds:

```
k wkF 5000    # Walk forward for 5 seconds
k bkF 3000    # Walk backward for 3 seconds
k vtF 2000    # Trot forward for 2 seconds
```

**Logic**:
1. Parse skill name and timing argument
2. Add task to queue: `tQueue->addTask('k', "wkF", 5000)`
3. Task queue executes the gait for specified duration
4. Robot automatically stops after timing expires

### Turning Gait Commands

For turning gaits (skills ending with 'L' or 'R'), the argument represents target angle in degrees:

```
k wkL 90      # Turn left (clockwise) 90 degrees
k wkR 45      # Turn right (counterclockwise) 45 degrees
k vtL 180     # Trot left 180 degrees
k vtR 30      # Trot right 30 degrees
```

**Logic**:
1. Parse skill name and angle argument
2. Set up turning control:
   - `turningQ = true`
   - `initialYawAngle = ypr[0]` (current yaw)
   - `targetYawAngle = initialYawAngle ± angle` (based on direction)
3. IMU continuously monitors yaw angle
4. When target reached: `imuException = IMU_EXCEPTION_TURNING`
5. Exception handler adds "up" task: `tQueue->addTask('k', "up")`
6. Robot stops and stands up

### Coordinate System

The system uses polar coordinate convention:
- **Positive angles** = Counterclockwise (right turn)
- **Negative angles** = Clockwise (left turn)
- **wkR** = Turn right (counterclockwise, positive angle)
- **wkL** = Turn left (clockwise, negative angle)

### Task Queue Integration

You can combine these commands with the task queue system for complex sequences:

```
# Basic sequence: walk forward, turn left, walk forward again
qk wkF 2000:k wkL 90:k wkF 2000:

# Complex sequence with different gaits
qk wkF 3000:k vtL 45:k bkF 1000:k vtR 90:k up:1000:

# Mixed commands with delays
qk sit:1000:k wkF 2000:k wkR 180:k up:500:
```

**Task Queue Format**:
```
q<token><command>:<delay>><token><command>:<delay>...
```

### Backward Compatibility

Commands without arguments work exactly as before:

```
k sit        # Sit posture (no timing/angle)
k up         # Stand up posture
k wkF        # Walk forward (continuous, no timing)
k wkL        # Walk left (continuous, no angle control)
```

### Debug Output

The system provides detailed debug information:

**When starting turning gait**:
```
Started turning gait: wkL
initial yaw: 45.2
target angle: 135.2
turning direction: LEFT (CW)
```

**When target is reached**:
```
Turning target reached! Current yaw: 135.1
Target was: 135.2
EXCEPTION: turning target reached
endTurn
```

### Error Handling

- **Invalid arguments**: Commands with invalid numbers are ignored
- **Missing arguments**: Commands without arguments work as before
- **Angle normalization**: Angles are automatically normalized to -180 to 180 degrees
- **Exception protection**: Turning exceptions are protected from being skipped

### Examples

#### Basic Movement Patterns
```
# Square pattern
k wkF 2000    # Forward 2s
k wkR 90      # Turn right 90°
k wkF 2000    # Forward 2s
k wkR 90      # Turn right 90°
k wkF 2000    # Forward 2s
k wkR 90      # Turn right 90°
k wkF 2000    # Forward 2s
k wkR 90      # Turn right 90°

# Circle pattern
k wkF 1000    # Forward 1s
k wkL 30      # Turn left 30°
k wkF 1000    # Forward 1s
k wkL 30      # Turn left 30°
# ... repeat for full circle
```

#### Advanced Sequences
```
# Patrol pattern with task queue
qk wkF 3000:k vtL 90:k wkF 2000:k vtR 90:k wkF 3000:k vtL 90:k wkF 2000:k vtR 90:

# Search pattern
qk wkF 1000:k wkL 45:k wkF 1000:k wkL 45:k wkF 1000:k wkL 45:k wkF 1000:k wkL 45:
```

## Implementation Flow

### Straight Gait Flow
1. User sends command: `k wkF 5000`
2. System parses: skill="wkF", time=5000
3. Adds task to queue: `tQueue->addTask('k', "wkF", 5000)`
4. Task queue executes gait for specified duration

### Turning Gait Flow
1. User sends command: `k wkL 90`
2. System parses: skill="wkL", angle=90
3. Sets up turning control:
   - `turningQ = true`
   - `initialYawAngle = ypr[0]`
   - `targetYawAngle = initialYawAngle + 90`
4. IMU continuously monitors yaw angle
5. When target reached: `imuException = IMU_EXCEPTION_TURNING`
6. Exception handler adds "up" task: `tQueue->addTask('k', "up")`
7. Robot stops and stands up

## Debug Output

The system provides comprehensive debug output:

**When starting turning gait**:
```
Started turning gait: wkL
initial yaw: 45.2
target angle: 135.2
turning direction: LEFT (CW)
```

**When target is reached**:
```
Turning target reached! Current yaw: 135.1
Target was: 135.2
EXCEPTION: turning target reached
endTurn
```

## Key Features

1. **Consistency** - Global ypr[0] values are correct throughout the system
2. **Simplicity** - No complex atomic operations or cross-core synchronization
3. **Maintainability** - Changes made at data source level
4. **Debug Friendly** - print6Axis() and other outputs show correct values
5. **Reliability** - Direct exception handling without complex flag management
6. **Backward Compatibility** - Commands without arguments work exactly as before
7. **Task Queue Integration** - Uses existing task queue mechanism for execution

## Notes

- All angles are normalized to the -180 to 180 degree range
- The system automatically handles angle wrapping (e.g., turning from 170° to -170°)
- Turning control is automatically disabled when the target is reached
- The robot will stop and stand up ("up" command) when the turning target is reached
- The system uses polar coordinate convention: positive angles are counterclockwise (right turn), negative angles are clockwise (left turn)
- No complex cross-core communication ensures turning exceptions are handled reliably

## Turning Exception Fix

### Problem Description

The turning exception was being skipped by the main program due to a race condition:

1. **Exception Generation**: When the turning target is reached, `getImuException()` sets `imuException = IMU_EXCEPTION_TURNING`
2. **Main Loop Timing**: The main program may have already passed `dealWithExceptions()` when the exception is generated
3. **Exception Reset**: In the next loop iteration, `getImuException()` resets `imuException = 0` before `dealWithExceptions()` can process it

### Solution

Added a `needTurning` flag to prevent the exception from being reset before it can be processed.

#### Implementation

1. **New Variable**:
   ```cpp
   bool needTurning = false;  // Flag to prevent turning exception from being skipped
   ```

2. **Exception Generation** (in `getImuException()`):
   ```cpp
   if (target_reached) {
     imuException = IMU_EXCEPTION_TURNING;
     turningQ = false;
     needTurning = true;  // Set flag to prevent exception from being skipped
   }
   ```

3. **Exception Protection** (in `getImuException()`):
   ```cpp
   } else if (!needTurning)  // Only reset exception if not waiting for turning processing
     imuException = 0;
   ```

4. **Exception Processing** (in `dealWithExceptions()`):
   ```cpp
   case IMU_EXCEPTION_TURNING:
     tQueue->addTask('k', "up");
     needTurning = false;  // Reset flag after creating task
     break;
   ```

#### Flow

1. **Turning Target Reached**: `needTurning = true`, `imuException = IMU_EXCEPTION_TURNING`
2. **Exception Protected**: `getImuException()` won't reset the exception while `needTurning = true`
3. **Exception Processed**: `dealWithExceptions()` creates the "up" task
4. **Flag Reset**: `needTurning = false` allows normal exception handling to resume

#### Benefits

- **Reliable Exception Handling**: Turning exceptions are never missed
- **Simple Implementation**: Minimal code changes
- **No Complex Synchronization**: Uses simple boolean flag
- **Maintains Existing Logic**: All other exception handling remains unchanged 
