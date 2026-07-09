# Firestore Security Specifications & TDD Spec

## 1. Data Invariants
For any document stored in the `/bookings` collection:
- **ID Integrity**: The document ID must be a valid string matching `^[a-zA-Z0-9_\-]+$` and have a length <= 128 characters.
- **Name Verification**: The passenger name must be a string, non-empty, and limited to a maximum length of 100 characters.
- **Contact Validation**: The mobile number must be exactly a 10-digit string containing only numeric characters.
- **Route & Package Validation**: `packageName` must be a non-empty string <= 100 characters.
- **Temporal Integrity**: `timestamp` must be a valid ISO string. During creation, it must be validated.
- **Capacity Integrity**: `passengers` must be a number between 1 and 7 (maximum capacity of Maruti Ertiga).
- **Seat Safety**: If `seats` are present:
  - It must be a List/Array.
  - Its size must be between 1 and 7.
  - Seat values must be integers between 1 and 7.
- **Immutable Fields**: Once a booking is created, its `id`, `date`, and `seats` (if any) are immutable and cannot be altered by normal users.

---

## 2. The "Dirty Dozen" Malicious Payloads
The following payloads must be explicitly rejected by the Firestore Security Rules.

### Payload 1: Resource Poisoning via Invalid ID
- **Target**: `/bookings/Poison-ID-$$$-!!!`
- **Violation**: Document ID contains non-alphanumeric/invalid characters.

### Payload 2: Empty Passenger Name (No name)
- **Target**: `/bookings/b-1`
- **Violation**: `name` is empty or missing.
- **Payload**: `{"id": "b-1", "mobile": "9876543210", "packageName": "Ghazipur → Lucknow", "date": "2026-07-08", "timestamp": "2026-07-08T07:45:23.000Z"}`

### Payload 3: Invalid Mobile Format (Too long)
- **Target**: `/bookings/b-2`
- **Violation**: `mobile` is 12 digits.
- **Payload**: `{"id": "b-2", "name": "Ramesh Kumar", "mobile": "987654321012", "packageName": "Ghazipur → Lucknow", "date": "2026-07-08", "timestamp": "2026-07-08T07:45:23.000Z"}`

### Payload 4: Invalid Mobile Characters
- **Target**: `/bookings/b-3`
- **Violation**: `mobile` contains letters.
- **Payload**: `{"id": "b-3", "name": "Ramesh Kumar", "mobile": "9876543abc", "packageName": "Ghazipur → Lucknow", "date": "2026-07-08", "timestamp": "2026-07-08T07:45:23.000Z"}`

### Payload 5: Missing Package Name
- **Target**: `/bookings/b-4`
- **Violation**: `packageName` is missing.
- **Payload**: `{"id": "b-4", "name": "Ramesh Kumar", "mobile": "9876543210", "date": "2026-07-08", "timestamp": "2026-07-08T07:45:23.000Z"}`

### Payload 6: Overflow Passengers Count
- **Target**: `/bookings/b-5`
- **Violation**: `passengers` exceeds vehicle seating capacity (e.g., 100 passengers).
- **Payload**: `{"id": "b-5", "name": "Ramesh Kumar", "mobile": "9876543210", "packageName": "Ghazipur → Lucknow", "passengers": 100, "date": "2026-07-08", "timestamp": "2026-07-08T07:45:23.000Z"}`

### Payload 7: Negative Passengers Count
- **Target**: `/bookings/b-6`
- **Violation**: `passengers` count is negative.
- **Payload**: `{"id": "b-6", "name": "Ramesh Kumar", "mobile": "9876543210", "packageName": "Ghazipur → Lucknow", "passengers": -2, "date": "2026-07-08", "timestamp": "2026-07-08T07:45:23.000Z"}`

### Payload 8: Invalid Seat Value (Out of Bounds)
- **Target**: `/bookings/b-7`
- **Violation**: Seats array contains seat 0 or seat 8.
- **Payload**: `{"id": "b-7", "name": "Ramesh Kumar", "mobile": "9876543210", "packageName": "Ghazipur → Lucknow", "date": "2026-07-08", "seats": [0, 8], "timestamp": "2026-07-08T07:45:23.000Z"}`

### Payload 9: Invalid Date Format
- **Target**: `/bookings/b-8`
- **Violation**: `date` is not in YYYY-MM-DD format.
- **Payload**: `{"id": "b-8", "name": "Ramesh", "mobile": "9876543210", "packageName": "Lucknow", "date": "08-07-2026", "timestamp": "2026-07-08T07:45:23.000Z"}`

### Payload 10: Unauthorized Document Deletion (Non-Admin)
- **Target**: `/bookings/b-9`
- **Violation**: Normal users are blocked from deleting any booking from the client; deletions must be performed via authenticated Admin flows.

### Payload 11: Attempting to Update Immutable Date
- **Target**: `/bookings/b-10`
- **Violation**: Modifying the journey `date` on an existing booking.

### Payload 12: Attempting to Modify Blocked Seats on Existing Bookings
- **Target**: `/bookings/b-11`
- **Violation**: Altering seat allocations without authentication.

---

## 3. Test Runner Definition
A formal local unit test suite `firestore.rules.test.ts` should be verified conceptually by our strict rules layout.
All "Dirty Dozen" payloads must be securely rejected with `PERMISSION_DENIED`.
