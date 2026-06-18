# ICS Transfer API - Detailed Usage Guide

> **Base URL:** `https://trial-dev.tourchain.net/b2badminapi/api/IcsTransfer`\
****Content-Type:** `application/json`\
****Authentication:** Bearer JWT Token (required for all endpoints)

---

## 1. Authentication

All API endpoints require a **Bearer JWT Token** in the request header.

### Required Headers

| Header | Value | Description |
| --- | --- | --- |
| `Authorization` | `Bearer <jwt_token>` | A valid JWT token |
| `Content-Type` | `application/json` | Request content type |

### JWT Token Structure

The JWT token is signed with `HS256` using the `JwtSecret` configured on the server. The token contains the following claims:

| Claim | Type | Description |
| --- | --- | --- |
| `Id` | `string` | User ID |
| `Client` | `string` | `"true"` if the user is a client login |
| `IsGuest` | `string` | `"True"` if the user is a guest manager |
| `Username` | `string` | Login username |
| `Password` | `string` | Authentication code (used for guest logins only) |

### Example Header

```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4
Content-Type: application/json
```

### Authentication Errors

| HTTP Status | Description |
| --- | --- |
| `401` | Token is invalid, expired, or missing |

---

## 2. API: Create or Update Transfer Booking

### `POST /api/IcsTransfer/webhook/booking-complete`

Creates a new or updates an existing ICS transfer booking (airport transfer: Bali, Thailand, Vietnam, etc.).

---

### Update: Input Tolerance

The `booking-complete` webhook now accepts a more tolerant payload for optional travel and geo fields:

- Date fields accept both `YYYY-MM-DD` and ISO datetime
- `countryCodePhone` accepts values with or without `+`
- `time`, `flightNumber`, `channelFareType` can be empty, `null`, or `N/A`
- `accommodation_items[].id`, `hotel_info.id`, and `geo_data.place_id` can be empty, `null`, or `N/A`
- Extra JSON fields are ignored safely
- Region resolution is best-effort only and does not block the webhook if geo enrichment cannot be resolved

---

### 2.1 Request Body 

```json
{
  "number": "B2024-001234",
  "stopFollowUpButton": true,
  "customer_email": "john.doe@example.com",
  "customer_given_name": "John",
  "customer_surname": "Doe",
  "customer_phone": "+84901234567",
  "transfer_information": {
    "type": "arrival-and-departure",
    "checkInDate": "2024-12-15",
    "checkOutDate": "2024-12-20",
    "fullname": "John Doe",
    "countryCodePhone": "+84",
    "phone": "0901234567",
    "luggage": 2,
    "oversizeLuggage": 0,
    "babyCarSeat": 0,
    "boosterSeat": 0,
    "notes": "Additional notes if any",
    "pickUpDescription": "Hotel lobby, ground floor",
    "dropOffDescription": "Terminal 2 - International Airport",
    "arrival": {
      "date": "2024-12-15",
      "time": "14:30",
      "flightNumber": "VN123",
      "channelFareType": "DPS-AR-Z1SED",
      "adults": 2,
      "children": 1,
      "voucherCode": "7SNWFY9S",
      "pickUpDescription": "Ngurah Rai International Airport (DPS)",
      "dropOffDescription": "The Ritz Carlton Bali"
    },
    "departure": {
      "date": "2024-12-20",
      "time": "09:00",
      "flightNumber": "VN456",
      "channelFareType": "DPS-DE-Z1SED",
      "adults": 2,
      "children": 1,
      "voucherCode": "CFQSAPP3",
      "pickUpDescription": "The Ritz Carlton Bali",
      "dropOffDescription": "Ngurah Rai International Airport (DPS)"
    }
  },
  "accommodation_items": [
    {
      "id": "hotel-item-id-001",
      "reservation": {
        "check_in": "2024-12-15",
        "check_out": "2024-12-20",
        "hotel_info": {
          "id": "hotel-bali-abc123",
          "name": "The Ritz Carlton Bali",
          "geo_data": {
            "country": "Indonesia",
            "administrative_area_level_1": "Bali",
            "place_id": "ChIJN1t_tDeuEmsRUsoyG83frY4"
          }
        }
      }
    }
  ]
}
```

---

### 2.2 Request Parameters

#### Root fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `number` | `string` | ✅ Yes | Booking number. e.g. `"B2024-001234"` |
| `stopFollowUpButton` | `boolean` | ✅ Yes | **Must be** `true`. Confirms the booking is complete |
| `customer_email` | `string` | ✅ Yes | Valid customer email address |
| `customer_given_name` | `string` | ✅ Yes | Customer's first name |
| `customer_surname` | `string` | ✅ Yes | Customer's last name |
| `customer_phone` | `string` | ✅ Yes | Customer's phone number |
| `transfer_information` | `object` | ✅ Yes | Detailed transfer information. See table below |
| `accommodation_items` | `array` | ✅ Yes | List of accommodation items. At least 1 item required |

---

#### `transfer_information` object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `type` | `string` | ✅ Yes | Transfer direction: `"arrival"`, `"departure"`, or `"arrival-and-departure"` |
| `checkInDate` | `string` | ❌ No | Check-in date. Accepts `"YYYY-MM-DD"` or ISO datetime |
| `checkOutDate` | `string` | ❌ No | Check-out date. Accepts `"YYYY-MM-DD"` or ISO datetime |
| `fullname` | `string` | ✅ Yes | Full name of the lead passenger |
| `countryCodePhone` | `string` | ❌ No | Country phone code. Accepts `"+84"` or `"84"` |
| `phone` | `string` | ❌ No | Phone number (without country code) |
| `luggage` | `integer` | ❌ No (default: `0`) | Number of luggage pieces. Must be &gt;= 0 |
| `oversizeLuggage` | `integer` | ❌ No (default: `0`) | Oversize luggage. Accepts only `0` or `1` |
| `babyCarSeat` | `integer` | ❌ No (default: `0`) | Baby car seat required. Accepts only `0` or `1` |
| `boosterSeat` | `integer` | ❌ No (default: `0`) | Booster seat count. Accepts any positive integer |
| `notes` | `string` | ❌ No | Additional notes |
| `pickUpDescription` | `string` | ❌ No | Booking-level pick-up point. Used as fallback when a leg has no `pickUpDescription` (swapped for the departure leg). Prefer the per-leg fields when hotels differ between legs |
| `dropOffDescription` | `string` | ❌ No | Booking-level drop-off point. Used as fallback when a leg has no `dropOffDescription` |
| `arrival` | `object` | ✅ When `type` includes `"arrival"` | Arrival transfer details. See `transfer leg` table below |
| `departure` | `object` | ✅ When `type` includes `"departure"` | Departure transfer details. See `transfer leg` table below |

---

#### `arrival` / `departure` (Transfer Leg) object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `date` | `string` | ❌ No | Flight date. Accepts `"YYYY-MM-DD"` or ISO datetime. Can be empty, `null`, or `N/A` |
| `time` | `string` | ❌ No | Flight time. Accepts `"HH:mm"`, empty, `null`, or `N/A` |
| `flightNumber` | `string` | ❌ No | IATA flight number or empty, `null`, `N/A` |
| `channelFareType` | `string` | ❌ No | Transfer fare/service code used for downstream service sync. Can be empty, `null`, or `N/A` |
| `adults` | `integer` | ✅ Yes | Number of adults. Must be &gt;= 1 |
| `children` | `integer` | ❌ No (default: `0`) | Number of children. Must be &gt;= 0 |
| `voucherCode` | `string` | ❌ No | LE voucher code for **this specific leg** (distinct from the root `number`). Persisted to the leg service/vehicle as `legVoucherCode` |
| `pickUpDescription` | `string` | ❌ No | Pick-up point for this leg. When present, used directly for this leg's route (no arrival/departure swap). Falls back to the top-level `pickUpDescription` if omitted |
| `dropOffDescription` | `string` | ❌ No | Drop-off point for this leg. When present, used directly for this leg's route. Falls back to the top-level `dropOffDescription` if omitted |

---

#### `accommodation_items[]` (array)

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | `string` | ❌ No | Accommodation item ID. Can be empty, `null`, or `N/A` |
| `reservation` | `object` | ✅ Yes | Reservation details. See `reservation` table below |

---

#### `reservation` object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `check_in` | `string` | ✅ Yes | Check-in date. Accepts `"YYYY-MM-DD"` or ISO datetime |
| `check_out` | `string` | ✅ Yes | Check-out date. Accepts `"YYYY-MM-DD"` or ISO datetime |
| `hotel_info` | `object` | ✅ Yes | Hotel information. See `hotel_info` table below |

---

#### `hotel_info` object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | `string` | ❌ No | Unique hotel ID in the system. Can be empty, `null`, or `N/A` |
| `name` | `string` | ✅ Yes | Hotel name |
| `geo_data` | `object` | ❌ No | Hotel geographic data |

---

#### `geo_data` object

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `country` | `string` | ❌ No | Country name. e.g. `"Indonesia"` |
| `administrative_area_level_1` | `string` | ❌ No | Province/Region. e.g. `"Bali"` |
| `place_id` | `string` | ❌ No | Google Place ID. Can be empty, `null`, or `N/A` |

---

### 2.3 Validation Rules

| Field | Rule |
| --- | --- |
| `number` | Must not be empty |
| `stopFollowUpButton` | Must be `true` |
| `customer_email` | Must not be empty and must be a valid email address |
| `customer_given_name` | Must not be empty |
| `customer_surname` | Must not be empty |
| `customer_phone` | Must not be empty |
| `transfer_information` | Must not be null |
| `transfer_information.type` | Must be `"arrival"`, `"departure"`, or `"arrival-and-departure"` |
| `transfer_information.fullname` | Must not be empty |
| `arrival.date` | Optional. If provided, must be `YYYY-MM-DD` or a valid ISO datetime |
| `arrival.adults` | Must be &gt;= 1 |
| `departure.date` | Optional. If provided, must be `YYYY-MM-DD` or a valid ISO datetime |
| `departure.adults` | Must be &gt;= 1 |
| `luggage` | Must be &gt;= 0 |
| `oversizeLuggage` | Must be `0` or `1` only |
| `babyCarSeat` | Must be `0` or `1` only |
| `boosterSeat` | Must be &gt;= 0 |
| `accommodation_items` | Must not be empty. At least 1 item required |
| `accommodation_items[].id` | Optional. Empty, `null`, and `N/A` are accepted |
| `reservation.check_in` | Must not be empty |
| `reservation.check_out` | Must not be empty |
| `hotel_info.id` | Optional. Empty, `null`, and `N/A` are accepted |
| `hotel_info.name` | Must not be empty |

---

### 2.4 Success Response

**HTTP Status:** `200 OK`

```json
{
  "status": 200,
  "message": "Booking updated successfully"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `status` | `integer` | HTTP status code (`200`) |
| `message` | `string` | Success message |

---

### 2.5 Error Response (Validation - 422)

**HTTP Status:** `422 Unprocessable Entity`

```json
{
  "status": 422,
  "error": 1,
  "messages": {
    "type": "error",
    "message": {
      "customer_email": "Valid customer email is required",
      "transfer_information.type": "Transfer type must be 'arrival-and-departure', 'arrival', or 'departure'",
      "accommodation_items[0].id": "Accommodation item id is required"
    }
  }
}
```

| Field | Type | Description |
| --- | --- | --- |
| `status` | `integer` | HTTP status code |
| `error` | `integer` | Always `1` when an error occurs |
| `messages.type` | `string` | Error type, always `"error"` |
| `messages.message` | `string|object` | A plain error string, or an object with field names as keys |

---

### 2.6 cURL Example

```bash
curl -X POST https://trial-dev.tourchain.net/b2badminapi/api/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "B2024-001234",
    "stopFollowUpButton": true,
    "customer_email": "john.doe@example.com",
    "customer_given_name": "John",
    "customer_surname": "Doe",
    "customer_phone": "+84901234567",
    "transfer_information": {
      "type": "arrival",
      "fullname": "John Doe",
      "luggage": 2,
      "oversizeLuggage": 0,
      "babyCarSeat": 0,
      "boosterSeat": 0,
      "arrival": {
        "date": "2024-12-15",
        "time": "14:30",
        "flightNumber": "VN123",
        "channelFareType": "DPS-AR-Z1SED",
        "adults": 2,
        "children": 0,
        "voucherCode": "7SNWFY9S",
        "pickUpDescription": "Ngurah Rai International Airport (DPS)",
        "dropOffDescription": "The Ritz Carlton Bali"
      }
    },
    "accommodation_items": [
      {
        "id": "hotel-item-001",
        "reservation": {
          "check_in": "2024-12-15",
          "check_out": "2024-12-20",
          "hotel_info": {
            "id": "hotel-bali-001",
            "name": "The Ritz Carlton Bali"
          }
        }
      }
    ]
  }'
```

---

## 3. API: Cancel Transfer Booking

### `DELETE /api/IcsTransfer/webhook/cancel/{bookingNumber}`

Cancels an ICS transfer booking by booking number. Sets the booking status to `"Cancelled"` in the system.

---

### 3.1 URL Parameter

| Parameter | Location | Type | Required | Description |
| --- | --- | --- | --- | --- |
| `bookingNumber` | Path | `string` | ✅ Yes | The booking number to cancel. e.g. `B2024-001234` |

### Example URL

```
DELETE /api/IcsTransfer/webhook/cancel/B2024-001234
```

---

### 3.2 Request Body

No request body required. Pass `bookingNumber` directly in the URL path.

---

### 3.3 Processing Flow

When the API is called, the system performs the following steps:

1. Find the `LeadModel` (pipedriveTour) containing a booking with `voucherCode = bookingNumber`.
2. Update `status = "Cancelled"` and `updatedDate = DateTime.UtcNow` for that booking.

> **Note:** If no matching booking is found in the system, the API returns `404 Not Found`.

---

### 3.4 Success Response

**HTTP Status:** `200 OK`

```json
{
  "status": 200,
  "message": "Booking updated successfully"
}
```

| Field | Type | Description |
| --- | --- | --- |
| `status` | `integer` | HTTP status code (`200`) |
| `message` | `string` | Success message |

---

### 3.5 Error Response (Validation - 422)

**HTTP Status:** `422 Unprocessable Entity` — when `bookingNumber` is empty:

```json
{
  "status": 422,
  "error": 1,
  "messages": {
    "type": "error",
    "message": "Booking number is required"
  }
}
```

**HTTP Status:** `404 Not Found` — when `bookingNumber` does not exist:

```json
{
  "status": 404,
  "error": 1,
  "messages": {
    "type": "error",
    "message": "Booking 'B2024-000000' not found"
  }
}
```

---

### 3.6 cURL Example

```bash
curl -X DELETE https://trial-dev.tourchain.net/b2badminapi/api/IcsTransfer/webhook/cancel/B2024-001234 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4"
```

## 4. HTTP Status Codes Summary

| Status Code | Scenario |
| --- | --- |
| `200` | Success |
| `401` | Token is invalid, expired, or missing |
| `404` | Booking number not found (Cancel endpoint) |
| `422` | Input validation failed |

---

## 5. Full Example: `arrival-and-departure` Booking

```bash
curl -X POST https://trial-dev.tourchain.net/b2badminapi/api/IcsTransfer/webhook/booking-complete \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4" \
  -H "Content-Type: application/json" \
  -d '{
    "number": "B2024-VN-9988",
    "stopFollowUpButton": true,
    "customer_email": "nguyen.van.a@gmail.com",
    "customer_given_name": "Van A",
    "customer_surname": "Nguyen",
    "customer_phone": "+84912345678",
    "transfer_information": {
      "type": "arrival-and-departure",
      "checkInDate": "2024-12-20",
      "checkOutDate": "2024-12-27",
      "fullname": "Nguyen Van A",
      "countryCodePhone": "+84",
      "phone": "0912345678",
      "luggage": 3,
      "oversizeLuggage": 0,
      "babyCarSeat": 1,
      "boosterSeat": 1,
      "notes": "Guest has an 18-month-old baby",
      "pickUpDescription": "Hotel lobby, ground floor",
      "dropOffDescription": "International Terminal T1",
      "arrival": {
        "date": "2024-12-20",
        "time": "10:00",
        "flightNumber": "VN7201",
        "channelFareType": "DAD-AR-Z1SED",
        "adults": 2,
        "children": 1,
        "voucherCode": "7SNWFY9S",
        "pickUpDescription": "Da Nang International Airport (DAD)",
        "dropOffDescription": "InterContinental Danang Sun Peninsula"
      },
      "departure": {
        "date": "2024-12-27",
        "time": "08:30",
        "flightNumber": "VN7202",
        "channelFareType": "DAD-DE-Z1SED",
        "adults": 2,
        "children": 1,
        "voucherCode": "CFQSAPP3",
        "pickUpDescription": "InterContinental Danang Sun Peninsula",
        "dropOffDescription": "Da Nang International Airport (DAD)"
      }
    },
    "accommodation_items": [
      {
        "id": "acc-danang-2024",
        "reservation": {
          "check_in": "2024-12-20",
          "check_out": "2024-12-27",
          "hotel_info": {
            "id": "hotel-intercontinental-dn",
            "name": "InterContinental Danang Sun Peninsula",
            "geo_data": {
              "country": "Vietnam",
              "administrative_area_level_1": "Da Nang",
              "place_id": "ChIJrTLr-GyuEmsRBfy61i59si0"
            }
          }
        }
      }
    ]
  }'
```

**Response:**

```json
{
  "status": 200,
  "message": "Booking updated successfully"
}
```

---

## 6. Cancel Booking Example

```bash
curl -X DELETE https://trial-dev.tourchain.net/b2badminapi/api/IcsTransfer/webhook/cancel/B2024-VN-9988 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1bmlxdWVfbmFtZSI6Imljcy1hcGktc2VydmljZSIsInJvbGUiOiJBZG1pbiIsInN1YiI6Imljcy1pbnRlZ3JhdGlvbiJ9.4F7TpC39-UNR4HVtQ3bKANaGE1Gh8qrHb0vX6-Vk_c4"
```

**Response:**

```json
{
  "status": 200,
  "message": "Booking updated successfully"
}
```

---

## 7. Important Notes

- `stopFollowUpButton` **must always be** `true` — the server will reject the request if the value is `false`.
- `transfer_information.type` determines which leg objects are required:
  - `"arrival"` → `arrival` object is required
  - `"departure"` → `departure` object is required
  - `"arrival-and-departure"` → both `arrival` and `departure` objects are required
- `oversizeLuggage` **and** `babyCarSeat` only accept integer values `0` or `1` (not boolean `true`/`false`). `boosterSeat` accepts any positive integer.
- **JWT Token** must be sent in the correct format: `Bearer <token>` (with a space between `Bearer` and the token).
- The Cancel API returns `404 Not Found` when the booking number does not exist in the system.

### Vehicle Data Saved from ICS Transfer

When a booking is processed, the following fields from the request are persisted to each vehicle record in the downstream booking:

| Request Field | Vehicle Field | Source |
| --- | --- | --- |
| `transfer_information.luggage` | `luggage` | `transferInformation` |
| `transfer_information.oversizeLuggage` | `oversizeLuggage` | `transferInformation` |
| `transfer_information.babyCarSeat` | `babyCarSeat` | `transferInformation` |
| `transfer_information.boosterSeat` | `boosterSeat` | `transferInformation` |
| `arrival.adults` / `departure.adults` | `adultsCount` | Leg document |
| `arrival.children` / `departure.children` | `childrenCount` | Leg document |
| `arrival.voucherCode` / `departure.voucherCode` | `legVoucherCode` | Leg document (per-leg voucher; root `number` still maps to `voucherCode`) |
| (resolved from leg services) | `packageName` | `ResolveVehiclePackageContext` |
| (resolved from leg services) | `packageInternalName` | `ResolveVehiclePackageContext` |
| (resolved from leg) | `flightInfo` | Built from flight number + time |
| (resolved from transfer) | `hotelName` | Resolved from accommodation items |
| `arrival`/`departure` `pickUpDescription` / `dropOffDescription` | `pickupPoint` / `dropOffPoint` | Per-leg route when present; otherwise the top-level `pickUpDescription`/`dropOffDescription` (swapped for departure) |
