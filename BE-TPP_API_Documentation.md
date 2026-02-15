# BE-TPP API Documentation

**Version:** 1.1
**Last Updated:** 15 February 2026
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
10. [Database Schema](#10-database-schema)
11. [RLS Policies Summary](#11-rls-policies-summary)
12. [Error Reference](#12-error-reference)
13. [Rate Limits & Constraints](#13-rate-limits--constraints)

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
                                              +-----+-----+
                                                    |
                                              Web Dashboard
```

### Tech Stack

| Component       | Technology                          |
|-----------------|-------------------------------------|
| Microcontroller | ESP32 (firmware v1.8.2)             |
| MQTT Broker     | EMQX Cloud (Serverless)             |
| Database        | Supabase (PostgreSQL)               |
| Authentication  | Supabase Auth (Magic Link)          |
| Edge Functions  | Supabase Edge Functions (Deno)      |
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
- **Data retention**: 180 days (automated via `pg_cron`)
- **Deduplication**: `time_bucket` column with UPSERT prevents duplicate entries

---

## 2. Authentication

BE-TPP uses **Supabase Auth with Magic Link** (passwordless email authentication).

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

Registers a new IoT device to the authenticated user's account. Each device can only be registered to one user at a time.

| Property        | Value                          |
|-----------------|--------------------------------|
| **Type**        | PostgreSQL RPC Function         |
| **Auth**        | Required (authenticated user)   |

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
  "message": "Device registered successfully"
}
```

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
  "error": "Device already registered to another user"
}
```

#### Error Cases

| Error                                           | Cause                                           |
|-------------------------------------------------|-------------------------------------------------|
| `Authentication required`                       | No valid session token                           |
| `Device already registered to your account`     | User already owns this device                    |
| `Device already registered to another user`     | Another user has registered this device          |

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

Retrieves air quality data for **all** devices that have GPS coordinates. This is a **public endpoint**  - no authentication required. Designed for the public-facing air quality map.

Provides different levels of detail based on device privacy settings:
- **All devices with coordinates**: Location, sensor readings, device name, data age
- **Public devices only (is_public = true)**: Additionally shows owner name, fan status, and online indicator

| Property        | Value                            |
|-----------------|----------------------------------|
| **Type**        | PostgreSQL RPC Function           |
| **Auth**        | **Not required** (public access)  |

#### Parameters

None.

#### Request Example

```javascript
// No auth required  - can call with anon key only
const { data, error } = await supabase.rpc('get_public_air_quality');
```

#### Response Schema

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

## 10. Database Schema

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
- `UNIQUE (device_id)`  --  one device per user globally

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

## 11. RLS Policies Summary

Row Level Security (RLS) is enabled on all four tables. All policies use the `(select auth.uid())` optimization to prevent PostgreSQL from re-evaluating `auth.uid()` for every row.

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

> **Note:** The INSERT policies with `true` are intentional  - EMQX Cloud uses the Supabase service role key to insert sensor/status data. These rows have no user context. The `SECURITY DEFINER` functions handle ownership checks independently.

---

## 12. Error Reference

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

---

## 13. Rate Limits & Constraints

### Query Limits

| Constraint            | Value     | Applied To                                         |
|-----------------------|-----------|-----------------------------------------------------|
| Max time range        | 7 days    | `get_readings_range`, `get_device_status_range`     |
| Max rows per query    | 2,000     | `get_readings_range`, `get_device_status_range`     |
| Device per user       | Unlimited | `register_device`                                   |
| Device uniqueness     | Global    | One device can only be registered to one user       |

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

### Supabase Free Tier Limits

| Resource          | Limit                  |
|-------------------|------------------------|
| Database size     | 500 MB                 |
| API requests      | Unlimited (throttled)  |
| Realtime connections | 200 concurrent       |
| Edge Function invocations | 500,000/month  |

---

*End of Documentation*

*This document is part of the BE-TPP Handoff Package for Phase 4 (Dashboard Development).*
*For project status and architecture details, see `BE-TPP_Project_Status_v6.md`.*
*For a live test console, visit: https://warunyususen.github.io/be-tpp-api-sample/*
