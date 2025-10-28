# Room Splitting Feature Implementation Guide

## Overview

This document provides a comprehensive summary of the room splitting feature implementation that automatically handles scenarios where hotel rooms exceed the maximum allowed adults per room (currently set to 3). The feature includes validation, automatic room distribution, and user choice UI components.

## Core Business Logic

### 1. Constants and Configuration

```javascript
const MAX_ADULTS_PER_ROOM = 3;
```

### 2. Validation Function: `checkIfAnyRoomExceedsMaxAdults`

**Purpose:** Validates if any room in the selection exceeds the maximum adult capacity.

**Location:** `HotelBookingUtils.js`

**Implementation:**

```javascript
export function checkIfAnyRoomExceedsMaxAdults(selectedRoomInfo) {
  const MAX_ADULTS_PER_ROOM = 3;

  if (!isEmpty(selectedRoomInfo)) {
    for (const room of selectedRoomInfo) {
      if (room?.adultCount > MAX_ADULTS_PER_ROOM) {
        return true;
      }
    }
  }

  return false;
}
```

**Usage Pattern:**

- Call this function before proceeding with room selection
- If returns `true`, show user choice popup
- If returns `false`, proceed normally

### 3. Room Splitting Algorithm: `getTotalSelectedRoomsWithSplitting`

**Purpose:** Automatically distributes guests across multiple rooms when exceeding capacity.

**Location:** `HotelBookingUtils.js`

**Key Features:**

- Splits rooms when adults > 3
- Distributes adults evenly across new rooms
- Places all children in the first room for simplicity
- Preserves original room metadata
- Uses `SelectedRoomInfo` constructor for proper object instantiation

**Implementation:**

```javascript
export function getTotalSelectedRoomsWithSplitting(selectedRoomInfo) {
  let totalRooms = 0;
  let guestsPerRoomArray = [];
  const MAX_ADULTS_PER_ROOM = 3;

  if (!isEmpty(selectedRoomInfo)) {
    selectedRoomInfo.forEach((room, originalIndex) => {
      if (room?.adultCount > 0) {
        const adults = room?.adultCount || 0;
        const children = room?.childCount || 0;
        const childAges = room?.childAge || [];

        // If adults <= 3, keep as single room
        if (adults <= MAX_ADULTS_PER_ROOM) {
          totalRooms += 1;
          const roomData = {
            roomIndex: room?.roomIndex || 0,
            originalRoomIndex: room?.roomIndex || 0,
            splitRoomNumber: 1,
            totalSplitRooms: 1,
            adultCount: adults,
            childCount: children,
            childAge: childAges,
            roomTypeCode: room?.roomTypeCode || null,
            roomVariant: room?.roomVariant || null,
            roomName: room?.roomName || null,
            guestInputData: [],
            selectedSpecialRequests: [],
            selectedBeddingPreference: null,
          };
          guestsPerRoomArray.push(new SelectedRoomInfo(roomData));
        } else {
          // Split room if adults > 3
          const numberOfRooms = Math.ceil(adults / MAX_ADULTS_PER_ROOM);
          const adultsPerRoom = Math.floor(adults / numberOfRooms);
          const remainingAdults = adults % numberOfRooms;

          // Distribute children across rooms (put all children in first room for simplicity)
          for (let i = 0; i < numberOfRooms; i++) {
            const currentRoomAdults =
              adultsPerRoom + (i < remainingAdults ? 1 : 0);
            const currentRoomChildren = i === 0 ? children : 0;
            const currentChildAges = i === 0 ? childAges : [];

            totalRooms += 1;
            const roomData = {
              roomIndex: i,
              originalRoomIndex: room.roomIndex,
              splitRoomNumber: i + 1,
              totalSplitRooms: numberOfRooms,
              adultCount: currentRoomAdults,
              childCount: currentRoomChildren,
              childAge: currentChildAges,
              roomTypeCode: room?.roomTypeCode || null,
              roomVariant: room?.roomVariant || null,
              roomName: room?.roomName || null,
              guestInputData: [],
              selectedSpecialRequests: [],
              selectedBeddingPreference: null,
            };
            guestsPerRoomArray.push(new SelectedRoomInfo(roomData));
          }
        }
      }
    });
  }

  return {
    totalRooms,
    guestsPerRoom: guestsPerRoomArray,
  };
}
```

**Algorithm Details:**

- **Even Distribution:** `Math.ceil(adults / MAX_ADULTS_PER_ROOM)` calculates required rooms
- **Remainder Handling:** Extra adults are distributed to first few rooms
- **Child Placement:** All children go to the first split room
- **Metadata Preservation:** Original room properties are maintained across split rooms

## UI Components and State Management

### 1. Required State Variables

**Location:** `HotelRoomPreference.js`

```javascript
const [showRoomSplittingPopup, setShowRoomSplittingPopup] =
  React.useState(false);
const [roomSplittingData, setRoomSplittingData] = React.useState(null);
```

### 2. Main Handler Function Integration

**Function:** `handleSelectRoomFunction`

**Changes Made:**

```javascript
function handleSelectRoomFunction() {
  let defaultRoomPreference = roomPreference.clearAllRoomSelection();
  defaultRoomPreference.resetFilters();
  defaultRoomPreference.resetSelectedRoomFilter();

  let selectedRoomInfoTemp = defaultRoomPreference.getSelectedRoomInfo();

  // ADDED: Check if any room has max adults exceeded
  const isMaxAdultsExceeded =
    checkIfAnyRoomExceedsMaxAdults(selectedRoomInfoTemp);
  if (isMaxAdultsExceeded) {
    // Store data for room splitting and show popup
    setRoomSplittingData({
      defaultRoomPreference,
      selectedRoomInfoTemp,
    });
    setShowRoomSplittingPopup(true);
    return; // Exit here to show popup instead of proceeding
  }

  // Original flow continues if no splitting needed
  setSavingPreferenceInProgress(true);

  setTimeout(() => {
    setSavingPreferenceInProgress(false);
    if (setRetryAvailRequestCount !== null) {
      setRetryAvailRequestCount(1);
    }
    makeAvailabilityRequest(
      defaultRoomPreference,
      true,
      null,
      true,
      false,
      true
    );
    handleCloseIntent(true);
  }, 2000);
}
```

### 3. Room Splitting Handler

**Function:** `handleRoomSplitting`

```javascript
function handleRoomSplitting() {
  if (roomSplittingData) {
    const { defaultRoomPreference, selectedRoomInfoTemp } = roomSplittingData;

    // Call the room splitting function
    const splittingResult =
      getTotalSelectedRoomsWithSplitting(selectedRoomInfoTemp);
    console.log("Room splitting result:", splittingResult);

    // Update the room preference with split rooms
    const updatedRooms = splittingResult.guestsPerRoom;
    console.log("Updated rooms after splitting:", updatedRooms);

    // Create a fresh room preference object to ensure re-render
    const newRoomPreference = new HotelRoomPreferenceInfo(
      defaultRoomPreference.toJSON()
    );
    newRoomPreference.setSelectedRoomInfoWithSplitting(updatedRooms);

    // Force state updates to trigger re-render of all components
    setSelectedRoomInfo([...updatedRooms]); // Create new array reference
    setRoomPreference(newRoomPreference); // Set new object instance

    // Close the popup
    setShowRoomSplittingPopup(false);
    setRoomSplittingData(null);
  }
}
```

### 4. Proceed Anyway Handler

**Function:** `handleProceedAnyway`

```javascript
function handleProceedAnyway() {
  if (roomSplittingData) {
    const { defaultRoomPreference } = roomSplittingData;

    // Proceed with the original room configuration without splitting
    console.log("User chose to proceed anyway with original configuration");

    // preference saving loader.
    setSavingPreferenceInProgress(true);

    setTimeout(() => {
      setSavingPreferenceInProgress(false);
      if (setRetryAvailRequestCount !== null) {
        setRetryAvailRequestCount(1);
      }
      makeAvailabilityRequest(
        defaultRoomPreference,
        true,
        null,
        true,
        false,
        true
      );
      handleCloseIntent(true);
    }, 2000);

    // Close the popup
    setShowRoomSplittingPopup(false);
    setRoomSplittingData(null);
  }
}
```

### 5. Popup Component: `RoomSplittingPopup`

**Location:** Bottom of `HotelRoomPreference.js`

**Features:**

- Two-button interface: "Split Rooms" and "Proceed Anyway"
- Centered dialog with 400px max width
- Responsive design with Material-UI
- Clear messaging about room capacity limitations

```javascript
function RoomSplittingPopup({ open, onSplit, onCancel, onProceedAnyway }) {
  const theme = useTheme();
  const fullScreen = useMediaQueryMUI(theme.breakpoints.down("sm"));

  return (
    <BootstrapDialog
      fullScreen={false}
      open={open}
      onClose={onCancel}
      aria-labelledby="room-splitting-dialog-title"
      PaperProps={{
        style: {
          maxWidth: "400px",
          width: "100%",
          margin: "auto",
        },
      }}
      sx={{
        "& .MuiDialog-container": {
          alignItems: "center",
          justifyContent: "center",
        },
      }}
    >
      <DialogTitle sx={{ m: 0, p: 2 }} id="room-splitting-dialog-title">
        <p className="title2">Room Capacity Exceeded</p>
      </DialogTitle>
      <IconButton
        color="inherit"
        sx={{
          position: "absolute",
          right: 8,
          top: 8,
          color: (theme) => theme.palette.grey[500],
        }}
        onClick={onCancel}
        aria-label="close"
      >
        <img src={itemDetailAssets.close_icon} alt=""></img>
      </IconButton>

      <DialogContent
        style={{
          padding: "0px 20px",
          overflowX: "hidden",
          minWidth: "350px",
        }}
      >
        <div style={{ padding: "16px 0" }}>
          <p
            className="body1"
            style={{ color: "var(--gray70)", marginBottom: "24px" }}
          >
            You may not see all room options. Try splitting guests across
            multiple rooms.
          </p>
        </div>
      </DialogContent>

      <DialogActions style={{ padding: "16px 20px" }}>
        <div style={{ width: "100%" }}>
          <button
            className="CTAButton__secondary"
            onClick={onProceedAnyway}
            style={{ flex: 1, width: "100%", marginBottom: "8px" }}
          >
            Proceed Anyway
          </button>
          <button
            className="CTAButton"
            onClick={onSplit}
            style={{ flex: 1, width: "100%" }}
          >
            Split Rooms
          </button>
        </div>
      </DialogActions>
    </BootstrapDialog>
  );
}
```

## Component State Synchronization

### 1. Counter Component Updates

**Key Issue:** UI components need to re-render when room data changes after splitting.

**Solution:** Added keys to force re-rendering:

```javascript
// Room Counter
<RoomCounterPreference
  key={selectedRoomInfo?.length} // Force re-render when room count changes
  roomPreference={roomPreference}
  setRoomPreference={setRoomPreference}
  setSelectedRoomInfo={setSelectedRoomInfo}
  selectedRoomInfo={selectedRoomInfo}
  maxAllowedBooking={roomPreference?.maxAllowedBooking}
/>;

// Guest Counter
{
  !isEmpty(selectedRoomInfo) &&
    selectedRoomInfo.map((roomInfo, roomIndex) => {
      return (
        <GuestCounter
          key={`room-${roomIndex}-${roomInfo.adultCount}-${roomInfo.childCount}`} // Force re-render when guest counts change
          roomInfo={roomInfo}
          roomIndex={roomIndex}
          updateSelectedRoomInfo={updateSelectedRoomInfo}
          roomCount={selectedRoomInfo?.length}
        />
      );
    });
}
```

### 2. Prop Passing Pattern

**Required Props for Both Desktop and Mobile Components:**

```javascript
showRoomSplittingPopup = { showRoomSplittingPopup };
handleRoomSplitting = { handleRoomSplitting };
handleRoomSplittingCancel = { handleRoomSplittingCancel };
handleProceedAnyway = { handleProceedAnyway };
```

## Required Imports

**HotelBookingUtils.js:**

```javascript
import { SelectedRoomInfo } from "./Models/SelectedRoomInfo";
```

**HotelRoomPreference.js:**

```javascript
import {
  checkIfAnyRoomExceedsMaxAdults,
  getTotalSelectedRoomsWithSplitting,
} from "../../HotelBookingUtils";
```

## Implementation Steps for Other Clients

### Step 1: Add Utility Functions

1. Copy `checkIfAnyRoomExceedsMaxAdults` function to your utils file
2. Copy `getTotalSelectedRoomsWithSplitting` function to your utils file
3. Ensure you have the equivalent of `SelectedRoomInfo` constructor
4. Set your `MAX_ADULTS_PER_ROOM` constant

### Step 2: Add State Management

1. Add popup state variables to your component
2. Add room splitting data state for storing temporary data

### Step 3: Integrate Validation

1. Add validation call in your room selection handler
2. Show popup when validation fails
3. Store necessary data for later processing

### Step 4: Create Popup Component

1. Create room splitting popup with two-button interface
2. Implement "Split Rooms" and "Proceed Anyway" functionality
3. Add proper styling and responsive behavior

### Step 5: Handle Room Splitting

1. Implement room splitting handler that calls the algorithm
2. Update state to trigger UI re-renders
3. Force component re-rendering with proper keys

### Step 6: Handle State Synchronization

1. Add keys to counter components for forced re-rendering
2. Ensure proper prop passing to child components
3. Create new object instances to trigger React updates

## Testing Scenarios

### Test Cases to Validate:

1. **Normal Cases:** 1-3 adults per room (no splitting needed)
2. **Split Scenarios:** 4-12 adults requiring multiple rooms
3. **Mixed Scenarios:** Some rooms need splitting, others don't
4. **Edge Cases:** 0 adults, empty arrays
5. **Children Handling:** Children placement in first room
6. **UI Updates:** Counter synchronization after splitting

### Expected Outcomes:

- Adults distributed evenly across rooms (max 3 per room)
- Children consolidated in first split room
- Original room metadata preserved
- UI components update correctly
- User can choose between splitting or proceeding anyway

## Key Design Decisions

1. **Maximum Adults Per Room:** Set to 3 (configurable constant)
2. **Child Distribution:** All children go to first split room for simplicity
3. **User Choice:** Always give users option to proceed without splitting
4. **Object Construction:** Use proper constructors for data integrity
5. **State Management:** Force re-renders with new object instances and component keys
6. **UI/UX:** Two-button interface prioritizing "Split Rooms" action

This implementation provides a robust, user-friendly solution for handling room capacity limitations while maintaining flexibility for different booking scenarios.
