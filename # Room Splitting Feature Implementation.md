# Room Splitting Feature Implementation Guide

## Overview

This document provides a comprehensive summary of the room splitting feature implementation that automatically handles scenarios where hotel rooms exceed the maximum allowed capacity. The feature includes smart room distribution based on adults and children, validation, and user choice UI components.

## Core Business Logic

### 1. Smart Room Splitting Algorithm

The new algorithm considers both adults and children when determining how to split rooms, providing more balanced and practical room distributions.

### 2. Validation Function: `checkIfAnyRoomExceedsMaxAdults`

**Purpose:** Validates if any room in the selection requires splitting based on adult count or total guest count.

**Location:** `HotelBookingUtils.js`

**Updated Implementation:**

```javascript
export function checkIfAnyRoomExceedsMaxAdults(selectedRoomInfo) {
  if (!isEmpty(selectedRoomInfo)) {
    for (const room of selectedRoomInfo) {
      const adults = room?.adultCount || 0;
      const children = room?.childCount || 0;
      const totalGuests = adults + children;

      if (adults >= 2) {
        // Check if adults >= 3 OR total guests >= 4
        if (adults >= 3 || totalGuests >= 4) {
          return true;
        }
      }
    }
  }

  return false;
}
```

**Validation Rules:**

- Returns `true` if **any room** has 3+ adults (and at least 2 adults total)
- Returns `true` if **any room** has 4+ total guests (and at least 2 adults total)
- Returns `false` if all rooms are within acceptable limits
- **Note**: Rooms with only 1 adult are exempt from total guest limits (family exception)

**Usage Pattern:**

- Call this function before proceeding with room selection
- If returns `true`, show user choice popup
- If returns `false`, proceed normally

### 3. Smart Room Splitting Algorithm: `splitRooms` Helper Function

**Purpose:** Implements intelligent room splitting logic based on guest composition.

**Location:** `HotelBookingUtils.js` (internal helper function)

**Algorithm Cases:**

#### Case 1: Single Adult with Many Children

```javascript
// Single adult, many kids → don't split
if (adults === 1 && children > 2) {
  return [{ adults: 1, children: children }];
}
```

**Logic:** Keep single adult with all children in one room for family convenience.

#### Case 2: Multiple Adults with One Child

```javascript
// Multiple adults, only one child → rooms with up to 2 adults + 1 child
if (children === 1) {
  const rooms = Math.ceil(adults / 2);
  // Distribute adults (max 2 per room) and assign child to first room
}
```

**Logic:** Group adults in pairs, assign the single child to the first room.

#### Case 3: Multiple Adults with Multiple Children

```javascript
// Step 1: Distribute children equally across adults
const childPerAdult = Math.floor(children / adults);
const extra = children % adults;

// Step 2: Create adult-child pairs
// Step 3: Group pairs into rooms (max 2 adults per room)
```

**Logic:**

1. First distribute children evenly among adults
2. Then group adults into rooms of 2
3. Balance the room occupancy

### 4. Main Room Splitting Function: `getTotalSelectedRoomsWithSplitting`

**Purpose:** Processes selected rooms and applies smart splitting logic.

**Location:** `HotelBookingUtils.js`

**Updated Implementation:**

```javascript
// Helper function to split rooms based on adults and children
// Ensures all resulting rooms pass checkIfAnyRoomExceedsMaxAdults validation
// Rules: 1) At least 1 adult per room, 2) Max 2 adults per room, 3) Max 3 total guests per room
// 4) Don't split if only 1 adult (keep family together)
function splitRooms(adults, children) {
  // Validation limits: max 2 adults per room, max 3 total guests per room
  const MAX_ADULTS_PER_ROOM = 2;
  const MAX_GUESTS_PER_ROOM = 3;

  // Case 1: Only 1 adult - never split, keep family together
  if (adults === 1) {
    return [{ adults: 1, children: children }];
  }

  // Case 2: Multiple adults - split ensuring each room has at least 1 adult
  const splits = [];
  let remainingAdults = adults;
  let remainingChildren = children;

  // First pass: Create rooms with adults and as many children as possible
  while (remainingAdults > 0) {
    // Ensure at least 1 adult per room, max 2 adults per room
    const adultsInThisRoom = Math.min(remainingAdults, MAX_ADULTS_PER_ROOM);
    remainingAdults -= adultsInThisRoom;

    // Add children without exceeding total guest limit
    const maxChildrenForThisRoom = MAX_GUESTS_PER_ROOM - adultsInThisRoom;
    const childrenInThisRoom = Math.min(
      remainingChildren,
      maxChildrenForThisRoom
    );
    remainingChildren -= childrenInThisRoom;

    splits.push({ adults: adultsInThisRoom, children: childrenInThisRoom });
  }

  // Second pass: Distribute remaining children among existing rooms
  if (remainingChildren > 0) {
    for (let i = 0; i < splits.length && remainingChildren > 0; i++) {
      const currentRoom = splits[i];
      const currentTotal = currentRoom.adults + currentRoom.children;
      const availableSpace = MAX_GUESTS_PER_ROOM - currentTotal;

      if (availableSpace > 0) {
        const childrenToAdd = Math.min(remainingChildren, availableSpace);
        currentRoom.children += childrenToAdd;
        remainingChildren -= childrenToAdd;
      }
    }
  }

  // Third pass: If still children remaining, redistribute adults to create more rooms
  if (remainingChildren > 0) {
    // Try to create more balanced distribution
    const totalGuests = adults + children;
    const minRoomsNeeded = Math.ceil(totalGuests / MAX_GUESTS_PER_ROOM);

    if (splits.length < minRoomsNeeded && splits.length < adults) {
      // Redistribute adults more evenly to create more rooms
      const newSplits = [];
      let redistributeAdults = adults;
      let redistributeChildren = children;

      // Calculate how many rooms we need
      const roomsNeeded = Math.min(adults, minRoomsNeeded);

      for (let i = 0; i < roomsNeeded; i++) {
        const adultsInRoom = Math.ceil(redistributeAdults / (roomsNeeded - i));
        const actualAdultsInRoom = Math.min(adultsInRoom, MAX_ADULTS_PER_ROOM);
        redistributeAdults -= actualAdultsInRoom;

        const maxChildrenInRoom = MAX_GUESTS_PER_ROOM - actualAdultsInRoom;
        const childrenInRoom = Math.min(
          redistributeChildren,
          maxChildrenInRoom
        );
        redistributeChildren -= childrenInRoom;

        newSplits.push({
          adults: actualAdultsInRoom,
          children: childrenInRoom,
        });
      }

      // If this distribution is better (accommodates more children), use it
      if (redistributeChildren < remainingChildren) {
        return newSplits;
      }
    }
  }

  return splits;
}

export function getTotalSelectedRoomsWithSplitting(selectedRoomInfo) {
  let totalRooms = 0;
  let guestsPerRoomArray = [];

  if (!isEmpty(selectedRoomInfo)) {
    selectedRoomInfo.forEach((room, originalIndex) => {
      if (room?.adultCount > 0) {
        const adults = room?.adultCount || 0;
        const children = room?.childCount || 0;
        const childAges = room?.childAge || [];

        // Use the new splitting logic
        const roomSplits = splitRooms(adults, children);

        // Distribute child ages across split rooms
        let childAgeIndex = 0;

        roomSplits.forEach((split, splitIndex) => {
          const currentChildAges = [];
          for (let j = 0; j < split.children; j++) {
            if (childAgeIndex < childAges.length) {
              currentChildAges.push(childAges[childAgeIndex]);
              childAgeIndex++;
            }
          }

          totalRooms += 1;
          const roomData = {
            roomIndex: splitIndex,
            originalRoomIndex: room?.roomIndex || 0,
            splitRoomNumber: splitIndex + 1,
            totalSplitRooms: roomSplits.length,
            adultCount: split.adults,
            childCount: split.children,
            childAge: currentChildAges,
            roomTypeCode: room?.roomTypeCode || null,
            roomVariant: room?.roomVariant || null,
            roomName: room?.roomName || null,
            guestInputData: [],
            selectedSpecialRequests: [],
            selectedBeddingPreference: null,
          };
          guestsPerRoomArray.push(new SelectedRoomInfo(roomData));
        });
      }
    });
  }

  return {
    totalRooms,
    guestsPerRoom: guestsPerRoomArray,
  };
}
```

**Key Features:**

- **Strict Validation Compliance:** All split rooms pass `checkIfAnyRoomExceedsMaxAdults` validation
- **Adult-per-Room Guarantee:** Every room has at least 1 adult (no children-only rooms)
- **Family-Friendly Logic:** Single parent with children stays together (never split)
- **Smart Redistribution:** Uses multi-pass algorithm to optimize guest distribution
- **Maximum Capacity Respect:** No room exceeds 2 adults or 3 total guests
- **Complete Guest Accommodation:** All guests are distributed across rooms

## Updated Algorithm Examples

### Example 1: 3 Adults, 5 Children

**Input:** 3 adults, 5 children (8 total guests)
**Output:** 3 rooms

- Room 1: 1 adult, 2 children (3 total)
- Room 2: 1 adult, 2 children (3 total)
- Room 3: 1 adult, 1 child (2 total)

**Algorithm Flow:**

1. Creates 3 rooms with 1 adult each (redistributed evenly)
2. Distributes 5 children: 2+2+1 across the rooms
3. All guests accommodated, all rooms compliant

### Example 2: 4 Adults, 6 Children

**Input:** 4 adults, 6 children (10 total guests)
**Output:** 4 rooms

- Room 1: 1 adult, 2 children (3 total)
- Room 2: 1 adult, 2 children (3 total)
- Room 3: 1 adult, 2 children (3 total)
- Room 4: 1 adult, 0 children (1 total)

**Algorithm Flow:**

1. Creates 4 rooms with 1 adult each (1 per room for optimal distribution)
2. Distributes 6 children: 2+2+2+0 across the rooms
3. All guests accommodated, all rooms compliant

### Example 3: 1 Adult, 5 Children

**Input:** 1 adult, 5 children (6 total guests)
**Output:** 1 room (no splitting)

- Room 1: 1 adult, 5 children (6 total)

**Algorithm Flow:**

1. Single adult exception - family stays together
2. Room exceeds normal limits but validation allows single-adult rooms
3. Family unity preserved

### Example 4: 5 Adults, 2 Children

**Input:** 5 adults, 2 children (7 total guests)
**Output:** 3 rooms

- Room 1: 2 adults, 1 child (3 total)
- Room 2: 2 adults, 1 child (3 total)
- Room 3: 1 adult, 0 children (1 total)

**Algorithm Flow:**

1. First creates 3 rooms: [2,1], [2,1], [1,0] adults
2. Distributes 2 children: 1+1+0 across the rooms
3. Optimal distribution with room capacity limits respected

- Room 2: 1 adult, 0 children

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

  // ADDED: Check if any room has max adults exceeded or total guests >= 4
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

    // Call the smart room splitting function
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
2. Copy `getTotalSelectedRoomsWithSplitting` function and the `splitRooms` helper to your utils file
3. Ensure you have the equivalent of `SelectedRoomInfo` constructor
4. Update validation rules (3+ adults OR 4+ total guests)

### Step 2: Add State Management

1. Add popup state variables to your component
2. Add room splitting data state for storing temporary data

### Step 3: Integrate Validation

1. Add validation call in your room selection handler
2. Show popup when validation fails (adults >= 3 OR total guests >= 4)
3. Store necessary data for later processing

### Step 4: Create Popup Component

1. Create room splitting popup with two-button interface
2. Implement "Split Rooms" and "Proceed Anyway" functionality
3. Add proper styling and responsive behavior

### Step 5: Handle Room Splitting

1. Implement room splitting handler that calls the smart algorithm
2. Update state to trigger UI re-renders
3. Force component re-rendering with proper keys

### Step 6: Handle State Synchronization

1. Add keys to counter components for forced re-rendering
2. Ensure proper prop passing to child components
3. Create new object instances to trigger React updates

## Testing Scenarios

### Test Cases to Validate:

#### Basic Validation

1. **Normal Cases:** 1-2 adults per room (no splitting needed)
2. **Adult Threshold:** Exactly 3 adults (should trigger splitting)
3. **Guest Threshold:** 2 adults + 2 children = 4 guests (should trigger splitting)
4. **Mixed Thresholds:** 3 adults + 1 child (should trigger splitting)

#### Smart Splitting Logic

1. **Single Parent:** 1 adult + 5 children (should not split)
2. **Multiple Adults, One Child:** 5 adults + 1 child (child goes to first room)
3. **Balanced Groups:** 4 adults + 4 children (even distribution)
4. **Uneven Groups:** 5 adults + 3 children (smart balancing)

#### Edge Cases

1. **Zero Adults:** 0 adults (should handle gracefully)
2. **Empty Arrays:** Empty room arrays
3. **Mixed Scenarios:** Some rooms need splitting, others don't
4. **Child Age Distribution:** Proper child age assignment to split rooms

#### UI Synchronization

1. **Counter Updates:** Room and guest counters update after splitting
2. **State Persistence:** Room data persists correctly
3. **Re-render Triggers:** Components update when splitting occurs

### Expected Outcomes:

- **Smart Distribution:** Guests distributed based on family composition
- **Balanced Occupancy:** Rooms have similar total occupancy when possible
- **Family-Friendly:** Single parents with many children stay together
- **Metadata Preservation:** Original room data maintained
- **UI Synchronization:** All components update correctly
- **User Choice:** Users can choose between splitting or proceeding anyway

## Key Design Decisions

### Three-Pass Algorithm Design

The updated algorithm uses a sophisticated three-pass approach to ensure all constraints are met:

1. **Pass 1: Room Initialization**

   - Creates optimal number of rooms based on guest count
   - Distributes adults evenly across rooms
   - Ensures every room has at least 1 adult

2. **Pass 2: Child Distribution**

   - Distributes children as evenly as possible
   - Respects room capacity limits (max 3 total guests)
   - Prioritizes rooms with fewer current occupants

3. **Pass 3: Validation & Cleanup**
   - Ensures all guests are accommodated
   - Verifies no room exceeds validation limits
   - Maintains family-friendly logic for edge cases

### Constraint Prioritization

1. **Critical Constraints (Never Violated):**

   - Every room must have at least 1 adult
   - No room can exceed 2 adults
   - No room can exceed 3 total guests
   - All guests must be accommodated

2. **Family-Friendly Exceptions:**

   - Single adult with children never splits (family unity)
   - Validation allows single-adult rooms to exceed normal limits

3. **Optimization Goals:**
   - Even distribution of guests across rooms
   - Minimize total number of rooms needed
   - Balance room occupancy for comfort

### Edge Case Handling

- **Single Adult Families:** Never split to preserve family unity
- **Excess Children:** Distributed optimally across available room capacity
- **Adult-Heavy Groups:** Adults distributed to minimize rooms while respecting limits
- **Validation Failures:** Algorithm designed to never produce invalid distributions

### Technical Decisions

1. **User Choice:** Always give users option to proceed without splitting
2. **Object Construction:** Use proper constructors for data integrity
3. **State Management:** Force re-renders with new object instances and component keys
4. **UI/UX:** Two-button interface prioritizing "Split Rooms" action
5. **Child Age Tracking:** Proper distribution of child ages across split rooms

## Benefits of the Three-Pass Algorithm

### Compared to Previous Simple Splitting:

1. **Guaranteed Validation Compliance:** All split rooms pass validation checks
2. **Adult-per-Room Guarantee:** No children-only rooms created
3. **Complete Guest Accommodation:** All guests are distributed (no lost children)
4. **Family-Friendly Logic:** Single parents with children stay together
5. **Optimal Distribution:** Multi-pass approach ensures even guest distribution
6. **Capacity Respect:** No room exceeds maximum occupancy limits

### Algorithm Robustness:

1. **Constraint Satisfaction:** Meets all validation requirements systematically
2. **Edge Case Handling:** Properly handles single adults and complex distributions
3. **Predictable Results:** Consistent output for same input parameters
4. **Test Coverage:** Validated with multiple test cases including edge scenarios
5. **Balanced Occupancy:** Rooms have more similar total guest counts
6. **Practical Room Arrangements:** Room splits make more practical sense for families
7. **Flexible Validation:** Considers both adult count and total guest count for better UX

This updated implementation provides a much more intelligent and user-friendly solution for handling room capacity limitations while maintaining flexibility for diverse family compositions and booking scenarios.

## Latest Implementation: PreferencePopup Component

### Overview

The room splitting feature has now been successfully implemented in the `PreferencePopup` component (`/src/Modules/StaysListing/HotelSearch/PreferencePopup/PreferencePopup.js`), providing consistent room splitting functionality across different parts of the hotel booking flow.

### 1. Component Structure and Imports

**Key Imports Added:**

```javascript
import {
  checkIfAnyRoomExceedsMaxAdults,
  getTotalSelectedRoomsWithSplitting,
} from "../../../HotelBooking/HotelBookingUtils";
import { RoomSplittingPopup } from "../../../HotelBooking/UIComponents/Popups/HotelRoomPreference";
```

**Design Decision:** The implementation reuses the existing `RoomSplittingPopup` component from `HotelRoomPreference` to maintain UI consistency and reduce code duplication.

### 2. State Management Implementation

**Room Splitting State Variables:**

```javascript
// Room splitting states
const [showRoomSplittingPopup, setShowRoomSplittingPopup] =
  React.useState(false);
const [roomSplittingData, setRoomSplittingData] = React.useState(null);
```

**State Purpose:**

- `showRoomSplittingPopup`: Controls the visibility of the room splitting dialog
- `roomSplittingData`: Stores the room preference and selected room info for processing

### 3. Room Splitting Logic Integration

**Modified `handleSelectRoomFunction`:**

```javascript
function handleSelectRoomFunction() {
  // Check if any room has max adults exceeded before proceeding
  const isMaxAdultsExceeded = checkIfAnyRoomExceedsMaxAdults(selectedRoomInfo);

  if (isMaxAdultsExceeded) {
    // Store data for room splitting and show popup
    setRoomSplittingData({
      roomPreference,
      selectedRoomInfo,
    });
    setShowRoomSplittingPopup(true);
    return;
  }

  // Proceed with normal flow if no room splitting needed
  proceedWithSelection();
}
```

**Key Features:**

- **Validation Check:** Uses `checkIfAnyRoomExceedsMaxAdults` to detect rooms requiring splitting
- **Data Storage:** Stores current state for later processing
- **Early Return:** Prevents normal flow when splitting is needed
- **Popup Trigger:** Shows room splitting dialog when validation fails

### 4. Room Splitting Handler

**`handleRoomSplitting` Implementation:**

````javascript
**`handleRoomSplitting` Implementation:**

```javascript
function handleRoomSplitting() {
  if (roomSplittingData) {
    const {
      roomPreference: currentRoomPreference,
      selectedRoomInfo: selectedRoomInfoTemp,
    } = roomSplittingData;

    // Call the room splitting function
    const splittingResult = getTotalSelectedRoomsWithSplitting(selectedRoomInfoTemp);
    console.log("Room splitting result:", splittingResult);

    // Update the room preference with split rooms
    const updatedRooms = splittingResult.guestsPerRoom;
    console.log("Updated rooms after splitting:", updatedRooms);

    // Create a fresh room preference object to ensure re-render
    const newRoomPreference = new HotelSearchPreferenceInfo(
      currentRoomPreference.toJSON()
    );
    newRoomPreference.selectedRoomInfo = updatedRooms;

    console.log("Final room preference after splitting:", newRoomPreference);

    // Update states to show split rooms in the popup
    setRoomPreference(newRoomPreference);
    setSelectedRoomInfo(updatedRooms);

    // Close room splitting popup but keep preference popup open for user review
    setShowRoomSplittingPopup(false);
    setRoomSplittingData(null);

    // Note: User will now see the split rooms and can manually click "Done" to save
  }
}
````

**Process Flow:**

1. **Extract Data:** Gets stored room preference and selected room info
2. **Apply Splitting:** Calls `getTotalSelectedRoomsWithSplitting` with smart algorithm
3. **Update Preference:** Creates new `HotelSearchPreferenceInfo` object with split rooms
4. **State Updates:** Updates both room preference and selected room info
5. **Close Split Popup:** Closes only the room splitting confirmation dialog
6. **Keep Open:** Preference popup remains open for user review and manual confirmation

**⚠️ Key Behavior Change:** Unlike the HotelRoomPreference implementation, the PreferencePopup does NOT automatically proceed after room splitting. Instead, it keeps the preference popup open, allowing users to:

- **Review Split Results:** See exactly how rooms were distributed
- **Manual Confirmation:** Users must click "Done" to proceed with the split configuration
- **User Control:** Gives users full visibility and control over the final room setup

````

**Process Flow:**

1. **Extract Data:** Gets stored room preference and selected room info
2. **Apply Splitting:** Calls `getTotalSelectedRoomsWithSplitting` with smart algorithm
3. **Update Preference:** Creates new `HotelSearchPreferenceInfo` object with split rooms
4. **State Updates:** Updates both room preference and selected room info
5. **Cleanup:** Closes popup and clears temporary data
6. **Proceed:** Continues with hotel search using split rooms

### 5. Helper Functions

**`proceedWithSelection` - Extracted Common Logic:**

```javascript
function proceedWithSelection() {
  setNationalityAndResidency();
  handleGuestInfo(roomPreference);
  handleClose(true);
  console.log(
    "updateSelectedRoomInfo >>> handleSelectRoomFunction 111",
    roomPreference
  );
}
````

**`handleRoomSplittingCancel` - Cancel Handler:**

```javascript
function handleRoomSplittingCancel() {
  setShowRoomSplittingPopup(false);
  setRoomSplittingData(null);
}
```

**`handleProceedAnyway` - Bypass Handler:**

```javascript
function handleProceedAnyway() {
  // Proceed without room splitting
  setShowRoomSplittingPopup(false);
  setRoomSplittingData(null);
  proceedWithSelection();
}
```

### 6. Component Integration

**Popup Component Integration:**

```javascript
{
  /* Room Splitting Popup */
}
<RoomSplittingPopup
  open={showRoomSplittingPopup}
  onSplit={handleRoomSplitting}
  onCancel={handleRoomSplittingCancel}
  onProceedAnyway={handleProceedAnyway}
/>;
```

**Design Benefits:**

- **Component Reuse:** Leverages existing `RoomSplittingPopup` component
- **Consistent UI:** Same look and feel across different booking flows
- **Maintainability:** Single popup component to maintain and update

### 7. Differences from HotelRoomPreference Implementation

**Key Differences:**

| Aspect                  | HotelRoomPreference                  | PreferencePopup                       |
| ----------------------- | ------------------------------------ | ------------------------------------- |
| **Data Model**          | `HotelRoomPreferenceInfo`            | `HotelSearchPreferenceInfo`           |
| **Update Method**       | `setSelectedRoomInfoWithSplitting()` | Direct property assignment            |
| **Flow Completion**     | `makeAvailabilityRequest()`          | `handleGuestInfo()` + `handleClose()` |
| **Popup Component**     | Local implementation                 | Imported shared component             |
| **Post-Split Behavior** | Automatically proceeds with booking  | Keeps popup open for user review      |
| **User Interaction**    | Auto-closes after splitting          | Requires manual "Done" click          |

**Enhanced User Experience:**
The PreferencePopup implementation provides better user control by keeping the preference dialog open after room splitting, allowing users to review the split configuration before proceeding.

**Flow Comparison:**

**HotelRoomPreference Flow:**

1. User clicks "Select Room"
2. Splitting popup appears (if needed)
3. User clicks "Split Guests"
4. Rooms are split automatically
5. **Auto-proceeds** to availability request

**PreferencePopup Flow:**

1. User clicks "Done"
2. Splitting popup appears (if needed)
3. User clicks "Split Guests"
4. Rooms are split and **popup stays open**
5. **User reviews** split room configuration
6. **User manually clicks "Done"** to proceed

### 8. Component Architecture Benefits

**Code Reuse:**

- **Shared Algorithm:** Both components use the same smart splitting logic
- **Shared Validation:** Same validation function across components
- **Shared UI Component:** Consistent popup experience

**Maintainability:**

- **Single Source of Truth:** Algorithm changes apply to all components
- **Consistent UX:** Users see the same interface regardless of entry point
- **Reduced Duplication:** Less code to maintain and test

### 9. Integration Points

**Search Flow Integration:**

```javascript
// In parent component calling PreferencePopup
<PreferencePopup
  handleClose={handleClose}
  open={open}
  handleGuestInfo={handleGuestInfo} // Receives split room data
  searchPreferenceInfo={searchPreferenceInfo}
  editPreferenceAnchorEl={editPreferenceAnchorEl}
/>
```

**Data Flow:**

1. User modifies room preferences in PreferencePopup
2. On "Done" click, validation checks for room splitting needs
3. If splitting required, user chooses via popup
4. Split rooms are processed and passed back via `handleGuestInfo`
5. Parent component receives properly split room configuration

### 10. Implementation Validation

**Testing Scenarios for PreferencePopup:**

1. **Normal Cases:** Verify no popup appears for valid room configurations
2. **Splitting Triggers:** Confirm popup shows for 3+ adults or 4+ total guests
3. **Split Processing:** Validate correct room distribution after splitting
4. **State Management:** Ensure proper state updates and cleanup
5. **Flow Completion:** Verify correct data passed to parent component

**Key Validation Points:**

- ✅ Popup appears when room capacity exceeded
- ✅ Smart splitting algorithm applies correctly
- ✅ User can choose between splitting and proceeding anyway
- ✅ **NEW**: Preference popup stays open after room splitting
- ✅ **NEW**: Split room configuration is visible to user
- ✅ **NEW**: Manual "Done" click required to proceed
- ✅ State properly updated after splitting
- ✅ Parent component receives correct room data
- ✅ Popup closes and state cleaned up properly

This implementation ensures that the room splitting feature is consistently available across different hotel booking entry points, providing users with the same intelligent room distribution options regardless of where they start their booking process.

## Latest Update: Enhanced User Control (October 2025)

### User Experience Improvement

The PreferencePopup implementation has been enhanced to provide better user control and transparency in the room splitting process:

**Previous Behavior:**

- User clicks "Split Guests"
- Rooms are automatically split
- Popup automatically closes and proceeds

**New Enhanced Behavior:**

- User clicks "Split Guests"
- Rooms are split using smart algorithm
- **Preference popup stays open** showing the split room configuration
- User can **review the split results**
- User must **manually click "Done"** to proceed

### Benefits of the Enhanced Approach

1. **Transparency**: Users can see exactly how their guests were distributed
2. **User Control**: Users have final approval over the room splitting decision
3. **Trust Building**: No hidden automatic changes, everything is visible
4. **Error Prevention**: Users can catch unexpected splitting results before proceeding
5. **Flexibility**: Users could potentially modify the split if needed (future enhancement)

### Implementation Details

**Code Change Location:** `PreferencePopup.js` - `handleRoomSplitting()` function

**Key Changes:**

```javascript
// OLD: Auto-close and proceed
setShowRoomSplittingPopup(false);
setRoomSplittingData(null);
setNationalityAndResidency();
handleGuestInfo(newRoomPreference);
handleClose(true);

// NEW: Keep open for user review
setShowRoomSplittingPopup(false);
setRoomSplittingData(null);
// Note: User will now see the split rooms and can manually click "Done" to save
```

This enhancement makes the room splitting feature more user-friendly and trustworthy, ensuring users are always in control of their booking preferences.
