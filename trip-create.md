# Create Trip API Documentation

## Table of Contents
1. [Overview](#overview)
2. [API Endpoint](#api-endpoint)
3. [Authentication](#authentication)
4. [Request Specification](#request-specification)
5. [Response Specification](#response-specification)
6. [Validation Rules](#validation-rules)
7. [Sample Requests](#sample-requests)
8. [Error Handling](#error-handling)
9. [Important Notes](#important-notes)
10. [Testing Checklist](#testing-checklist)

---

## Overview

The Create Trip API enables external systems to programmatically create trip records for employees within the DICE platform. This API supports comprehensive trip management including origin, destination, intermediate cities, and return journey configurations.

### Supported Trip Types

1. **Round Trip**
   - Trip originates from a city and returns to the same origin city
   - Requires `return` object in the request payload

2. **Multi-City with Round Trip**
   - Trip visits multiple cities and returns to the origin
   - Supports complex itineraries with different travel and stay types

3. **End Trip Here**
   - Trip ends at the last visited city without returning to origin
   - Requires `endTripHere: true` flag in the request payload

---

## API Endpoint

### HTTP Method
```
POST
```

### Endpoint Path
```
/apis/external/trips/create
```

### Base URLs

| Environment | Base URL |
|------------|----------|
| **Production** | `https://heimdall.eka.io` |
| **Pre-Production** | `https://dice-uat.eka.io` |
| **Stage** | `https://api-uat-sandbox.eka.io` |

### Complete Endpoint URLs

- **Production**: `https://heimdall.eka.io/apis/external/trips/create`
- **Pre-Production**: `https://dice-uat.eka.io/apis/external/trips/create`
- **Stage**: `https://api-uat-sandbox.eka.io/apis/external/trips/create`

---

## Authentication

All API requests require authentication using custom headers. The following headers must be included in every request.

### Required Headers

| Header Name | Type | Required | Description | Example Value |
|------------|------|----------|-------------|---------------|
| `DICE-APP-ID` | String | Yes | Application identifier for the client | `BRLPS` |
| `X-CLIENT-ID` | String | Yes | Unique client identifier | `52afccbdc` |
| `X-CLIENT-SECRET` | String | Yes | Client secret key for authentication | `52e6c69fb8204387a2295628aa24b405` |
| `X-Forwarded-For` | String | Yes | Client IP address | `13.234.5.184` |
| `Content-Type` | String | Yes | Request content type | `application/json` |

### Example Request Headers

```
DICE-APP-ID: BRLPS
X-CLIENT-ID: 52afccbdc
X-CLIENT-SECRET: 52e6c69fb8204387a2295628aa24b405
X-Forwarded-For: 13.234.5.184
Content-Type: application/json
```

---

## Request Specification

### Request Body Structure

The request body is a JSON object containing trip information, employee details, and approval workflow data.

### Root Level Fields

| Field | Data Type | Required | Description |
|-------|-----------|----------|-------------|
| `tripId` | String | **Yes** | Unique trip identifier. Must follow the format: `TRIP-{PREFIX}-{NUMBER}`. Example: `TRIP-BRLPS-0007` |
| `employeeCode` | String | **Yes** | Employee code to associate the trip with. If the employee does not exist in the system, `employeeData` must be provided |
| `tripStatus` | String | **Yes** | Current status of the trip. Valid values: `PENDING`, `APPROVED`, `ACTIVE`, `COMPLETED`, etc. |
| `tripStartDate` | Long (Integer) | **Yes** | Trip start timestamp in milliseconds (epoch time). Must match `data.origin.dateTime` |
| `tripEndDate` | Long (Integer) | **Yes** | Trip end timestamp in milliseconds (epoch time). Must match `data.return.dateTime` (round trips) or last city's `endTime` (endTripHere trips) |
| `data` | Object | **Yes** | Trip data object containing origin, cities, and return information (see Data Object section) |
| `approvers` | Object | **Yes** | Approvers information object (see Approvers Object section) |
| `employeeData` | Object | Conditional | Employee data object. **Required** if employee does not exist in the system. Used to create a new employee record |

---

### Data Object (`data`)

The `data` object contains the core trip itinerary information.

| Field | Data Type | Required | Description |
|-------|-----------|----------|-------------|
| `allowance` | Long (Integer) | **Yes** | Allowance amount in the base currency. Send `0` if no allowance is applicable. This field is mandatory |
| `title` | String | **Yes** | Trip title or description. Used for identification and reporting purposes |
| `origin` | Object | **Yes** | Origin city information object (see Origin Object section) |
| `cities` | Array | Conditional | Array of city objects representing intermediate destinations. Required if the trip includes intermediate cities between origin and return |
| `return` | Object | Conditional | Return city information object. **Required for round trips**. Must not be present if `endTripHere` is `true` |
| `endTripHere` | Boolean | Conditional | Set to `true` if the trip ends at the last city without returning to origin. **Required if no return city is specified** |

---

### Origin Object (`data.origin`)

Defines the trip's starting point.

| Field | Data Type | Required | Description |
|-------|-----------|----------|-------------|
| `cityName` | String | **Yes** | Name of the origin city. Must match exactly with city names in the DICE system |
| `dateTime` | Long (Integer) | **Yes** | Timestamp when the trip starts, in milliseconds (epoch time). Must equal `tripStartDate` at the root level |
| `departureTime` | Long (Integer) | Optional | Departure timestamp in milliseconds. Recommended for flight travel. If provided, should equal `dateTime` |

**Validation Rules:**
- `origin.dateTime` must equal `tripStartDate` (root level)
- `origin.departureTime` should equal `origin.dateTime` (if provided)
- `origin.dateTime` must not be greater than the first city's `startTime` (if cities are present)

---

### City Object (`data.cities[]`)

Represents an intermediate destination in the trip itinerary.

| Field | Data Type | Required | Description |
|-------|-----------|----------|-------------|
| `cityName` | String | **Yes** | Name of the city. Must match exactly with city names in the DICE system |
| `startTime` | Long (Integer) | **Yes** | Timestamp when the employee arrives at this city, in milliseconds (epoch time) |
| `endTime` | Long (Integer) | **Yes** | Timestamp when the employee departs from this city, in milliseconds (epoch time) |
| `arrivalTime` | Long (Integer) | Conditional | Arrival timestamp in milliseconds. **Required if traveling by flight**. Should equal `startTime` |
| `departureTime` | Long (Integer) | Conditional | Departure timestamp in milliseconds. **Required if traveling by flight**. Should equal `endTime` |
| `stay` | Object | **Yes** | Stay information object (see Stay Object section) |
| `travel` | Object | **Yes** | Travel information object (see Travel Object section) |

**Validation Rules:**
- `startTime` must equal `arrivalTime` (if both are provided)
- `endTime` must equal `departureTime` (if both are provided)
- `startTime` must not be greater than `endTime`
- Previous city's `endTime` must not be greater than current city's `startTime`
- For the last city in `endTripHere` trips: `endTime` must equal `tripEndDate`

---

### Return Object (`data.return`)

Defines the return journey destination (for round trips only).

| Field | Data Type | Required | Description |
|-------|-----------|----------|-------------|
| `cityName` | String | **Yes** | Name of the return city. Typically matches the origin city for round trips |
| `dateTime` | Long (Integer) | **Yes** | Timestamp when the employee returns, in milliseconds (epoch time). Must equal `tripEndDate` at the root level |
| `arrivalTime` | Long (Integer) | Conditional | Arrival timestamp in milliseconds. **Required if traveling by flight**. Should equal `dateTime` |
| `stay` | Object | Optional | Stay information object. Can be an empty object `{}` if no stay is required |
| `travel` | Object | Optional | Travel information object (see Travel Object section) |

**Validation Rules:**
- `return.dateTime` must equal `tripEndDate` (root level)
- `return.arrivalTime` should equal `return.dateTime` (if provided)

---

### Travel Object

Specifies the mode of transportation between cities.

| Field | Data Type | Required | Description |
|-------|-----------|----------|-------------|
| `type` | String | Optional | Travel type. Valid values (case-insensitive): `flight`, `train`, `bus`, `cab`, `own`, `none` |

**Supported Travel Types:**
- `flight` - Air travel
- `train` - Train travel
- `bus` - Bus travel
- `cab` - Taxi/Cab service
- `own` - Personal vehicle
- `none` - No travel required

---

### Stay Object

Specifies accommodation details for a city.

| Field | Data Type | Required | Description |
|-------|-----------|----------|-------------|
| `type` | String | Optional | Stay type. Valid values (case-insensitive): `none`, `hotel` |

**Supported Stay Types:**
- `none` - No accommodation required
- `hotel` - Hotel accommodation

---

### Approvers Object (`approvers`)

Contains the list of approvers for the trip.

| Field | Data Type | Required | Description |
|-------|-----------|----------|-------------|
| `tripApprovers` | Array | **Yes** | Array of approver objects (see TripApprover Object section) |

---

### TripApprover Object (`approvers.tripApprovers[]`)

Represents an individual approver in the approval workflow.

| Field | Data Type | Required | Description |
|-------|-----------|----------|-------------|
| `name` | String | **Yes** | Approver's full name |
| `email` | String | **Yes** | Approver's email address |
| `code` | String | **Yes** | Approver's employee code. Must exist in the DICE system |
| `status` | String | **Yes** | Approval status. Valid values: `APPROVED`, `PENDING`, `REJECTED` |
| `approvalTimestamp` | Long (Integer) | **Yes** | Approval timestamp in milliseconds (epoch time) |

---

### EmployeeData Object (`employeeData`)

Employee information used to create a new employee record if the employee does not exist.

| Field | Data Type | Required | Description |
|-------|-----------|----------|-------------|
| `firstName` | String | **Yes** | Employee's first name |
| `lastName` | String | **Yes** | Employee's last name |
| `email` | String | **Yes** | Employee's email address |
| `mobile` | String | **Yes** | Employee's mobile number |
| `code` | String | **Yes** | Employee's code. Must match `employeeCode` at the root level |
| `department` | String | Optional | Employee's department name |
| `office` | String | Optional | Employee's office location |
| `gender` | String | Optional | Employee's gender |
| `customFields` | Object | Optional | Custom fields as key-value pairs for additional employee attributes |

**Important Note:** This object is required if the employee specified by `employeeCode` does not exist in the system. The system will automatically create a new employee record using this data.

---

## Response Specification

### Success Response

**HTTP Status Code:** `200 OK`

**Response Body:**
```json
{
    "success": true,
    "message": "Success",
    "tripId": "TRIP-BRLPS-0007"
}
```

**Response Fields:**

| Field | Data Type | Description |
|-------|-----------|-------------|
| `success` | Boolean | Indicates whether the request was successful (`true`) |
| `message` | String | Success message |
| `tripId` | String | The trip identifier that was created or updated |

**Response Headers:**
- `X-Request-Received-At`: Timestamp when the request was received (milliseconds)
- `X-Request-Processed-At`: Timestamp when the request was processed (milliseconds)
- `X-Time-Taken-Ms`: Total processing time in milliseconds

---

### Error Response

**HTTP Status Code:** `400 Bad Request` or `500 Internal Server Error`

**Response Body:**
```json
{
    "success": false,
    "message": "Error message describing what went wrong"
}
```

**Response Fields:**

| Field | Data Type | Description |
|-------|-----------|-------------|
| `success` | Boolean | Always `false` for error responses |
| `message` | String | Detailed error message describing the issue |

---

## Validation Rules

### Date and Time Validations

1. **Trip Start Date Validation**
   - `tripStartDate` (root level) must equal `data.origin.dateTime`
   - `origin.dateTime` must not be greater than the first city's `startTime` (if cities are present)

2. **Trip End Date Validation**
   - `tripEndDate` (root level) must equal:
     - `data.return.dateTime` (for round trips), OR
     - Last city's `endTime` (for endTripHere trips)
   - `tripEndDate` must not be before `tripStartDate`

3. **City Time Validations**
   - `startTime` must not be greater than `endTime` for any city
   - Previous city's `endTime` must not be greater than next city's `startTime`

### Field Relationship Validations

1. **Time Field Consistency**
   - `startTime` and `arrivalTime` must be equal (if both are provided)
   - `endTime` and `departureTime` must be equal (if both are provided)
   - `origin.departureTime` should equal `origin.dateTime` (if `departureTime` is provided)
   - `return.arrivalTime` should equal `return.dateTime` (if `arrivalTime` is provided)

### Conditional Validations

1. **Round Trip Requirements**
   - `data.return` object is **required**
   - `endTripHere` must be `false` or not present

2. **End Trip Here Requirements**
   - `endTripHere` must be `true`
   - `data.return` must not be present

3. **Employee Data Requirements**
   - If employee does not exist: `employeeData` is **required**

4. **Flight Travel Requirements**
   - If `travel.type` is `flight`: `arrivalTime` and `departureTime` are **required** for cities and return

---

## Sample Requests

### Example 1: Round Trip

This example demonstrates a simple round trip from Bhopal to Indore and back.

**Request:**
```json
{
    "tripId": "TRIP-BRLPS-0007",
    "employeeCode": "406542",
    "tripStatus": "PENDING",
    "tripStartDate": 1767234900000,
    "tripEndDate": 1767342780000,
    "data": {
        "allowance": 0,
        "title": "Testing Round trip Bhopal-Indore-Bhopal",
        "origin": {
            "cityName": "Bhopal",
            "dateTime": 1767234900000,
            "departureTime": 1767234900000
        },
        "cities": [
            {
                "cityName": "Indore",
                "arrivalTime": 1767249000000,
                "departureTime": 1767324600000,
                "startTime": 1767249000000,
                "endTime": 1767324600000,
                "stay": {
                    "type": "None"
                },
                "travel": {
                    "type": "Flight"
                }
            }
        ],
        "return": {
            "cityName": "Bhopal",
            "arrivalTime": 1767342780000,
            "dateTime": 1767342780000,
            "stay": {},
            "travel": {
                "type": "Flight"
            }
        }
    },
    "approvers": {
        "tripApprovers": [
            {
                "name": "Arif",
                "email": "user4@hditybirlh.in",
                "code": "454681",
                "status": "APPROVED",
                "approvalTimestamp": 1766143664983
            }
        ]
    },
    "employeeData": {
        "firstName": "Shruti",
        "lastName": "Shrivastav",
        "email": "shruti9@yopmail.com",
        "mobile": "9830503032",
        "code": "406542",
        "department": "QA",
        "office": "1111- atlanta .",
        "gender": "Male",
        "customFields": {}
    }
}
```

---

### Example 2: Multi-City with Round Trip

This example demonstrates a complex multi-city trip with different travel and stay types.

**Request:**
```json
{
    "tripId": "TRIP-EXT-00011",
    "employeeCode": "10001",
    "tripStatus": "PENDING",
    "tripStartDate": 1772668800000,
    "tripEndDate": 1773204000000,
    "data": {
        "title": "Testing multi city with different travel type flight, cab, bus, train, own and stay type hotel or none",
        "allowance": 0,
        "origin": {
            "cityName": "Silchar",
            "dateTime": 1772668800000,
            "departureTime": 1772668800000
        },
        "cities": [
            {
                "cityName": "Kolkata",
                "arrivalTime": 1772683200000,
                "startTime": 1772683200000,
                "endTime": 1772736000000,
                "stay": {
                    "type": "Hotel"
                },
                "travel": {
                    "type": "Flight"
                }
            },
            {
                "cityName": "Hyderabad",
                "startTime": 1772743200000,
                "endTime": 1772856000000,
                "stay": {
                    "type": "None"
                },
                "travel": {
                    "type": "Bus"
                }
            },
            {
                "cityName": "Bengaluru",
                "startTime": 1772863200000,
                "endTime": 1772976000000,
                "stay": {
                    "type": "None"
                },
                "travel": {
                    "type": "Train"
                }
            },
            {
                "cityName": "Mumbai",
                "startTime": 1772983200000,
                "endTime": 1773096000000,
                "stay": {
                    "type": "None"
                },
                "travel": {
                    "type": "cab"
                }
            },
            {
                "cityName": "Delhi",
                "startTime": 1773103200000,
                "endTime": 1773187140000,
                "stay": {
                    "type": "None"
                },
                "travel": {
                    "type": "own"
                }
            }
        ],
        "return": {
            "cityName": "Silchar",
            "dateTime": 1773204000000,
            "travel": {
                "type": "none"
            }
        }
    },
    "approvers": {
        "tripApprovers": [
            {
                "name": "Shruti Test",
                "email": "shruti9@yopmail.com",
                "code": "10001",
                "status": "APPROVED",
                "approvalTimestamp": 1730925564000
            }
        ]
    },
    "employeeData": {
        "firstName": "Shruti",
        "lastName": "Test",
        "email": "shruti9@yopmail.com",
        "mobile": "6776543228",
        "code": "10001",
        "department": "Department 1",
        "office": "grade 1",
        "gender": "FEMALE"
    }
}
```

---

### Example 3: End Trip Here (No Return City)

This example demonstrates a trip that ends at the last destination without returning to the origin.

**Request:**
```json
{
    "tripId": "TRIP-BRLPS-0009",
    "employeeCode": "406542",
    "tripStatus": "PENDING",
    "tripStartDate": 1767674711000,
    "tripEndDate": 1767779100000,
    "data": {
        "title": "endTripHere Chennai-hydrabad-bangalore",
        "allowance": 0,
        "origin": {
            "cityName": "Chennai",
            "dateTime": 1767674711000,
            "departureTime": 1767674711000
        },
        "cities": [
            {
                "cityName": "Hyderabad",
                "arrivalTime": 1767682041000,
                "startTime": 1767682041000,
                "departureTime": 1767757462000,
                "endTime": 1767757462000,
                "stay": {
                    "type": "None"
                },
                "travel": {
                    "type": "Flight"
                }
            },
            {
                "cityName": "bangalore",
                "arrivalTime": 1767535200000,
                "startTime": 1767535200000,
                "departureTime": 1767779100000,
                "endTime": 1767779100000,
                "stay": {
                    "type": "None"
                },
                "travel": {
                    "type": "Flight"
                }
            }
        ],
        "endTripHere": true
    },
    "approvers": {
        "tripApprovers": [
            {
                "name": "Arif",
                "email": "user4@hditybirlh.in",
                "code": "454681",
                "status": "APPROVED",
                "approvalTimestamp": 1766143664983
            }
        ]
    },
    "employeeData": {
        "firstName": "Shruti",
        "lastName": "Shrivastav",
        "email": "shruti9@yopmail.com",
        "mobile": "9830503032",
        "code": "406542",
        "department": "QA",
        "office": "1111- atlanta .",
        "gender": "Male",
        "customFields": {}
    }
}
```

---

## Error Handling

### Common Error Messages

The following table lists common error messages and their descriptions:

| Error Message | HTTP Status | Description |
|---------------|-------------|------------|
| `tripStartDate does not match with data.origin.dateTime` | 400 | The `tripStartDate` value at the root level does not match the `data.origin.dateTime` value |
| `tripEndDate does not match with data.return.dateTime` | 400 | The `tripEndDate` value does not match `data.return.dateTime` (for round trips) |
| `tripEndDate does not match with last city endTime` | 400 | The `tripEndDate` value does not match the last city's `endTime` (for endTripHere trips) |
| `tripEndDate cannot be before tripStartDate` | 400 | The trip end date is chronologically before the trip start date |
| `employee not found, please provide employeeData to create new Employee` | 400 | The specified employee does not exist in the system and `employeeData` was not provided in the request |
| `Origin city with name {cityName} not found` | 400 | The origin city name is not recognized in the DICE system. City names must match exactly |
| `City with name {cityName} not found` | 400 | A city name in the `cities` array is not recognized in the DICE system |
| `Return city with name {cityName} not found` | 400 | The return city name is not recognized in the DICE system |
| `some approvers with code {codes} not found` | 400 | One or more approver codes specified in the request do not exist in the system |
| `Trip APIs not enabled for this client` | 403 | The Trip API feature is not enabled for the client associated with the provided credentials |

### Error Response Format

All error responses follow this structure:

```json
{
    "success": false,
    "message": "Detailed error message describing the validation failure or system error"
}
```

---

## Important Notes

### Timestamps
- All timestamps must be provided in **milliseconds** (epoch time)
- Ensure timestamps are consistent across related fields (e.g., `tripStartDate` and `origin.dateTime`)

### City Names
- City names must **match exactly** with city names in the DICE system
- City name matching is case-sensitive
- Verify city names before making API calls

### Employee Management
- If an employee does not exist, the system will automatically create a new employee record using `employeeData`
- The `employeeData.code` must match the `employeeCode` at the root level
- Employee creation is transactional - if trip creation fails, employee creation is rolled back

### Approvers
- All approver codes must exist in the system before trip creation
- The system validates approvers before processing the trip
- Approver validation failure will result in a 400 error response

### Trip Status
- Valid status values include: `PENDING`, `APPROVED`, `ACTIVE`, `COMPLETED`, etc.
- Status values are case-sensitive

### Allowance
- The `allowance` field is mandatory and must be provided
- Send `0` if no allowance is applicable
- Allowance amount is in the base currency

### Flight Travel
- When `travel.type` is `"flight"` (case-insensitive), `arrivalTime` and `departureTime` are **required** for:
  - All cities in the `cities` array
  - The `return` object (if present)

### Performance Considerations
- The API includes performance timing headers in responses:
  - `X-Request-Received-At`: Request reception timestamp
  - `X-Request-Processed-At`: Request processing completion timestamp
  - `X-Time-Taken-Ms`: Total processing time in milliseconds
- Large multi-city trips may take longer to process
- Consider implementing retry logic for transient failures

---

## Testing Checklist

Use this checklist to ensure your API requests are properly formatted before submission:

### Authentication
- [ ] All required headers are present (`DICE-APP-ID`, `X-CLIENT-ID`, `X-CLIENT-SECRET`, `X-Forwarded-For`)
- [ ] Header values are correct and valid
- [ ] `Content-Type` header is set to `application/json`

### Trip Identification
- [ ] `tripId` is unique and follows the expected format
- [ ] `tripId` has not been used in previous requests (unless updating)

### Date and Time Validation
- [ ] `tripStartDate` equals `data.origin.dateTime`
- [ ] `tripEndDate` matches:
  - `data.return.dateTime` (for round trips), OR
  - Last city's `endTime` (for endTripHere trips)
- [ ] `tripEndDate` is not before `tripStartDate`
- [ ] All timestamps are in milliseconds (epoch time)

### City Information
- [ ] All city names exist in the DICE system
- [ ] City names match exactly (case-sensitive)
- [ ] Origin city is specified
- [ ] For round trips: return city is specified
- [ ] For endTripHere trips: `endTripHere` is set to `true` and return object is absent
- [ ] Time validations: `startTime ≤ endTime` for all cities
- [ ] Sequential validation: previous city's `endTime ≤ next city's startTime`

### Employee Information
- [ ] Employee exists in the system, OR
- [ ] `employeeData` is provided with all required fields
- [ ] `employeeData.code` matches `employeeCode` at root level

### Approvers
- [ ] All approver codes exist in the system
- [ ] At least one approver is specified
- [ ] All approver fields are provided (name, email, code, status, approvalTimestamp)

### Travel and Stay Information
- [ ] Travel type is specified for all cities and return (if applicable)
- [ ] Stay type is specified for all cities
- [ ] If flight travel: `arrivalTime` and `departureTime` are provided

### Request Format
- [ ] Request body is valid JSON
- [ ] All required fields are present
- [ ] Data types match the specification (e.g., timestamps are numbers, not strings)

---

## Support and Contact

For API support, integration assistance, or to report issues, please contact the DICE API support team.

**Document Version:** 1.0  
**Last Updated:** 2025  
**API Version:** v1
