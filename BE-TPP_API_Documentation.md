# BE-TPP API Documentation

**Version:** 1.5
**Last Updated:** 13 April 2026
**Project:** BE-TPP IoT (Breatheeasy Total Positive Pressure)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Authentication](#2-authentication)
3. [Client Setup](#3-client-setup)
4. [API Functions  --  Sensor Data](#4-api-functions--sensor-data)
   - 4.1 [get_latest_readings](#41-get_latest_readings)
   - 4.2 [get_readings_range](#42-get_readings_range)
5. [API Functions  --  Device Status](#5-api-functions--device-status)
   - 5.1 [get_device_status](#51-get_device_status)
   - 5.2 [get_device_status_range](#52-get_device_status_range)
6. [API Functions  --  Device Management](#6-api-functions--device-management)
   - 6.1 [get_my_devices](#61-get_my_devices)
   - 6.2 [register_device](#62-register_device)
   - 6.3 [update_device](#63-update_device)
   - 6.4 [unregister_device](#64-unregister_device)
7. [API Functions  --  Air Quality Map](#7-api-functions--air-quality-map)
   - 7.1 [get_air_quality_map](#71-get_air_quality_map)
   - 7.2 [get_public_air_quality](#72-get_public_air_quality)
8. [API Functions  --  Fan Control](#8-api-functions--fan-control)
   - 8.1 [fan_control (Edge Function)](#81-fan_control-edge-function)
   - 8.2 [Realtime Subscription (device_status)](#82-realtime-subscription-device_status)
9. [API Functions  --  Profile Management](#9-api-functions--profile-management)
   - 9.1 [Get Profile](#91-get-profile)
   - 9.2 [Update Profile](#92-update-profile)
10. [API Functions  --  Outdoor Air Quality](#10-api-functions--outdoor-air-quality)
    - 10.1 [get-outdoor-air (Edge Function) ⚠ DEPRECATED](#101-get-outdoor-air-edge-function)
    - 10.2 [get-outdoor-air-batch (Edge Function) ⚠ DEPRECATED](#102-get-outdoor-air-batch-edge-function)
    - 10.3 [aqicn_stations (REST Query) ★ NEW](#103-aqicn_stations-rest-query)
11. [API Functions  --  Thai Air Quality (Nationwide)](#11-api-functions--thai-air-quality-nationwide)
    - 11.1 [v_thai_air_latest (View — REST Query)](#111-v_thai_air_latest-view--rest-query)
    - 11.2 [thai_air_readings (History — REST Query)](#112-thai_air_readings-history--rest-query)
    - 11.3 [fetch-thai-air (Edge Function — Cron/Admin)](#113-fetch-thai-air-edge-function--cronadmin)
12. [Database Schema](#12-database-schema)
13. [RLS Policies Summary](#13-rls-policies-summary)
14. [Error Reference](#14-error-reference)
15. [Rate Limits & Constraints](#15-rate-limits--constraints)

---

## 1. Overview

BE-TPP (Breatheeasy Total Positive Pressure) is an IoT platform for monitoring air quality and controlling ventilation fans. The system collects sensor data (PM2.5, PM10, CO2, temperature, humidity) from ESP32 devices and provides real-time monitoring and fan control through a web dashboard.

### Architecture

```
ESP32 Devices --MQTT--> EMQX Cloud --REST--> Supabase PostgreSQL
                                                    |
                                              +-----+-----+
                                              |  RPC API   |
                                              |  Auth API  |
                                              |  Realtime  |
                                              |  Edge Fn   |
                                              +-----+-----+
                                                    |
                                    +---------------+---------------+
                                    |               |               |
                              Web Dashboard   AQICN API     Air4Thai + CUSense
                                          (map/bounds,    (245 stations,
                                           pg_cron 1hr)    pg_cron 1hr)
```

### Tech Stack

| Component       | Technology                          |
|-----------------|-------------------------------------|
| Microcontroller | ESP32 (firmware v1.8.2)             |
| MQTT Broker     | EMQX Cloud (Serverless)             |
| Database        | Supabase (PostgreSQL)               |
| Authentication  | Supabase Auth (Magic Link + Email OTP) |
| Edge Functions  | Supabase Edge Functions (Deno)      |
| Outdoor Air     | AQICN API (via Edge Function proxy) |
| Thai Air        | Air4Thai (PCD) + CUSense (Chula) — 245 stations |
| Hosting         | GitHub Pages (test console)         |
| Email           | Resend (noreply@be-tpp.com)         |

### Base URLs

| Service   | URL                                                          |
|-----------|--------------------------------------------------------------|
| REST API  | `https://brgzimwzcfbwkgymqzvy.supabase.co/rest/v1/`        |
| Auth API  | `https://brgzimwzcfbwkgymqzvy.supabase.co/auth/v1/`        |
| RPC       | `https://brgzimwzcfbwkgymqzvy.supabase.co/rest/v1/rpc/`    |
| Edge Fn   | `https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/`   |

### Data Flow

- **Sensor data**: ESP32 publishes every 10 seconds -> EMQX samples every 5 minutes -> Supabase INSERT
- **Device status**: ESP32 publishes every 30 seconds -> EMQX samples every 5 minutes -> Supabase INSERT
- **Fan control**: Web App -> Edge Function -> MQTT publish to EMQX + INSERT to `device_status` -> Realtime notification
- **Outdoor air quality**: Web/Mobile App -> Edge Function (`get-outdoor-air`) -> AQICN API (cache 1 hr in `outdoor_air_readings`)
- **Thai air quality**: pg_cron (hourly) -> Edge Function (`fetch-thai-air`) -> Air4Thai + CUSense -> `thai_air_stations` + `thai_air_readings` (245 stations, ~2s per run)
- **Data retention**: 180 days (sensor/device/outdoor), 90 days (thai_air_readings)
- **Deduplication**: `time_bucket` column with UPSERT prevents duplicate entries

---

## 2. Authentication

BE-TPP uses **Supabase Auth** with passwordless email authentication. Two methods are available:

| Method | Use Case | Flow |
|--------|----------|------|
| Magic Link | Web app (click link in email) | Email → click link → redirect back with token |
| Email OTP | Mobile app / test console | Email → enter OTP code → verify in app |

Both methods use `noreply@be-tpp.com` as the sender. The email contains both a Magic Link and an OTP code.

### Magic Link Flow

1. User submits their email address.
2. Supabase sends a Magic Link email from `noreply@be-tpp.com`.
3. User clicks the link in the email.
4. Browser redirects to the app with an access token in the URL hash.
5. Supabase client automatically exchanges the token for a session.

### Login (Send Magic Link)

```javascript
const { error } = await supabase.auth.signInWithOtp({
  email: 'user@example.com',
  options: {
    emailRedirectTo: 'https://your-app.com/callback'
  }
});
```

### Email OTP Flow (Phase 3E)

1. User submits their email address.
2. Supabase sends an email with an OTP code (6 digits).
3. User enters the OTP code in the app.
4. App calls `verifyOtp()` to exchange the code for a session.

### Login (Send OTP)

```javascript
const { error } = await supabase.auth.signInWithOtp({
  email: 'user@example.com',
  options: {
    shouldCreateUser: false  // Do not create new users via OTP
  }
});
```

### Verify OTP

```javascript
const { data, error } = await supabase.auth.verifyOtp({
  email: 'user@example.com',
  token: '17777512',   // 8-digit OTP code from email
  type: 'email'
});
// data.session contains the access token on success
```

### Session Token

After authentication, the Supabase client automatically manages the session. The access token (JWT) is included in all API requests via the `Authorization` header:

```
Authorization: Bearer <access_token>
```

### Token Refresh

Supabase client automatically refreshes tokens before they expire. You can listen for refresh events:

```javascript
supabase.auth.onAuthStateChange((event, session) => {
  if (event === 'TOKEN_REFRESHED') {
    console.log('Token refreshed:', session);
  }
  if (event === 'SIGNED_OUT') {
    console.log('Session expired or user logged out');
  }
});
```

### Logout

```javascript
await supabase.auth.signOut();
```

---

## 3. Client Setup

### Install Supabase Client

```bash
npm install @supabase/supabase-js
```

### Initialize Client

```javascript
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  'https://brgzimwzcfbwkgymqzvy.supabase.co',
  'YOUR_ANON_KEY'
);
```

### CDN (Browser)

```html
<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2"></script>
<script>
  const supabase = window.supabase.createClient(
    'https://brgzimwzcfbwkgymqzvy.supabase.co',
    'YOUR_ANON_KEY'
  );
</script>
```

### Calling RPC Functions

All PostgreSQL functions are called via Supabase RPC:

```javascript
const { data, error } = await supabase.rpc('function_name', {
  param1: 'value1',
  param2: 'value2'
});
```

---

## 4. API Functions - Sensor Data

### 4.1 get_latest_readings

Retrieves the most recent sensor reading for a specific device, including `age_seconds` to indicate data freshness.

| Property        | Value                          |
|-----------------|--------------------------------|
| **Type**        | PostgreSQL RPC Function         |
| **Auth**        | Required (authenticated user)   |
| **Ownership**   | User must own the device        |

#### Parameters

| Name          | Type   | Required | Description                        |
|---------------|--------|----------|------------------------------------|
| `p_device_id` | TEXT   | Yes      | Device serial number (e.g., `582D34712B43`) |

#### Request Example

```javascript
const { data, error } = await supabase.rpc('get_latest_readings', {
  p_device_id: '582D34712B43'
});
```

#### Response Schema

Returns a single-row array:

```json
[
  {
    "reading_time": "2026-02-15T10:30:00.000Z",
    "device_id": "582D34712B43",
    "temperature": 28.5,
    "humidity": 65.2,
    "pm25": 12.3,
    "pm10": 18.7,
    "co2": 450,
    "tvoc_index": 120,
    "battery": 100,
    "age_seconds": 45
  }
]
```

#### Response Fields

| Field          | Type            | Description                                          |
|----------------|-----------------|------------------------------------------------------|
| `reading_time` | TIMESTAMPTZ     | Timestamp of the reading (UTC)                       |
| `device_id`    | TEXT            | Device serial number                                 |
| `temperature`  | DOUBLE PRECISION| Temperature in °C                                    |
| `humidity`     | DOUBLE PRECISION| Relative humidity in %                               |
| `pm25`         | DOUBLE PRECISION| PM2.5 concentration in µg/m³                         |
| `pm10`         | DOUBLE PRECISION| PM10 concentration in µg/m³                          |
| `co2`          | DOUBLE PRECISION| CO2 concentration in ppm                             |
| `tvoc_index`   | DOUBLE PRECISION| TVOC index value                                     |
| `battery`      | DOUBLE PRECISION| Battery level (0-100)                                |
| `age_seconds`  | INTEGER         | Seconds since this reading was recorded              |

#### Error Cases

| Error                              | HTTP Code | Cause                                       |
|------------------------------------|-----------|---------------------------------------------|
| `Not authenticated`                | 400       | No valid session token                       |
| `Device not found or access denied`| 400       | Device does not exist or not owned by user   |
| Empty array `[]`                   | 200       | No sensor data recorded yet for this device  |

---

### 4.2 get_readings_range

Retrieves historical sensor readings within a specified time range. Maximum range is 7 days per query, limited to 2,000 rows.

| Property        | Value                          |
|-----------------|--------------------------------|
| **Type**        | PostgreSQL RPC Function         |
| **Auth**        | Required (authenticated user)   |
| **Ownership**   | User must own the device        |

#### Parameters

| Name          | Type        | Required | Default | Description                           |
|---------------|-------------|----------|---------|---------------------------------------|
| `p_device_id` | TEXT        | Yes      |  --        | Device serial number                  |
| `p_start`     | TIMESTAMPTZ | Yes      |  --        | Start of time range (UTC ISO 8601)    |
| `p_end`       | TIMESTAMPTZ | No       | `NOW()` | End of time range (UTC ISO 8601)      |

#### Request Example

```javascript
const now = new Date();
const past24h = new Date(now.getTime() - 24 * 60 * 60 * 1000);

const { data, error } = await supabase.rpc('get_readings_range', {
  p_device_id: '582D34712B43',
  p_start: past24h.toISOString(),
  p_end: now.toISOString()
});
```

#### Response Schema

Returns an array of readings sorted by time ascending:

```json
[
  {
    "reading_time": "2026-02-14T10:00:00.000Z",
    "device_id": "582D34712B43",
    "temperature": 27.8,
    "humidity": 62.1,
    "pm25": 15.4,
    "pm10": 22.1,
    "co2": 480,
    "tvoc_index": 110,
    "battery": 100
  },
  {
    "reading_time": "2026-02-14T10:05:00.000Z",
    "device_id": "582D34712B43",
    "temperature": 27.9,
    "humidity": 62.3,
    "pm25": 14.8,
    "pm10": 21.5,
    "co2": 475,
    "tvoc_index": 108,
    "battery": 100
  }
]
```

#### Response Fields

Same fields as `get_latest_readings`, excluding `age_seconds`.

#### Error Cases

| Error                                            | HTTP Code | Cause                                     |
|--------------------------------------------------|-----------|-------------------------------------------|
| `Not authenticated`                              | 400       | No valid session token                     |
| `Device not found or access denied`              | 400       | Device not owned by user                   |
| `Invalid time range: start must be before end`   | 400       | `p_start` >= `p_end`                       |
| `Time range too large: maximum 7 days per query` | 400       | Range exceeds 7 days                       |

---

## 5. API Functions - Device Status

### 5.1 get_device_status

Retrieves the latest status of a specific device including fan state, firmware version, network info, and online/offline status.

| Property        | Value                          |
|-----------------|--------------------------------|
| **Type**        | PostgreSQL RPC Function         |
| **Auth**        | Required (authenticated user)   |
| **Ownership**   | User must own the device        |

#### Parameters

| Name          | Type   | Required | Description              |
|---------------|--------|----------|--------------------------|
| `p_device_id` | TEXT   | Yes      | Device serial number     |

#### Request Example

```javascript
const { data, error } = await supabase.rpc('get_device_status', {
  p_device_id: '582D34712B43'
});
```

#### Response Schema

Returns a single-row array:

```json
[
  {
    "status_time": "2026-02-15T10:30:00.000Z",
    "device_id": "582D34712B43",
    "device_name": "Bedroom",
    "fan_speed": 45,
    "fan_mode": "automatic",
    "fan_state": "running",
    "fw_ver": "1.8.2",
    "esp_temp": 42.5,
    "rssi": -55,
    "heap": 180000,
    "uptime": 86400,
    "pm25_ema": 12.5,
    "co2_ema": 450.0,
    "is_online": true,
    "last_seen_seconds": 120
  }
]
```

#### Response Fields

| Field               | Type            | Description                                           |
|---------------------|-----------------|-------------------------------------------------------|
| `status_time`       | TIMESTAMPTZ     | Timestamp of the status report                        |
| `device_id`         | TEXT            | Device serial number                                  |
| `device_name`       | TEXT            | User-assigned device name                             |
| `fan_speed`         | INTEGER         | Current fan speed (0-100%)                            |
| `fan_mode`          | TEXT            | Fan mode: `"automatic"` or `"custom"`                 |
| `fan_state`         | TEXT            | Fan state: `"running"`, `"idle"`, `"off"`, etc.       |
| `fw_ver`            | TEXT            | Firmware version string                               |
| `esp_temp`          | DOUBLE PRECISION| ESP32 internal temperature in °C                      |
| `rssi`              | INTEGER         | WiFi signal strength in dBm (typically -30 to -90)    |
| `heap`              | INTEGER         | Free heap memory in bytes                             |
| `uptime`            | BIGINT          | Device uptime in seconds                              |
| `pm25_ema`          | DOUBLE PRECISION| Exponential moving average of PM2.5                   |
| `co2_ema`           | DOUBLE PRECISION| Exponential moving average of CO2                     |
| `is_online`         | BOOLEAN         | `true` if last status received within 10 minutes      |
| `last_seen_seconds` | INTEGER         | Seconds since last status report                      |

#### Error Cases

| Error                              | HTTP Code | Cause                                       |
|------------------------------------|-----------|---------------------------------------------|
| `Not authenticated`                | 400       | No valid session token                       |
| `Device not found or access denied`| 400       | Device not owned by user                     |
| Empty array `[]`                   | 200       | No status data recorded yet                  |

---

### 5.2 get_device_status_range

Retrieves historical device status entries within a specified time range. Maximum range is 7 days per query, limited to 2,000 rows. Useful for charting fan speed, RSSI, or uptime over time.

| Property        | Value                          |
|-----------------|--------------------------------|
| **Type**        | PostgreSQL RPC Function         |
| **Auth**        | Required (authenticated user)   |
| **Ownership**   | User must own the device        |

#### Parameters

| Name          | Type        | Required | Description                           |
|---------------|-------------|----------|---------------------------------------|
| `p_device_id` | TEXT        | Yes      | Device serial number                  |
| `p_start`     | TIMESTAMPTZ | Yes      | Start of time range (UTC ISO 8601)    |
| `p_end`       | TIMESTAMPTZ | Yes      | End of time range (UTC ISO 8601)      |

> **Note:** If the range exceeds 7 days, `p_end` is automatically capped to `p_start + 7 days`.

#### Request Example

```javascript
const now = new Date();
const past24h = new Date(now.getTime() - 24 * 60 * 60 * 1000);

const { data, error } = await supabase.rpc('get_device_status_range', {
  p_device_id: '582D34712B43',
  p_start: past24h.toISOString(),
  p_end: now.toISOString()
});
```

#### Response Schema

Returns an array sorted by time ascending:

```json
[
  {
    "time": "2026-02-14T10:00:00.000Z",
    "device_id": "582D34712B43",
    "fan_speed": 40,
    "fan_mode": "automatic",
    "fan_state": "running",
    "fw_ver": "1.8.2",
    "esp_temp": 41.2,
    "rssi": -52,
    "heap": 182000,
    "uptime": 72000,
    "pm25_ema": 11.8,
    "co2_ema": 440.0,
    "client_id": "be-tpp-3081b529e748"
  }
]
```

#### Response Fields

| Field        | Type        | Description                                   |
|--------------|-------------|-----------------------------------------------|
| `time`       | TIMESTAMPTZ | Timestamp of the status report                |
| `device_id`  | TEXT        | Device serial number                          |
| `fan_speed`  | INTEGER     | Fan speed (0-100%)                            |
| `fan_mode`   | TEXT        | `"automatic"` or `"custom"`                   |
| `fan_state`  | TEXT        | Fan running state                             |
| `fw_ver`     | TEXT        | Firmware version                              |
| `esp_temp`   | REAL        | ESP32 internal temperature (°C)               |
| `rssi`       | INTEGER     | WiFi signal strength (dBm)                    |
| `heap`       | INTEGER     | Free heap memory (bytes)                      |
| `uptime`     | INTEGER     | Device uptime (seconds)                       |
| `pm25_ema`   | REAL        | PM2.5 exponential moving average              |
| `co2_ema`    | REAL        | CO2 exponential moving average                |
| `client_id`  | TEXT        | ESP32 MAC-based client ID                     |

#### Error Cases

| Error                                     | HTTP Code | Cause                                   |
|-------------------------------------------|-----------|------------------------------------------|
| `Not authenticated`                       | 400       | No valid session token                    |
| `Device not found or not owned by you`    | 400       | Device not owned by user                  |

---

## 6. API Functions - Device Management

### 6.1 get_my_devices

Retrieves all devices registered to the authenticated user.

| Property        | Value                          |
|-----------------|--------------------------------|
| **Type**        | PostgreSQL RPC Function         |
| **Auth**        | Required (authenticated user)   |

#### Parameters

None.

#### Request Example

```javascript
const { data, error } = await supabase.rpc('get_my_devices');
```

#### Response Schema

```json
[
  {
    "device_id": "582D34712B43",
    "client_id": "be-tpp-cca7f7f924f0",
    "device_name": "Bedroom",
    "latitude": 13.7563,
    "longitude": 100.5018,
    "is_public": true,
    "registered_at": "2026-01-30T08:00:00.000Z",
    "updated_at": "2026-02-10T14:30:00.000Z"
  },
  {
    "device_id": "582D34712B59",
    "client_id": "be-tpp-3081b529e748",
    "device_name": "Office",
    "latitude": null,
    "longitude": null,
    "is_public": false,
    "registered_at": "2026-01-30T08:05:00.000Z",
    "updated_at": "2026-01-30T08:05:00.000Z"
  }
]
```

#### Response Fields

| Field           | Type            | Description                                  |
|-----------------|-----------------|----------------------------------------------|
| `device_id`     | TEXT            | Device serial number (unique identifier)     |
| `client_id`     | TEXT            | ESP32 MAC-based client ID (nullable)         |
| `device_name`   | TEXT            | User-assigned device name                    |
| `latitude`      | DOUBLE PRECISION| Device GPS latitude (nullable)               |
| `longitude`     | DOUBLE PRECISION| Device GPS longitude (nullable)              |
| `is_public`     | BOOLEAN         | Whether device appears on the public air quality map |
| `registered_at` | TIMESTAMPTZ     | When the device was registered               |
| `updated_at`    | TIMESTAMPTZ     | When the device was last updated             |

#### Error Cases

| Error              | HTTP Code | Cause                    |
|--------------------|-----------|--------------------------|
| `Not authenticated`| 400       | No valid session token   |
| Empty array `[]`   | 200       | User has no devices      |

---

### 6.2 register_device

Registers a new IoT device to the authenticated user's account. A device can be shared by up to **3 users** (Phase 3E).

| Property        | Value                          |
|-----------------|--------------------------------|
| **Type**        | PostgreSQL RPC Function         |
| **Auth**        | Required (authenticated user)   |
| **Sharing**     | Max 3 users per device          |

#### Parameters

| Name            | Type            | Required | Default | Description                        |
|-----------------|-----------------|----------|---------|------------------------------------|
| `p_device_id`   | TEXT            | Yes      |  --        | Device serial number               |
| `p_device_name` | TEXT            | No       | NULL    | Display name for the device        |
| `p_latitude`    | DOUBLE PRECISION| No       | NULL    | GPS latitude for map display       |
| `p_longitude`   | DOUBLE PRECISION| No       | NULL    | GPS longitude for map display      |
| `p_is_public`   | BOOLEAN         | No       | `false` | Show on public air quality map     |

#### Request Example

```javascript
const { data, error } = await supabase.rpc('register_device', {
  p_device_id: '582D34712B99',
  p_device_name: 'Living Room',
  p_latitude: 13.7563,
  p_longitude: 100.5018,
  p_is_public: true
});
```

#### Response Schema

```json
{
  "success": true,
  "device_id": "582D34712B99",
  "message": "Device registered successfully",
  "shared_users": 2
}
```

#### Response Fields

| Field          | Type    | Description                                      |
|----------------|---------|--------------------------------------------------|
| `success`      | BOOLEAN | `true` on success                                |
| `device_id`    | TEXT    | Device serial number                             |
| `message`      | TEXT    | Success message                                  |
| `shared_users` | INTEGER | Total number of users sharing this device (1-3)  |

#### Error Responses

```json
{
  "success": false,
  "error": "Device already registered to your account"
}
```

```json
{
  "success": false,
  "error": "Device has reached maximum number of users (3)"
}
```

#### Error Cases

| Error                                               | Cause                                               |
|-----------------------------------------------------|-----------------------------------------------------|
| `Authentication required`                           | No valid session token                               |
| `Device already registered to your account`         | User already owns this device                        |
| `Device has reached maximum number of users (3)`    | 3 users already registered to this device            |

---

### 6.3 update_device

Updates settings for an existing device owned by the authenticated user. Only provided parameters are updated; omitted parameters retain their current values.

| Property        | Value                          |
|-----------------|--------------------------------|
| **Type**        | PostgreSQL RPC Function         |
| **Auth**        | Required (authenticated user)   |
| **Ownership**   | User must own the device        |

#### Parameters

| Name            | Type            | Required | Default | Description                        |
|-----------------|-----------------|----------|---------|------------------------------------|
| `p_device_id`   | TEXT            | Yes      |  --        | Device serial number (lookup key)  |
| `p_device_name` | TEXT            | No       | NULL    | New display name                   |
| `p_client_id`   | TEXT            | No       | NULL    | ESP32 MAC-based client ID          |
| `p_latitude`    | DOUBLE PRECISION| No       | NULL    | New GPS latitude                   |
| `p_longitude`   | DOUBLE PRECISION| No       | NULL    | New GPS longitude                  |
| `p_is_public`   | BOOLEAN         | No       | NULL    | New public visibility setting      |

#### Request Example

```javascript
const { data, error } = await supabase.rpc('update_device', {
  p_device_id: '582D34712B43',
  p_device_name: 'Master Bedroom',
  p_is_public: true
});
```

#### Response Schema

```json
{
  "success": true,
  "device_id": "582D34712B43",
  "message": "Device updated successfully"
}
```

#### Error Cases

| Error                              | Cause                                       |
|------------------------------------|---------------------------------------------|
| `Authentication required`          | No valid session token                       |
| `Device not found or access denied`| Device not owned by user                     |

---

### 6.4 unregister_device

Removes a device from the authenticated user's account. The device can be re-registered by the same or a different user afterward.

| Property        | Value                          |
|-----------------|--------------------------------|
| **Type**        | PostgreSQL RPC Function         |
| **Auth**        | Required (authenticated user)   |
| **Ownership**   | User must own the device        |

#### Parameters

| Name          | Type | Required | Description                 |
|---------------|------|----------|-----------------------------|
| `p_device_id` | TEXT | Yes      | Device serial number to remove |

#### Request Example

```javascript
const { data, error } = await supabase.rpc('unregister_device', {
  p_device_id: '582D34712B99'
});
```

#### Response Schema

```json
{
  "success": true,
  "device_id": "582D34712B99",
  "message": "Device unregistered successfully"
}
```

#### Error Cases

| Error                              | Cause                                       |
|------------------------------------|---------------------------------------------|
| `Authentication required`          | No valid session token                       |
| `Device not found or access denied`| Device not owned by user                     |

---

## 7. API Functions - Air Quality Map

### 7.1 get_air_quality_map

Retrieves air quality data for all **public** devices (where `is_public = true` and coordinates are set). Intended for rendering a simple public air quality map. Requires authentication.

| Property        | Value                          |
|-----------------|--------------------------------|
| **Type**        | PostgreSQL RPC Function         |
| **Auth**        | Required (authenticated user)   |

#### Parameters

None.

#### Request Example

```javascript
const { data, error } = await supabase.rpc('get_air_quality_map');
```

#### Response Schema

```json
[
  {
    "latitude": 13.7563,
    "longitude": 100.5018,
    "pm25": 12.3,
    "pm10": 18.7,
    "co2": 450,
    "temperature": 28.5,
    "humidity": 65.2,
    "last_updated": "2026-02-15T10:30:00.000Z"
  }
]
```

#### Response Fields

| Field          | Type            | Description                                     |
|----------------|-----------------|--------------------------------------------------|
| `latitude`     | DOUBLE PRECISION| Device GPS latitude                              |
| `longitude`    | DOUBLE PRECISION| Device GPS longitude                             |
| `pm25`         | REAL            | Latest PM2.5 reading (µg/m³)                    |
| `pm10`         | REAL            | Latest PM10 reading (µg/m³)                     |
| `co2`          | INTEGER         | Latest CO2 reading (ppm)                         |
| `temperature`  | REAL            | Latest temperature (°C)                          |
| `humidity`     | REAL            | Latest humidity (%)                              |
| `last_updated` | TIMESTAMPTZ     | Timestamp of the latest reading                  |

#### Error Cases

| Error                                           | HTTP Code | Cause                    |
|-------------------------------------------------|-----------|--------------------------|
| `Authentication required to view air quality map`| 400       | No valid session token   |
| Empty array `[]`                                 | 200       | No public devices with coordinates |

---

### 7.2 get_public_air_quality

Retrieves air quality data for devices that have GPS coordinates. This is a **public endpoint**  - no authentication required. Designed for the public-facing air quality map.

By default (`p_public_only = true`), returns only devices where `is_public = true`. Pass `p_public_only = false` to include all devices with coordinates (original behavior).

Provides different levels of detail based on device privacy settings:
- **All devices with coordinates**: Location, sensor readings, device name, data age
- **Public devices only (is_public = true)**: Additionally shows owner name, fan status, and online indicator

| Property        | Value                            |
|-----------------|----------------------------------|
| **Type**        | PostgreSQL RPC Function           |
| **Auth**        | **Not required** (public access)  |

#### Parameters

| Name             | Type    | Required | Default | Description                                      |
|------------------|---------|----------|---------|--------------------------------------------------|
| `p_public_only`  | BOOLEAN | No       | `true`  | If true, return only `is_public = true` devices. If false, return all devices with coordinates. |

#### Request Example

```javascript
// Default: public devices only (recommended for map)
const { data, error } = await supabase.rpc('get_public_air_quality');

// Explicit: same as default
const { data, error } = await supabase.rpc('get_public_air_quality', {
  p_public_only: true
});

// All devices with coordinates (original behavior)
const { data, error } = await supabase.rpc('get_public_air_quality', {
  p_public_only: false
});
```

#### Response Schema

> **Note:** The default call `get_public_air_quality()` returns **only public devices** (`is_public = true`). The example below shows `p_public_only: false` to illustrate the visibility difference between public and non-public devices.

**Default call** `get_public_air_quality()` — returns only public devices:

```json
[
  {
    "device_id": "582D34712B43",
    "device_name": "Bedroom",
    "latitude": 13.7563,
    "longitude": 100.5018,
    "is_public": true,
    "pm25": 12.3,
    "pm10": 18.7,
    "co2": 450,
    "temperature": 28.5,
    "humidity": 65.2,
    "age_seconds": 120.5,
    "display_name": "John Doe",
    "fan_speed": 45,
    "fan_mode": "automatic",
    "fan_state": "running",
    "is_online": true
  }
]
```

**With** `p_public_only: false` — returns all devices (public + non-public with limited fields):

```json
[
  {
    "device_id": "582D34712B43",
    "device_name": "Bedroom",
    "latitude": 13.7563,
    "longitude": 100.5018,
    "is_public": true,
    "pm25": 12.3,
    "pm10": 18.7,
    "co2": 450,
    "temperature": 28.5,
    "humidity": 65.2,
    "age_seconds": 120.5,
    "display_name": "John Doe",
    "fan_speed": 45,
    "fan_mode": "automatic",
    "fan_state": "running",
    "is_online": true
  },
  {
    "device_id": "582D34712B59",
    "device_name": "Office",
    "latitude": 13.7580,
    "longitude": 100.5035,
    "is_public": false,
    "pm25": 18.1,
    "pm10": 25.3,
    "co2": 520,
    "temperature": 29.0,
    "humidity": 60.8,
    "age_seconds": 180.2,
    "display_name": null,
    "fan_speed": null,
    "fan_mode": null,
    "fan_state": null,
    "is_online": null
  }
]
```

#### Response Fields

| Field          | Type            | Visibility     | Description                                      |
|----------------|-----------------|----------------|--------------------------------------------------|
| `device_id`    | TEXT            | All devices    | Device serial number                             |
| `device_name`  | TEXT            | All devices    | User-assigned device name                        |
| `latitude`     | DOUBLE PRECISION| All devices    | GPS latitude                                     |
| `longitude`    | DOUBLE PRECISION| All devices    | GPS longitude                                    |
| `is_public`    | BOOLEAN         | All devices    | Whether device is public                         |
| `pm25`         | REAL            | All devices    | Latest PM2.5 (µg/m³)                            |
| `pm10`         | REAL            | All devices    | Latest PM10 (µg/m³)                             |
| `co2`          | INTEGER         | All devices    | Latest CO2 (ppm)                                 |
| `temperature`  | REAL            | All devices    | Latest temperature (°C)                          |
| `humidity`     | REAL            | All devices    | Latest humidity (%)                              |
| `age_seconds`  | DOUBLE PRECISION| All devices    | Seconds since last reading                       |
| `display_name` | TEXT            | Public only    | Device owner's display name (NULL if not public) |
| `fan_speed`    | INTEGER         | Public only    | Fan speed (NULL if not public)                   |
| `fan_mode`     | TEXT            | Public only    | Fan mode (NULL if not public)                    |
| `fan_state`    | TEXT            | Public only    | Fan state (NULL if not public)                   |
| `is_online`    | BOOLEAN         | Public only    | Online if status within 10 min (NULL if not public) |

#### Error Cases

| Error              | HTTP Code | Cause                            |
|--------------------|-----------|----------------------------------|
| Empty array `[]`   | 200       | No devices have GPS coordinates  |

---

## 8. API Functions - Fan Control

### 8.1 fan_control (Edge Function)

Sends a command to control a fan device via MQTT. The Edge Function verifies user authentication, checks device ownership, then publishes the command to EMQX Cloud which forwards it to the ESP32 device.

| Property        | Value                              |
|-----------------|------------------------------------|
| **Type**        | Supabase Edge Function (HTTP POST) |
| **Auth**        | Required (Bearer token)            |
| **Ownership**   | User must own the device           |
| **Status**      | Active (deployed 15 Feb 2026)    |

#### Endpoint

```
POST https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/fan-control
```

#### Headers

```
Authorization: Bearer <access_token>
Content-Type: application/json
```

#### Request Body

| Field       | Type   | Required | Description                                      |
|-------------|--------|----------|--------------------------------------------------|
| `device_id` | STRING | Yes      | Device serial number                             |
| `command`   | STRING | Yes      | Command type: `"mode"` or `"speed"`              |
| `value`     | STRING | Yes      | Command value (see below)                        |

**Command values:**

| Command  | Valid Values                    | Description                        |
|----------|---------------------------------|------------------------------------|
| `mode`   | `"automatic"`, `"custom"`       | Set fan control mode               |
| `speed`  | `"0"` to `"100"`                | Set fan speed percentage           |

#### Request Example - Set Mode

```javascript
const { data: { session } } = await supabase.auth.getSession();

const response = await fetch(
  'https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/fan-control',
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${session.access_token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      device_id: '582D34712B43',
      command: 'mode',
      value: 'custom'
    })
  }
);

const result = await response.json();
```

#### Request Example - Set Speed

```javascript
const response = await fetch(
  'https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/fan-control',
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${session.access_token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      device_id: '582D34712B43',
      command: 'speed',
      value: '45'
    })
  }
);
```

#### Success Response (200)

```json
{
  "success": true,
  "device_id": "582D34712B43",
  "device_name": "Bedroom",
  "command": "speed",
  "value": "45",
  "topic": "/be-tpp/582D34712B43/fan/speed",
  "timestamp": "2026-02-15T10:30:00.000Z",
  "db_synced": true,
  "message": "Fan speed set to 45 for Bedroom"
}
```

#### Error Responses

| HTTP Code | Error                                         | Cause                                    |
|-----------|-----------------------------------------------|------------------------------------------|
| 401       | `Missing authorization header`                | No `Authorization` header                |
| 401       | `Unauthorized`                                | Invalid or expired token                 |
| 400       | `device_id is required`                       | Missing `device_id` in body              |
| 400       | `command is required (mode or speed)`         | Missing `command` in body                |
| 400       | `command must be 'mode' or 'speed'`           | Invalid command type                     |
| 400       | `mode value must be 'automatic' or 'custom'`  | Invalid mode value                       |
| 400       | `speed value must be 0-100`                   | Speed out of range or not a number       |
| 403       | `Device not found or access denied`           | Device not owned by user                 |
| 405       | `Method not allowed`                          | Not a POST request                       |
| 500       | `Internal server error`                       | MQTT connection or publish failure       |

#### MQTT Topics

The Edge Function publishes to the following MQTT topics:

| Command | Topic Pattern                         | Payload Example |
|---------|---------------------------------------|-----------------|
| `mode`  | `/be-tpp/{device_id}/fan/mode`       | `"automatic"`   |
| `speed` | `/be-tpp/{device_id}/fan/speed`      | `"45"`          |

#### Fan Control Flow (v2.1)

```
1. Web App sends POST to Edge Function            ~0ms
2. Edge Function validates auth + ownership        ~200ms
3. Edge Function publishes to EMQX via MQTT/WSS    ~300ms
4. Edge Function INSERTs expected status to DB      ~100ms
5. Response: "Command sent" + db_synced returned    ~600ms total
6. Supabase Realtime notifies subscribed browser    ~100ms
7. UI shows "Device confirmed" (from DB INSERT)     ~700ms total
8. EMQX delivers command to ESP32                   ~100ms
9. ESP32 adjusts fan + publishes full status        ~100ms
10. EMQX Rule 2 (5-min sampling) inserts to DB      next cycle
    Total command feedback: ~0.7 seconds (from DB)
    Total ESP32 confirmation: ~2.5-3 seconds (from ESP32 status)
```

> **Note:** In v2.1, the Edge Function writes directly to `device_status` after MQTT publish succeeds. This triggers Supabase Realtime immediately without waiting for ESP32 to respond. The ESP32's full status (with firmware, RSSI, heap, etc.) arrives later via EMQX Rule 2 sampling.

---

### 8.2 Realtime Subscription (device_status)

Subscribe to real-time INSERT events on the `device_status` table to receive instant feedback when fan commands are executed or when ESP32 devices report their status.

| Property        | Value                                      |
|-----------------|--------------------------------------------|
| **Type**        | Supabase Realtime (PostgreSQL Changes)      |
| **Auth**        | Required (authenticated user with session)  |
| **RLS**         | User sees only own devices (via RLS SELECT) |
| **Prerequisite**| `device_status` must be in `supabase_realtime` publication |

#### Setup (one-time)

The `device_status` table must be added to the Supabase Realtime publication:

```sql
ALTER PUBLICATION supabase_realtime ADD TABLE device_status;
```

#### Subscribe Example

```javascript
// Subscribe to status changes for a specific device
const channel = supabase.channel('device-status-582D34712B43')
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'device_status',
      filter: 'device_id=eq.582D34712B43'
    },
    (payload) => {
      console.log('New status:', payload.new);
      // payload.new contains: fan_speed, fan_mode, fan_state, fw_ver, rssi, etc.
    }
  )
  .subscribe((status) => {
    if (status === 'SUBSCRIBED') {
      console.log('Realtime connected');
    }
  });
```

#### Unsubscribe Example

```javascript
// Unsubscribe when changing device or logging out
supabase.removeChannel(channel);
```

#### Event Payload

When a new row is inserted into `device_status`, the payload contains:

| Field        | Type     | Description                                  |
|--------------|----------|----------------------------------------------|
| `device_id`  | TEXT     | Device serial number                         |
| `client_id`  | TEXT     | ESP32 MAC-based client ID                    |
| `fan_speed`  | INTEGER  | Fan speed 0-100 (null if not in this update) |
| `fan_mode`   | TEXT     | "automatic" or "custom" (null if not set)    |
| `fan_state`  | TEXT     | "running", "idle", "off", "boost" (null if partial) |
| `fw_ver`     | TEXT     | Firmware version (null if partial update)    |
| `rssi`       | INTEGER  | WiFi signal strength in dBm                  |
| `heap`       | INTEGER  | Free heap memory in bytes                    |
| `uptime`     | INTEGER  | Device uptime in seconds                     |
| `time`       | TIMESTAMPTZ | Timestamp of the status report            |

#### Event Sources

There are two types of INSERT events:

| Source              | Trigger                        | Fields Populated          |
|---------------------|--------------------------------|---------------------------|
| Edge Function v2.1  | After fan command MQTT publish | `fan_speed` OR `fan_mode` only (partial row) |
| ESP32 via EMQX Rule 2 | Every 5 min (sampled from 30s) | All fields (full status row) |

#### Lifecycle Best Practices

```
Login  -> loadDevices() -> subscribeRealtime(firstDevice)
Change device -> unsubscribe old -> subscribeRealtime(newDevice)
Logout -> unsubscribeRealtime()
```

> **Tip:** Use a duplicate guard to prevent re-subscribing to the same device when `loadDevices()` is called multiple times (e.g., from both `getSession()` and `onAuthStateChange`).

---

## 9. API Functions - Profile Management

Profile data is accessed directly via the `profiles` table using Supabase client methods (not RPC functions). RLS policies ensure users can only read and update their own profile.

### 9.1 Get Profile

Retrieves the authenticated user's profile.

| Property        | Value                          |
|-----------------|--------------------------------|
| **Type**        | Supabase table query            |
| **Auth**        | Required (authenticated user)   |
| **RLS**         | User can only read own profile  |

#### Request Example

```javascript
const { data: { user } } = await supabase.auth.getUser();

const { data, error } = await supabase
  .from('profiles')
  .select('*')
  .eq('id', user.id)
  .single();
```

#### Response Schema

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "email": "user@example.com",
  "display_name": "John Doe",
  "phone": "+66812345678",
  "address": "123 Main St, Bangkok",
  "created_at": "2026-01-30T08:00:00.000Z",
  "updated_at": "2026-02-15T10:00:00.000Z"
}
```

#### Response Fields

| Field          | Type        | Description                          |
|----------------|-------------|--------------------------------------|
| `id`           | UUID        | User ID (matches auth.users.id)     |
| `email`        | TEXT        | User's email address                 |
| `display_name` | TEXT        | Display name (nullable)              |
| `phone`        | TEXT        | Phone number (nullable)              |
| `address`      | TEXT        | Address (nullable)                   |
| `created_at`   | TIMESTAMPTZ | Account creation time                |
| `updated_at`   | TIMESTAMPTZ | Last profile update time             |

---

### 9.2 Update Profile

Updates the authenticated user's profile fields.

| Property        | Value                            |
|-----------------|----------------------------------|
| **Type**        | Supabase table update             |
| **Auth**        | Required (authenticated user)     |
| **RLS**         | User can only update own profile  |

#### Request Example

```javascript
const { data: { user } } = await supabase.auth.getUser();

const { data, error } = await supabase
  .from('profiles')
  .update({
    display_name: 'John Doe',
    phone: '+66812345678',
    address: '123 Main St, Bangkok'
  })
  .eq('id', user.id)
  .select()
  .single();
```

#### Error Cases

| Error                | HTTP Code | Cause                                  |
|----------------------|-----------|----------------------------------------|
| RLS policy violation | 403       | Attempting to update another user's profile |

> **Note:** Profiles are automatically created when a new user signs up via the `handle_new_user` trigger. You do not need to manually INSERT a profile row.

---

## 10. API Functions - Outdoor Air Quality

> **⚠ DEPRECATION NOTICE (v1.5, 13 Apr 2026)**
>
> `get-outdoor-air` (Section 10.1) and `get-outdoor-air-batch` (Section 10.2) are **deprecated**.
> These per-device Edge Functions will continue to function but are no longer recommended.
>
> **Replacement:** Use `aqicn_stations` REST API (Section 10.3) which provides pre-fetched
> AQICN data for 470 stations nationwide, updated every hour via pg_cron.
> Use `haversine()` on the client side to find the nearest station.
>
> See **Migration Guide** (`BE-TPP_Migration_Guide_aqicn_stations.md`) for transition details.

### 10.1 get-outdoor-air (Edge Function)

Retrieves outdoor air quality data from the AQICN API via a backend proxy. Uses geo zone caching (1 hour) to minimize API calls. Accepts either a `device_id` (uses device's registered coordinates) or explicit `latitude`/`longitude`.

| Property        | Value                              |
|-----------------|------------------------------------|
| **Type**        | Supabase Edge Function (HTTP POST) |
| **Auth**        | Required (Bearer token)            |
| **Cache**       | Geo zone cache, 1 hour TTL         |
| **Data Source**  | AQICN (World Air Quality Index)   |
| **Status**      | ⚠ **DEPRECATED** (since v1.5, 13 Apr 2026) — Use `aqicn_stations` REST API instead |

#### Endpoint

```
POST https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/get-outdoor-air
```

#### Headers

```
Authorization: Bearer <access_token>
Content-Type: application/json
```

#### Request Body

Send **either** `device_id` **or** `latitude` + `longitude`:

| Field       | Type   | Required | Description                                      |
|-------------|--------|----------|--------------------------------------------------|
| `device_id` | STRING | Option A | Device serial number (uses device's lat/lng from user_devices) |
| `latitude`  | NUMBER | Option B | GPS latitude (-90 to 90)                         |
| `longitude` | NUMBER | Option B | GPS longitude (-180 to 180)                      |

#### Request Example - Using Device ID

```javascript
const { data: { session } } = await supabase.auth.getSession();

const response = await fetch(
  'https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/get-outdoor-air',
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${session.access_token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      device_id: '582D34712B43'
    })
  }
);

const result = await response.json();
```

#### Request Example - Using Custom Coordinates

```javascript
const response = await fetch(
  'https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/get-outdoor-air',
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${session.access_token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      latitude: 18.7883,
      longitude: 98.9853
    })
  }
);
```

#### Success Response (200)

```json
{
  "success": true,
  "source": "aqicn_api",
  "cache_age_minutes": 0,
  "geo_zone": "18.79_98.99",
  "device_id": "582D34712B43",
  "device_name": "H&T Living Room",
  "db_synced": true,
  "data": {
    "aqi": 158,
    "pm25": 158,
    "pm10": 91,
    "o3": 14.6,
    "no2": 7.5,
    "so2": 0.6,
    "co": null,
    "temperature": 35,
    "humidity": 24,
    "wind": 3,
    "dominant_pollutant": "pm25",
    "station_name": "City Hall, Chiangmai, Thailand",
    "station_url": "https://aqicn.org/city/thailand/chiangmai/city-hall"
  },
  "timestamp": "2026-03-27T06:37:17.111Z"
}
```

#### Response Fields

| Field               | Type    | Description                                          |
|---------------------|---------|------------------------------------------------------|
| `success`           | BOOLEAN | Always `true` on success                             |
| `source`            | TEXT    | `"aqicn_api"` (fresh) or `"cache"` (from DB cache)  |
| `cache_age_minutes` | INTEGER | Minutes since cache was stored (0 = fresh from API)  |
| `geo_zone`          | TEXT    | Cache key, e.g. `"18.79_98.99"` (lat/lng rounded to 2 decimals, ~1.1 km grid) |
| `device_id`         | TEXT    | Device serial number (null if using custom lat/lng)  |
| `device_name`       | TEXT    | Device name (null if using custom lat/lng)           |
| `db_synced`         | BOOLEAN | Whether the reading was saved to DB cache            |
| `data.aqi`          | INTEGER | AQI index (US EPA standard)                          |
| `data.pm25`         | REAL    | PM2.5 concentration                                  |
| `data.pm10`         | REAL    | PM10 concentration                                   |
| `data.o3`           | REAL    | Ozone (null if unavailable)                          |
| `data.no2`          | REAL    | NO2 (null if unavailable)                            |
| `data.so2`          | REAL    | SO2 (null if unavailable)                            |
| `data.co`           | REAL    | CO (null if unavailable)                             |
| `data.temperature`  | REAL    | Temperature from station (C)                         |
| `data.humidity`     | REAL    | Humidity from station (%)                            |
| `data.wind`         | REAL    | Wind speed (m/s)                                     |
| `data.dominant_pollutant` | TEXT | Primary pollutant, e.g. `"pm25"`, `"o3"`       |
| `data.station_name` | TEXT    | Nearest AQICN station name                           |
| `data.station_url`  | TEXT    | URL to AQICN station page                            |
| `timestamp`         | TEXT    | ISO 8601 timestamp of the reading                    |

#### AQI Levels Reference

| AQI Range | Level                    | Color    |
|-----------|--------------------------|----------|
| 0-50      | Good                     | Green    |
| 51-100    | Moderate                 | Yellow   |
| 101-150   | Unhealthy for Sensitive  | Orange   |
| 151-200   | Unhealthy                | Red      |
| 201-300   | Very Unhealthy           | Purple   |
| 301+      | Hazardous                | Maroon   |

#### Geo Zone Cache Logic

```
1. Round lat/lng to 2 decimal places (~1.1 km grid)
2. Create geo_zone key: "{lat_rounded}_{lng_rounded}"
3. Check outdoor_air_readings for matching geo_zone within 1 hour
4. Cache HIT  -> return cached data (no AQICN API call)
5. Cache MISS -> call AQICN API -> store in DB -> return fresh data
```

#### Error Responses

| HTTP Code | Error                                                        | Cause                                    |
|-----------|--------------------------------------------------------------|------------------------------------------|
| 401       | `Missing authorization header`                               | No `Authorization` header                |
| 401       | `Unauthorized`                                               | Invalid or expired token                 |
| 400       | `Either device_id or latitude+longitude is required`         | No parameters provided                   |
| 400       | `Device has no coordinates. Please update device location first.` | Device lat/lng is null              |
| 400       | `Invalid latitude or longitude values`                       | Non-numeric values                       |
| 400       | `Coordinates out of range (lat: -90 to 90, lng: -180 to 180)` | Values exceed valid range             |
| 403       | `Device not found or access denied`                          | Device not owned by user                 |
| 405       | `Method not allowed`                                         | Not a POST request                       |
| 502       | `AQICN API request failed`                                   | AQICN API is down                        |
| 502       | `AQICN API returned error`                                   | AQICN token invalid or other API error   |
| 500       | `Internal server error`                                      | Unexpected error                         |

---

### 10.2 get-outdoor-air-batch (Edge Function)

Retrieves outdoor air quality data from the AQICN API for **multiple devices** in a single request. Supports two modes: user's own devices (`my_devices`) or all devices with coordinates (`public`). Uses geo zone deduplication and caching to minimize API calls.

| Property        | Value                              |
|-----------------|------------------------------------|
| **Type**        | Supabase Edge Function (HTTP POST) |
| **Auth**        | Required (Bearer token)            |
| **Cache**       | Geo zone cache, 1 hour TTL         |
| **Data Source**  | AQICN (World Air Quality Index)   |
| **Limits**      | Max 100 devices, max 50 unique zones |
| **AQICN Calls** | Sequential (not parallel)          |
| **Status**      | ⚠ **DEPRECATED** (since v1.5, 13 Apr 2026) — Use `aqicn_stations` REST API instead |

#### Endpoint

```
POST https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/get-outdoor-air-batch
```

#### Headers

```
Authorization: Bearer <access_token>
Content-Type: application/json
```

#### Request Body

| Field         | Type    | Required | Default | Description                                      |
|---------------|---------|----------|---------|--------------------------------------------------|
| `source`      | STRING  | Yes      |  --        | `"my_devices"` or `"public"`                     |
| `public_only` | BOOLEAN | No       | `false` | If true, filter only `is_public = true` devices (only applies when source = "public") |

#### Mode A: my_devices

Returns outdoor AQI for all devices registered to the authenticated user that have coordinates.

```javascript
const response = await fetch(
  'https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/get-outdoor-air-batch',
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${session.access_token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      source: 'my_devices'
    })
  }
);
```

#### Mode B: public (for map)

Returns outdoor AQI for all devices with coordinates. Use `public_only: true` for the public-facing map.

```javascript
const response = await fetch(
  'https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/get-outdoor-air-batch',
  {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${session.access_token}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      source: 'public',
      public_only: true
    })
  }
);
```

#### Success Response (200)

```json
{
  "success": true,
  "source": "public",
  "public_only": true,
  "total_devices": 3,
  "total_zones": 3,
  "summary": {
    "cache_hits": 1,
    "api_calls": 2,
    "errors": 0
  },
  "results": [
    {
      "device_id": "582D34712B43",
      "device_name": "Living Room",
      "latitude": 13.7440,
      "longitude": 100.5753,
      "is_public": true,
      "geo_zone": "13.74_100.58",
      "source": "aqicn_api",
      "cache_age_minutes": 0,
      "data": {
        "aqi": 118,
        "pm25": 118,
        "pm10": 60,
        "o3": 4.8,
        "no2": 15,
        "so2": null,
        "co": null,
        "temperature": 37,
        "humidity": 32,
        "wind": 3,
        "dominant_pollutant": "pm25",
        "station_name": "National Housing Authority Dindaeng, Bangkok",
        "station_url": "https://aqicn.org/city/thailand/bangkok/national-housing-authority-dindaeng"
      },
      "timestamp": "2026-04-06T08:06:19.511Z",
      "db_synced": true
    },
    {
      "device_id": "582D34712B40",
      "device_name": "PTIS JS office",
      "latitude": 18.9145,
      "longitude": 98.9094,
      "is_public": true,
      "geo_zone": "18.91_98.91",
      "source": "cache",
      "cache_age_minutes": 5,
      "data": {
        "aqi": 171,
        "pm25": 171,
        "pm10": 91,
        "...": "..."
      },
      "timestamp": "2026-04-06T08:06:01.768+00:00",
      "db_synced": true
    }
  ]
}
```

#### Response Fields (Top Level)

| Field            | Type    | Description                                      |
|------------------|---------|--------------------------------------------------|
| `success`        | BOOLEAN | Always `true` on success                         |
| `source`         | TEXT    | `"my_devices"` or `"public"`                     |
| `public_only`    | BOOLEAN | Whether public-only filter was applied           |
| `total_devices`  | INTEGER | Number of devices in results                     |
| `total_zones`    | INTEGER | Number of unique geo zones queried               |
| `summary`        | OBJECT  | Breakdown of cache hits, API calls, and errors   |
| `results`        | ARRAY   | Per-device results with outdoor AQI data         |

#### Response Fields (Per Device in results[])

| Field              | Type    | Description                                      |
|--------------------|---------|--------------------------------------------------|
| `device_id`        | TEXT    | Device serial number                             |
| `device_name`      | TEXT    | User-assigned device name                        |
| `latitude`         | NUMBER  | Device GPS latitude                              |
| `longitude`        | NUMBER  | Device GPS longitude                             |
| `is_public`        | BOOLEAN | Whether device is public                         |
| `geo_zone`         | TEXT    | Cache key, e.g. `"18.91_98.91"`                 |
| `source`           | TEXT    | `"cache"` or `"aqicn_api"`                       |
| `cache_age_minutes`| INTEGER | Minutes since cache was stored (0 = fresh)       |
| `data`             | OBJECT  | AQI data (same fields as get-outdoor-air)        |
| `timestamp`        | TEXT    | ISO 8601 timestamp of the reading                |
| `db_synced`        | BOOLEAN | Whether data was saved to database               |
| `error`            | TEXT    | Error message if this zone failed (optional)     |

#### Geo Zone Deduplication

```
1. Collect all devices with lat/lng
2. Round each device's lat/lng to 2 decimal places (~1.1 km grid)
3. Group devices by geo_zone key
4. For each unique zone (sequential, not parallel):
   a. Check cache (outdoor_air_readings, < 1 hour)
   b. Cache HIT  -> reuse cached data
   c. Cache MISS -> call AQICN API -> store in DB
5. Map zone results back to individual devices
```

Example: 10 devices in 4 unique zones = max 4 AQICN API calls (not 10).

#### Error Responses

| HTTP Code | Error                                                        | Cause                                    |
|-----------|--------------------------------------------------------------|------------------------------------------|
| 401       | `Missing authorization header`                               | No `Authorization` header                |
| 401       | `Unauthorized`                                               | Invalid or expired token                 |
| 400       | `Invalid or missing 'source' parameter`                      | source is not "my_devices" or "public"   |
| 400       | `Too many devices (N). Maximum is 100.`                      | Exceeds device limit                     |
| 400       | `Too many unique geo zones (N). Maximum is 50.`              | Exceeds zone limit                       |
| 405       | `Method not allowed`                                         | Not a POST request                       |
| 500       | `Internal server error`                                      | Unexpected error                         |

> **Note:** Individual zone errors (e.g. AQICN API failure) do not cause the entire request to fail. Instead, affected devices will have an `error` field in their result while other devices return normally.

---

### 10.3 aqicn_stations (REST Query) ★ NEW

Pre-fetched AQICN air quality data for ~470 stations across Thailand and neighboring countries. Updated automatically every hour by the `fetch-aqicn-map` Edge Function via pg_cron. This is the **recommended replacement** for `get-outdoor-air` and `get-outdoor-air-batch`.

| Property        | Value                              |
|-----------------|------------------------------------|
| **Type**        | Supabase REST API (HTTP GET)       |
| **Auth**        | anon key (no login required)       |
| **Data Source**  | AQICN map/bounds API              |
| **Update**      | pg_cron every hour (:30)           |
| **Status**      | ✅ Active (deployed 13 Apr 2026)   |

#### Endpoint

```
GET https://brgzimwzcfbwkgymqzvy.supabase.co/rest/v1/aqicn_stations?select=uid,station_name,latitude,longitude,aqi,measured_at,fetched_at&aqi=not.is.null
```

#### Headers

```
apikey: <SUPABASE_ANON_KEY>
```

#### Response (Array of Objects)

| Field            | Type        | Description                                    |
|------------------|-------------|------------------------------------------------|
| `uid`            | TEXT        | AQICN station unique ID (e.g. "@1234")         |
| `station_name`   | TEXT        | Station display name                           |
| `latitude`       | REAL        | GPS latitude                                   |
| `longitude`      | REAL        | GPS longitude                                  |
| `aqi`            | INTEGER     | AQI index (US EPA standard)                    |
| `measured_at`    | TIMESTAMPTZ | When station last reported                     |
| `fetched_at`     | TIMESTAMPTZ | When our system fetched the data               |

#### Example Response

```json
[
  {
    "uid": "@7397",
    "station_name": "Bangkok US Embassy",
    "latitude": 13.7563,
    "longitude": 100.5018,
    "aqi": 78,
    "measured_at": "2026-04-13T07:00:00+00:00",
    "fetched_at": "2026-04-13T07:30:12+00:00"
  },
  {
    "uid": "@10570",
    "station_name": "Chiang Mai",
    "latitude": 18.7883,
    "longitude": 98.9853,
    "aqi": 152,
    "measured_at": "2026-04-13T07:00:00+00:00",
    "fetched_at": "2026-04-13T07:30:12+00:00"
  }
]
```

#### Usage Notes

- Returns ~446 stations (those with non-null AQI out of ~470 total)
- Response size: ~53 KB — can be cached in client memory
- No login required — public data readable by anon key
- AQI is US EPA standard — use `aqiToPm25()` to convert to PM2.5 µg/m³ (approximate ±10%)
- Use `haversine()` on client side to find nearest station to a device
- Data freshness: updated every hour at :30 by pg_cron

#### Client-side Helper Functions

```javascript
// haversine — คำนวณระยะทาง (km)
function haversine(lat1, lon1, lat2, lon2) {
  var R = 6371, toRad = Math.PI / 180;
  var dLat = (lat2 - lat1) * toRad, dLon = (lon2 - lon1) * toRad;
  var a = Math.sin(dLat/2) * Math.sin(dLat/2) +
          Math.cos(lat1 * toRad) * Math.cos(lat2 * toRad) *
          Math.sin(dLon/2) * Math.sin(dLon/2);
  return R * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
}

// aqiToPm25 — แปลง AQI → PM2.5 µg/m³ (ค่าประมาณ)
function aqiToPm25(aqi) {
  if (aqi == null) return null;
  var bp = [
    [0, 50, 0, 12.0], [51, 100, 12.1, 35.4], [101, 150, 35.5, 55.4],
    [151, 200, 55.5, 150.4], [201, 300, 150.5, 250.4], [301, 500, 250.5, 500.4]
  ];
  for (var i = 0; i < bp.length; i++) {
    var b = bp[i];
    if (aqi >= b[0] && aqi <= b[1]) {
      return Math.round(((aqi - b[0]) / (b[1] - b[0]) * (b[3] - b[2]) + b[2]) * 10) / 10;
    }
  }
  return aqi > 500 ? 500.4 : null;
}
```

#### Migration from get-outdoor-air

**ก่อน (get-outdoor-air):**
```javascript
const { data } = await supabase.functions.invoke('get-outdoor-air', {
  body: { device_id: 'xxx' }
});
// Returns: aqi, pm25, pm10, temperature, humidity, wind, station_name
```

**หลัง (aqicn_stations):**
```javascript
// ไม่ต้อง login — ใช้ anon key
const { data: stations } = await supabase
  .from('aqicn_stations')
  .select('uid,station_name,latitude,longitude,aqi,measured_at,fetched_at')
  .not('aqi', 'is', null);

// หาสถานีใกล้สุดจาก device
const nearest = findNearestStation(deviceLat, deviceLng, stations);
if (nearest && nearest.distance <= 50) {
  const aqi = nearest.station.aqi;
  const pm25 = aqiToPm25(aqi);  // ≈ ค่าประมาณ
}
```

**สิ่งที่เปลี่ยน:**

| รายการ | ก่อน | หลัง |
|--------|------|------|
| Auth | Bearer token (ต้อง login) | anon key (ไม่ต้อง login) |
| API call | 1 call ต่อ device | 1 call ได้ทุกสถานี |
| Nearest station | Backend คำนวณ (geo_zone) | Frontend คำนวณ (haversine) |
| ข้อมูลที่ได้ | AQI, PM2.5, PM10, Temp, Humid, Wind | AQI เท่านั้น (PM2.5 ต้อง aqiToPm25) |
| ความสด | Cache 1 ชม. (may older) | pg_cron ทุก 1 ชม. (:30) |

**ข้อมูลที่หาย (ไม่ critical):**
- Temperature, Humidity, Wind → ใช้ Thai Air (CUSense) แทนได้ (มี Temp, Humidity)
- PM2.5 ค่าจริง → ใช้ `aqiToPm25()` แปลงจาก AQI (ค่าประมาณ ±10%)
- PM10, O3, NO2, SO2, CO → ไม่มีจาก map/bounds API

**Timeline:**

| วันที่ | สถานะ |
|--------|-------|
| 13 เม.ย. 2026 | aqicn_stations พร้อมใช้ + API Doc v1.5 |
| — | ทีมแอปเริ่ม migration |
| +30 วัน (ประมาณ) | ตรวจสอบว่าทุกคนเปลี่ยนแล้ว |
| +60 วัน (ประมาณ) | ลบ get-outdoor-air + get-outdoor-air-batch |

---

## 11. API Functions - Thai Air Quality (Nationwide)

ข้อมูลคุณภาพอากาศทั่วประเทศไทย จาก 2 แหล่ง: **Air4Thai** (กรมควบคุมมลพิษ, 186 สถานี) + **CUSense** (จุฬาลงกรณ์, ~59 สถานี) รวม **245 สถานี** อัปเดตทุก 1 ชั่วโมง ผ่าน Edge Function `fetch-thai-air` (pg_cron)

| Property        | Value                              |
|-----------------|------------------------------------|
| **Data Sources** | Air4Thai (PCD) + CUSense (Chula)  |
| **Stations**    | 245 (186 Air4Thai + ~59 CUSense)   |
| **Update**      | Every hour (pg_cron at xx:05)      |
| **Retention**   | 90 days (pg_cron cleanup 03:30 UTC)|
| **Auth**        | Public read (anon key), Admin write|
| **Status**      | Active (deployed 9-10 Apr 2026)    |

### 11.1 v_thai_air_latest (View — REST Query)

ดึงข้อมูลล่าสุดของทุกสถานี (JOIN stations + readings ล่าสุด + คำนวณ aqi_level)
เป็น **public view** ไม่ต้อง login — ใช้ anon key ได้เลย

#### Endpoint

```
GET https://brgzimwzcfbwkgymqzvy.supabase.co/rest/v1/v_thai_air_latest?select=*
```

#### Headers

```
apikey: <SUPABASE_ANON_KEY>
```

#### Filter Examples

```
// ดึงทุกสถานี (245 rows)
GET .../rest/v1/v_thai_air_latest?select=*

// ดึงเฉพาะจังหวัด
GET .../rest/v1/v_thai_air_latest?select=*&province=eq.อุดรธานี

// ดึงเฉพาะ PM2.5 > 50 เรียงจากมากไปน้อย
GET .../rest/v1/v_thai_air_latest?select=*&pm25=gt.50&order=pm25.desc

// ดึงเฉพาะ source
GET .../rest/v1/v_thai_air_latest?select=*&source=eq.cusense
```

#### Request Example (JavaScript)

```javascript
const { data, error } = await supabase
  .from('v_thai_air_latest')
  .select('*');

// Filter by province
const { data, error } = await supabase
  .from('v_thai_air_latest')
  .select('*')
  .eq('province', 'อุดรธานี');

// Filter PM2.5 > 50
const { data, error } = await supabase
  .from('v_thai_air_latest')
  .select('*')
  .gt('pm25', 50)
  .order('pm25', { ascending: false });
```

#### Success Response (200)

```json
{
  "station_id": 1,
  "source": "air4thai",
  "source_station_id": "119t",
  "name": "สวนสาธารณะธารา",
  "name_en": "Thara Public Park",
  "latitude": 8.0506237,
  "longitude": 98.9180489,
  "province": "กระบี่",
  "amphoe": null,
  "station_type": "GROUND",
  "pm25": 18.3,
  "pm10": null,
  "pm1": null,
  "aqi": 34,
  "temperature": null,
  "humidity": null,
  "measured_at": "2026-04-09T16:00:00+07:00",
  "fetched_at": "2026-04-10T09:05:04.086+00:00",
  "aqi_level": "good"
}
```

#### Response Fields

| Field               | Type    | Description                                          |
|---------------------|---------|------------------------------------------------------|
| `station_id`        | INTEGER | Internal station ID (FK to thai_air_stations)        |
| `source`            | TEXT    | `"air4thai"` or `"cusense"`                          |
| `source_station_id` | TEXT    | Original station ID from source API                  |
| `name`              | TEXT    | Station name (Thai)                                  |
| `name_en`           | TEXT    | Station name (English, Air4Thai only)                |
| `latitude`          | REAL    | GPS latitude                                         |
| `longitude`         | REAL    | GPS longitude                                        |
| `province`          | TEXT    | Province name (Thai)                                 |
| `amphoe`            | TEXT    | District name (CUSense only)                         |
| `station_type`      | TEXT    | Air4Thai: GROUND/BKK/MOBILE, CUSense: cusensor3/nansensor/cusensor2 |
| `pm25`              | REAL    | PM2.5 (µg/m³)                                       |
| `pm10`              | REAL    | PM10 (µg/m³)                                        |
| `pm1`               | REAL    | PM1.0 (CUSense only)                                |
| `aqi`               | INTEGER | AQI index (Air4Thai only)                            |
| `temperature`       | REAL    | Temperature (CUSense only)                           |
| `humidity`          | REAL    | Humidity (CUSense only)                              |
| `measured_at`       | TEXT    | When the source API recorded the data                |
| `fetched_at`        | TEXT    | When our system fetched the data                     |
| `aqi_level`         | TEXT    | Calculated AQI level based on PM2.5 (Thai standard) |

#### AQI Level (Thai Standard — based on PM2.5)

| aqi_level   | PM2.5 (µg/m³) | Color  | Meaning                      |
|-------------|----------------|--------|------------------------------|
| excellent   | 0 - 15.0       | Blue   | คุณภาพอากาศดีมาก            |
| good        | 15.1 - 25.0    | Green  | คุณภาพอากาศดี                |
| moderate    | 25.1 - 37.5    | Yellow | ปานกลาง                      |
| unhealthy   | 37.6 - 75.0    | Orange | มีผลกระทบต่อสุขภาพ          |
| hazardous   | > 75.0         | Red    | มีผลกระทบต่อสุขภาพมาก       |

---

### 11.2 thai_air_readings (History — REST Query)

ดึงประวัติข้อมูลรายชั่วโมงของสถานีเดียว (สำหรับกราฟ)

#### Endpoint

```
GET https://brgzimwzcfbwkgymqzvy.supabase.co/rest/v1/thai_air_readings?select=*&station_id=eq.{id}&order=measured_at.desc&limit=24
```

#### Headers

```
apikey: <SUPABASE_ANON_KEY>
```

#### Request Example (JavaScript)

```javascript
// ดึง 24 ชั่วโมงล่าสุดของสถานี ID 1
const { data, error } = await supabase
  .from('thai_air_readings')
  .select('*')
  .eq('station_id', 1)
  .order('measured_at', { ascending: false })
  .limit(24);
```

#### Success Response (200)

```json
{
  "id": 123,
  "station_id": 1,
  "pm25": 18.3,
  "pm10": 25.1,
  "pm1": null,
  "aqi": 34,
  "temperature": null,
  "humidity": null,
  "co2": null,
  "o3": 12.5,
  "no2": 5.2,
  "so2": 1.1,
  "co": 0.3,
  "measured_at": "2026-04-10T09:00:00+07:00",
  "fetched_at": "2026-04-10T09:05:04.086+00:00"
}
```

---

### 11.3 fetch-thai-air (Edge Function — Cron/Admin)

Edge Function สำหรับดึงข้อมูลจาก Air4Thai + CUSense และ sync ลง DB ทำงานผ่าน pg_cron ทุก 1 ชั่วโมง (นาทีที่ 5) ไม่ได้ออกแบบให้ Frontend เรียกโดยตรง

| Property        | Value                              |
|-----------------|------------------------------------|
| **Type**        | Supabase Edge Function (HTTP POST) |
| **Auth**        | service_role key or valid user JWT |
| **Schedule**    | pg_cron `5 * * * *` (ทุก 1 ชม.)  |
| **Performance** | ~2 seconds (batch operations)      |
| **Status**      | Active (Round 3A — 10 Apr 2026)    |

#### Endpoint

```
POST https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/fetch-thai-air
```

#### Manual Test (cURL)

```bash
curl -X POST 'https://brgzimwzcfbwkgymqzvy.supabase.co/functions/v1/fetch-thai-air' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer {SERVICE_ROLE_KEY}' \
  -d '{}'
```

#### Success Response (200)

```json
{
  "success": true,
  "fetched": {
    "air4thai": { "stations": 186, "readings": 186, "error": null },
    "cusense": { "stations": 58, "readings": 58, "error": null },
    "total_stations": 244,
    "total_readings": 244
  },
  "synced": {
    "stations_upserted": 244,
    "readings_inserted": 177,
    "readings_skipped_duplicate": 0,
    "errors": []
  },
  "elapsed_ms": 1918,
  "next_run": "ทุก 1 ชั่วโมง (pg_cron)"
}
```

#### Data Sources

| Source   | URL / Endpoint                                          | Auth              | Stations |
|----------|--------------------------------------------------------|-------------------|----------|
| Air4Thai | `http://air4thai.pcd.go.th/services/getNewAQI_JSON.php`| None (public)     | 186      |
| CUSense  | `https://www.cusense.net:8082/api/v1/sensorData/realtime/all` | X-Gravitee-Api-Key | ~59 (non-PCD) |

> **Note:** CUSense stations starting with `PCD/` (158 stations) are duplicates of Air4Thai data and are excluded. Only cusensor3/nansensor/cusensor2 projects (~59 stations) are fetched.

---

## 12. Database Schema

### Table: profiles

| Column       | Type        | Nullable | Default | Description                    |
|--------------|-------------|----------|---------|--------------------------------|
| `id`         | UUID (PK)   | No       |  --        | References auth.users(id)      |
| `email`      | TEXT        | Yes      | NULL    | User's email                   |
| `display_name`| TEXT       | Yes      | NULL    | Display name                   |
| `phone`      | TEXT        | Yes      | NULL    | Phone number                   |
| `address`    | TEXT        | Yes      | NULL    | Address                        |
| `created_at` | TIMESTAMPTZ | Yes      | NOW()   | Creation timestamp             |
| `updated_at` | TIMESTAMPTZ | Yes      | NOW()   | Last update timestamp          |

**Constraints:** `id` references `auth.users(id) ON DELETE CASCADE`

---

### Table: user_devices

| Column         | Type            | Nullable | Default | Description                      |
|----------------|-----------------|----------|---------|----------------------------------|
| `id`           | BIGSERIAL (PK)  | No       | Auto    | Internal row ID                  |
| `user_id`      | UUID (FK)       | No       |  --        | References auth.users(id)        |
| `device_id`    | TEXT            | No       |  --        | Device serial number             |
| `client_id`    | TEXT            | Yes      | NULL    | ESP32 MAC-based client ID        |
| `device_name`  | TEXT            | Yes      | NULL    | User-assigned name               |
| `latitude`     | DOUBLE PRECISION| Yes      | NULL    | GPS latitude                     |
| `longitude`    | DOUBLE PRECISION| Yes      | NULL    | GPS longitude                    |
| `is_public`    | BOOLEAN         | Yes      | false   | Show on public map               |
| `registered_at`| TIMESTAMPTZ     | Yes      | NOW()   | Registration timestamp           |
| `updated_at`   | TIMESTAMPTZ     | Yes      | NOW()   | Last update timestamp            |

**Constraints:**
- `user_id` references `auth.users(id) ON DELETE CASCADE`
- `UNIQUE (user_id, device_id)`  --  one user can register a device once; max 3 users per device (enforced in `register_device()` function)

---

### Table: sensor_readings

| Column       | Type        | Nullable | Default | Description                          |
|--------------|-------------|----------|---------|--------------------------------------|
| `id`         | BIGSERIAL (PK)| No     | Auto    | Internal row ID                      |
| `time`       | TIMESTAMPTZ | Yes      |  --        | Reading timestamp                    |
| `time_bucket`| TIMESTAMPTZ | Yes      |  --        | 5-minute bucket for deduplication    |
| `device_id`  | TEXT        | Yes      |  --        | Device serial number                 |
| `temperature`| REAL        | Yes      |  --        | Temperature (°C)                     |
| `humidity`   | REAL        | Yes      |  --        | Relative humidity (%)                |
| `pm25`       | REAL        | Yes      |  --        | PM2.5 (µg/m³)                       |
| `pm10`       | REAL        | Yes      |  --        | PM10 (µg/m³)                        |
| `co2`        | INTEGER     | Yes      |  --        | CO2 (ppm)                           |
| `tvoc_index` | INTEGER     | Yes      |  --        | TVOC index                          |
| `battery`    | INTEGER     | Yes      |  --        | Battery level (0-100)               |

> **Note:** The `user_id` column was removed in Phase 3D. EMQX inserts data using a service role (no auth.uid()), so ownership is verified through the `user_devices` table.

---

### Table: device_status

| Column       | Type        | Nullable | Default | Description                          |
|--------------|-------------|----------|---------|--------------------------------------|
| `id`         | BIGSERIAL (PK)| No     | Auto    | Internal row ID                      |
| `time`       | TIMESTAMPTZ | Yes      |  --        | Status report timestamp              |
| `device_id`  | TEXT        | Yes      |  --        | Device serial number                 |
| `fan_speed`  | INTEGER     | Yes      |  --        | Fan speed (0-100%)                   |
| `fan_mode`   | TEXT        | Yes      |  --        | `"automatic"` or `"custom"`          |
| `fan_state`  | TEXT        | Yes      |  --        | e.g., `"running"`, `"idle"`, `"off"` |
| `fw_ver`     | TEXT        | Yes      |  --        | Firmware version string              |
| `esp_temp`   | REAL        | Yes      |  --        | ESP32 internal temp (°C)             |
| `rssi`       | INTEGER     | Yes      |  --        | WiFi signal strength (dBm)           |
| `heap`       | INTEGER     | Yes      |  --        | Free heap memory (bytes)             |
| `uptime`     | INTEGER     | Yes      |  --        | Uptime (seconds)                     |
| `pm25_ema`   | REAL        | Yes      |  --        | PM2.5 exponential moving average     |
| `co2_ema`    | REAL        | Yes      |  --        | CO2 exponential moving average       |
| `client_id`  | TEXT        | Yes      |  --        | ESP32 MAC-based client ID            |

> **Note:** The `user_id` column was removed in Phase 3D for the same reason as `sensor_readings`.

---

### Table: outdoor_air_readings

| Column              | Type        | Nullable | Default | Description                          |
|---------------------|-------------|----------|---------|--------------------------------------|
| `id`                | BIGSERIAL (PK)| No     | Auto    | Internal row ID                      |
| `time`              | TIMESTAMPTZ | No       | NOW()   | When the reading was fetched         |
| `geo_zone`          | TEXT        | No       |  --        | Cache key, e.g. `"18.91_98.91"`     |
| `station_name`      | TEXT        | Yes      |  --        | Nearest AQICN station name           |
| `station_url`       | TEXT        | Yes      |  --        | URL to AQICN station page            |
| `aqi`               | INTEGER     | Yes      |  --        | AQI index (US EPA)                   |
| `pm25`              | REAL        | Yes      |  --        | PM2.5 concentration                  |
| `pm10`              | REAL        | Yes      |  --        | PM10 concentration                   |
| `o3`                | REAL        | Yes      |  --        | Ozone                                |
| `no2`               | REAL        | Yes      |  --        | NO2                                  |
| `so2`               | REAL        | Yes      |  --        | SO2                                  |
| `co`                | REAL        | Yes      |  --        | CO                                   |
| `temperature`       | REAL        | Yes      |  --        | Temperature from station (C)         |
| `humidity`          | REAL        | Yes      |  --        | Humidity from station (%)            |
| `wind`              | REAL        | Yes      |  --        | Wind speed (m/s)                     |
| `dominant_pollutant`| TEXT        | Yes      |  --        | Primary pollutant (e.g. `"pm25"`)    |
| `raw_json`          | JSONB       | Yes      |  --        | Raw AQICN API response for reference |

**Index:** `idx_outdoor_air_geo_zone_time` on `(geo_zone, time DESC)` for cache lookup.

> **Note:** This table acts as a geo zone cache for the AQICN API. The `get-outdoor-air` Edge Function inserts rows using the service role key. Data is retained for 180 days (pg_cron cleanup).

### Table: aqicn_stations ★ NEW

| Column              | Type        | Nullable | Default | Description                          |
|---------------------|-------------|----------|---------|--------------------------------------|
| `uid`               | TEXT (PK)   | No       |  --     | AQICN station unique ID              |
| `station_name`      | TEXT        | Yes      |  --     | Station display name                 |
| `latitude`          | REAL        | Yes      |  --     | GPS latitude                         |
| `longitude`         | REAL        | Yes      |  --     | GPS longitude                        |
| `aqi`               | INTEGER     | Yes      |  --     | AQI index (US EPA)                   |
| `dominant_pollutant`| TEXT        | Yes      |  --     | Primary pollutant                    |
| `measured_at`       | TIMESTAMPTZ | Yes      |  --     | When station last reported           |
| `fetched_at`        | TIMESTAMPTZ | No       | NOW()   | When our system fetched the data     |
| `raw_json`          | JSONB       | Yes      |  --     | Raw AQICN API response               |

**Indexes:**
- Primary key: `uid`
- `idx_aqicn_stations_aqi` on `(aqi)` — filter NULL AQI
- `idx_aqicn_stations_geo` on `(latitude, longitude)` — geo queries
- `idx_aqicn_stations_measured` on `(measured_at)` — freshness check

> **Note:** Data is UPSERT every hour by `fetch-aqicn-map` Edge Function (pg_cron job #7). No historical data retained — always shows latest snapshot. ~470 rows total.

### Table: thai_air_stations

| Column              | Type         | Nullable | Default | Description                          |
|---------------------|--------------|----------|---------|--------------------------------------|
| `id`                | BIGSERIAL (PK)| No     | Auto    | Internal station ID                  |
| `source`            | TEXT        | No       |  --        | `"air4thai"` or `"cusense"`         |
| `source_station_id` | TEXT        | No       |  --        | Original station ID from source API  |
| `name`              | TEXT        | Yes      |  --        | Station name (Thai)                  |
| `name_en`           | TEXT        | Yes      |  --        | Station name (English)               |
| `latitude`          | REAL        | Yes      |  --        | GPS latitude                         |
| `longitude`         | REAL        | Yes      |  --        | GPS longitude                        |
| `province`          | TEXT        | Yes      |  --        | Province name (Thai)                 |
| `amphoe`            | TEXT        | Yes      |  --        | District name                        |
| `tambol`            | TEXT        | Yes      |  --        | Sub-district name                    |
| `station_type`      | TEXT        | Yes      |  --        | Type (GROUND/BKK/MOBILE/cusensor3/nansensor) |
| `is_active`         | BOOLEAN     | No       | true    | Whether station is active            |
| `updated_at`        | TIMESTAMPTZ | No       | NOW()   | Last updated time                    |

**Unique Constraint:** `(source, source_station_id)` — UPSERT key for deduplication across sources.

### Table: thai_air_readings

| Column              | Type         | Nullable | Default | Description                          |
|---------------------|--------------|----------|---------|--------------------------------------|
| `id`                | BIGSERIAL (PK)| No     | Auto    | Internal row ID                      |
| `station_id`        | BIGINT (FK) | No       |  --        | References thai_air_stations(id)     |
| `pm25`              | REAL        | Yes      |  --        | PM2.5 (µg/m³)                       |
| `pm10`              | REAL        | Yes      |  --        | PM10 (µg/m³)                        |
| `pm1`               | REAL        | Yes      |  --        | PM1.0 (CUSense only)                |
| `aqi`               | INTEGER     | Yes      |  --        | AQI index (Air4Thai only)            |
| `temperature`       | REAL        | Yes      |  --        | Temperature (CUSense only)           |
| `humidity`          | REAL        | Yes      |  --        | Humidity (CUSense only)              |
| `co2`               | REAL        | Yes      |  --        | CO2 (CUSense only)                  |
| `o3`                | REAL        | Yes      |  --        | Ozone (Air4Thai only)                |
| `no2`               | REAL        | Yes      |  --        | NO2 (Air4Thai only)                  |
| `so2`               | REAL        | Yes      |  --        | SO2 (Air4Thai only)                  |
| `co`                | REAL        | Yes      |  --        | CO (Air4Thai only)                   |
| `measured_at`       | TIMESTAMPTZ | No       |  --        | When source API recorded the data    |
| `fetched_at`        | TIMESTAMPTZ | No       | NOW()   | When our system fetched the data     |

**Unique Index:** `idx_thai_air_readings_dedup` on `(station_id, measured_at)` — prevents duplicate readings.

> **Note:** Data is retained for 90 days (pg_cron cleanup at 03:30 UTC daily). The `fetch-thai-air` Edge Function inserts rows via batch UPSERT with `ignoreDuplicates: true`.

### View: v_thai_air_latest

A materialized-style view that JOINs `thai_air_stations` with the latest reading per station from `thai_air_readings`, and calculates `aqi_level` based on Thai PM2.5 standard. Frontend should query this view for the air quality map.

```sql
-- Simplified structure
SELECT s.*, r.pm25, r.pm10, r.aqi, r.measured_at, r.fetched_at,
  CASE
    WHEN r.pm25 <= 15.0 THEN 'excellent'
    WHEN r.pm25 <= 25.0 THEN 'good'
    WHEN r.pm25 <= 37.5 THEN 'moderate'
    WHEN r.pm25 <= 75.0 THEN 'unhealthy'
    ELSE 'hazardous'
  END AS aqi_level
FROM thai_air_stations s
LEFT JOIN LATERAL (
  SELECT * FROM thai_air_readings
  WHERE station_id = s.id
  ORDER BY measured_at DESC LIMIT 1
) r ON true
WHERE s.is_active = true;
```

> **Note:** `security_invoker = on` — view respects caller's RLS permissions.

---

## 13. RLS Policies Summary

Row Level Security (RLS) is enabled on all five tables. All policies use the `(select auth.uid())` optimization to prevent PostgreSQL from re-evaluating `auth.uid()` for every row.

### profiles (2 Policies)

| Policy                       | Operation | Rule                                                    |
|------------------------------|-----------|---------------------------------------------------------|
| Users can view own profile   | SELECT    | `id = (select auth.uid())`                              |
| Users can update own profile | UPDATE    | `id = (select auth.uid())`                              |

### user_devices (4 Policies)

| Policy                        | Operation | Rule                                                   |
|-------------------------------|-----------|--------------------------------------------------------|
| Users can view own devices    | SELECT    | `user_id = (select auth.uid())`                        |
| Users can insert own devices  | INSERT    | `user_id = (select auth.uid())`                        |
| Users can update own devices  | UPDATE    | `user_id = (select auth.uid())`                        |
| Users can delete own devices  | DELETE    | `user_id = (select auth.uid())`                        |

### sensor_readings (2 Policies)

| Policy                            | Operation | Rule                                                       |
|-----------------------------------|-----------|-------------------------------------------------------------|
| Service can insert readings       | INSERT    | `true` (allows service role / EMQX to insert any row)      |
| Users can view own device readings| SELECT    | `device_id IN (SELECT device_id FROM user_devices WHERE user_id = (select auth.uid()))` |

### device_status (2 Policies)

| Policy                           | Operation | Rule                                                        |
|----------------------------------|-----------|--------------------------------------------------------------|
| Service can insert status        | INSERT    | `true` (allows service role / EMQX to insert any row)       |
| Users can view own device status | SELECT    | `device_id IN (SELECT device_id FROM user_devices WHERE user_id = (select auth.uid()))` |

### outdoor_air_readings (2 Policies)

| Policy                                       | Operation | Rule                                                   |
|----------------------------------------------|-----------|--------------------------------------------------------|
| Authenticated users can view outdoor air data| SELECT    | `true` (all authenticated users can read)              |
| Service can insert outdoor air readings      | INSERT    | `true` (allows service role / Edge Function to insert) |

> **Note:** The INSERT policies with `true` are intentional  - EMQX Cloud uses the Supabase service role key to insert sensor/status data, and the `get-outdoor-air` Edge Function uses the service role key to insert outdoor air cache data. These rows have no user context. The `SECURITY DEFINER` functions handle ownership checks independently.

### aqicn_stations (2 Policies) ★ NEW

| Policy                              | Operation | Rule                                                   |
|-------------------------------------|-----------|--------------------------------------------------------|
| aqicn_stations_read_anon            | SELECT    | `true` (public data, readable by anon key)             |
| aqicn_stations_read_authenticated   | SELECT    | `true` (public data, readable by authenticated users)  |

> **Note:** AQICN station data is fully public. Write access is restricted to service_role (used by `fetch-aqicn-map` via pg_cron). No Security/Performance Advisor warnings.

### thai_air_stations (2 Policies)

| Policy                              | Operation | Rule                                                   |
|-------------------------------------|-----------|--------------------------------------------------------|
| thai_air_stations_read_all          | SELECT    | `true` (public data, all users can read)               |
| thai_air_stations_service_write     | ALL       | `true` (service_role only via Edge Function)            |

### thai_air_readings (2 Policies)

| Policy                              | Operation | Rule                                                   |
|-------------------------------------|-----------|--------------------------------------------------------|
| thai_air_readings_read_all          | SELECT    | `true` (public data, all users can read)               |
| thai_air_readings_service_write     | ALL       | `true` (service_role only via Edge Function)            |

> **Note:** Thai Air data is public — both stations and readings are readable by anyone with the anon key. Write access is restricted to service_role (used by the `fetch-thai-air` Edge Function via pg_cron).

---

## 14. Error Reference

### Common Error Patterns

| Error Code | Error Message                                    | Meaning                                         |
|------------|--------------------------------------------------|-------------------------------------------------|
| `PGRST301` | `JWT expired`                                    | Session token has expired; re-authenticate      |
| `42501`    | `new row violates row-level security policy`     | RLS policy prevented the operation              |
| `P0001`    | `Not authenticated`                              | Function requires login but no session found    |
| `P0001`    | `Device not found or access denied`              | Device doesn't exist or isn't owned by user     |
| `P0001`    | `Time range too large: maximum 7 days per query` | Requested range exceeds 7-day limit             |
| `23505`    | `duplicate key value violates unique constraint`  | Device already registered (unique_device)       |

### HTTP Status Codes

| Status | Meaning                         |
|--------|---------------------------------|
| 200    | Success                         |
| 400    | Bad request / Validation error  |
| 401    | Authentication required         |
| 403    | Forbidden (RLS or ownership)    |
| 405    | Method not allowed              |
| 500    | Internal server error           |
| 502    | Bad gateway (upstream API error)|

---

## 15. Rate Limits & Constraints

### Query Limits

| Constraint            | Value     | Applied To                                         |
|-----------------------|-----------|-----------------------------------------------------|
| Max time range        | 7 days    | `get_readings_range`, `get_device_status_range`     |
| Max rows per query    | 2,000     | `get_readings_range`, `get_device_status_range`     |
| Device per user       | Unlimited | `register_device`                                   |
| Users per device      | Max 3     | `register_device` (Phase 3E device sharing)         |
| Device uniqueness     | Per user  | Same user cannot register same device twice          |
| Max devices per batch | 100       | `get-outdoor-air-batch` (Phase 3F)                  |
| Max zones per batch   | 50        | `get-outdoor-air-batch` (Phase 3F)                  |

### Data Sampling (EMQX)

| Data Type      | ESP32 Publish Rate | EMQX Sampling | Stored Rate       |
|----------------|--------------------|---------------|--------------------|
| Sensor data    | Every 10 seconds   | Every 5 min   | ~288 rows/day/device |
| Device status  | Every 30 seconds   | Every 5 min   | ~288 rows/day/device |

### Data Retention

| Policy             | Duration  | Mechanism |
|--------------------|-----------|-----------|
| sensor_readings    | 180 days  | pg_cron   |
| device_status      | 180 days  | pg_cron   |
| outdoor_air_readings | 180 days | pg_cron  |
| thai_air_readings  | 90 days   | pg_cron (03:30 UTC daily) |

### AQICN API Limits

| Constraint          | Value                  | Notes                                        |
|---------------------|------------------------|----------------------------------------------|
| Free tier           | ~1,000 requests/day    | Shared across all AQICN API calls            |
| fetch-aqicn-map     | 24 req/day             | pg_cron every hour (:30) — **primary**       |
| get-outdoor-air     | ~50 req/day (declining)| ⚠ DEPRECATED — ทีมแอปกำลังเปลี่ยน           |
| get-outdoor-air-batch| ~0-10 req/day         | ⚠ DEPRECATED — ทีมแอปกำลังเปลี่ยน           |
| Total estimated     | ~74 req/day            | Well within 1,000 limit                      |
| Post-migration      | ~24 req/day            | After all clients switch to aqicn_stations   |

### Supabase Free Tier Limits

| Resource          | Limit                  |
|-------------------|------------------------|
| Database size     | 500 MB                 |
| API requests      | Unlimited (throttled)  |
| Realtime connections | 200 concurrent       |
| Edge Function invocations | 500,000/month  |

### Thai Air Quality Limits

| Constraint          | Value                  | Notes                                        |
|---------------------|------------------------|----------------------------------------------|
| Air4Thai            | 1 request/hour         | Returns all 186 stations in 1 call           |
| CUSense             | 1 request/hour         | Returns all ~59 non-PCD stations in 1 call   |
| Total API calls     | 2 per hour (pg_cron)   | Minimal load on source APIs                  |
| fetch-thai-air      | ~2 seconds per run     | Batch UPSERT + batch INSERT (Round 3A)       |
| Dedup               | (station_id, measured_at) | Prevents duplicate readings on re-run     |
| Storage estimate    | ~100 MB at 90 days     | 245 stations x 24 hrs x 90 days = ~529K rows|

---

*End of Documentation*

*This document is part of the BE-TPP Handoff Package for Phase 4 (Dashboard Development).*
*For project status and architecture details, see `BE-TPP_Project_Status_v10.md`.*
*For a live test console, visit: https://warunyususen.github.io/be-tpp-api-sample/*
*Change from v1.4: Deprecated get-outdoor-air + get-outdoor-air-batch (Section 10.1-10.2), added aqicn_stations REST API (Section 10.3), added Migration Guide, added aqicn_stations DB schema + RLS policies, updated AQICN API Limits*
*Change from v1.3: Added Section 11 (Thai Air Quality — v_thai_air_latest, thai_air_readings, fetch-thai-air), thai_air_stations/readings DB schema, RLS policies, retention 90 days, Thai Air rate limits*
*Change from v1.2: Added Section 10.2 (get-outdoor-air-batch), updated Section 7.2 (get_public_air_quality p_public_only parameter)*
*Change from v1.1: Added Section 10 (Outdoor Air Quality - AQICN Edge Function), outdoor_air_readings schema, RLS policies, AQICN rate limits*
