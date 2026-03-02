# Step 7 - Intercom Device Authentication

## Goal

Add authentication for the intercom Android SBC device using a device provisioning code flow. An admin creates a device entry, the installer enters the provisioning code on the intercom during first-time setup, and the server issues a long-lived JWT for all future connections.

## Overview

- Admin API to create/manage device entries and generate provisioning codes
- Intercom shows a setup screen on first boot (enter provisioning code)
- Server validates the code and returns a JWT
- Intercom stores the JWT locally and uses it for WebSocket authentication
- Server validates JWT on every WebSocket connection

## Prerequisites

1. Step 2 completed (signaling server running)
2. Step 4 completed (Firebase Admin SDK configured on server)

---

## Step 7.1 - Add JWT Support to Signaling Server

1. Install dependencies:
   ```bash
   cd vidrom-signaling-server
   npm install jsonwebtoken uuid
   ```

2. Generate a secret key for signing JWTs:
   ```javascript
   // In server config or .env file
   const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key-change-in-production';
   ```

3. Create `auth.js` utility:
   ```javascript
   const jwt = require('jsonwebtoken');
   
   function generateDeviceToken(deviceId, buildingId) {
     return jwt.sign(
       {
         deviceId,
         buildingId,
         role: 'intercom',
       },
       JWT_SECRET,
       { expiresIn: '365d' }
     );
   }
   
   function verifyToken(token) {
     return jwt.verify(token, JWT_SECRET);
   }
   
   module.exports = { generateDeviceToken, verifyToken };
   ```

---

## Step 7.2 - Add Device Management Storage

1. Create a simple in-memory store (replace with database later):
   ```javascript
   // devices.js
   const { v4: uuidv4 } = require('uuid');
   
   const devices = new Map(); // deviceId -> device info
   const provisioningCodes = new Map(); // code -> deviceId
   
   function createDevice(buildingId, name) {
     const deviceId = uuidv4();
     const code = generateProvisioningCode();
     
     devices.set(deviceId, {
       deviceId,
       buildingId,
       name,
       status: 'pending', // pending | active | revoked
       createdAt: new Date().toISOString(),
     });
     
     provisioningCodes.set(code, deviceId);
     
     return { deviceId, code };
   }
   
   function generateProvisioningCode() {
     // 6-digit numeric code, easy to type on device
     return Math.floor(100000 + Math.random() * 900000).toString();
   }
   
   function validateProvisioningCode(code) {
     const deviceId = provisioningCodes.get(code);
     if (!deviceId) return null;
     
     const device = devices.get(deviceId);
     if (!device || device.status !== 'pending') return null;
     
     // Mark as active and remove the code
     device.status = 'active';
     provisioningCodes.delete(code);
     
     return device;
   }
   
   function getDevice(deviceId) {
     return devices.get(deviceId);
   }
   
   function revokeDevice(deviceId) {
     const device = devices.get(deviceId);
     if (device) device.status = 'revoked';
   }
   
   module.exports = {
     createDevice,
     validateProvisioningCode,
     getDevice,
     revokeDevice,
   };
   ```

---

## Step 7.3 - Add REST Endpoints to Signaling Server

1. Add REST endpoints to the existing HTTP server in `server.js` (port 8080, alongside WebSocket).
   No need for Express — we use the built-in `http` module that already powers the WebSocket server.

2. Import auth and device modules at the top of `server.js`:
   ```javascript
   const { generateDeviceToken, verifyToken } = require('./auth');
   const { createDevice, validateProvisioningCode, getDevice, revokeDevice } = require('./devices');
   ```

3. Add a `readBody()` helper and CORS headers to the HTTP handler, then add these routes:

   ```javascript
   // Admin: Create a new device (returns provisioning code)
   // POST /api/admin/devices  { buildingId, name }
   // → { deviceId, provisioningCode }

   // Device: Exchange provisioning code for JWT
   // POST /api/devices/provision  { code }
   // → { token, deviceId, buildingId }

   // Admin: Revoke a device
   // POST /api/admin/devices/:deviceId/revoke
   // → { success: true }
   ```

   All endpoints run on the same port 8080 as the WebSocket server.

---

## Step 7.4 - Authenticate WebSocket Connections

1. Update WebSocket `register` handler to require token for intercom role:
   ```javascript
   case 'register':
     if (message.role === 'intercom') {
       // Require JWT for intercom devices
       if (!message.token) {
         ws.send(JSON.stringify({ type: 'error', message: 'Token required' }));
         ws.close();
         return;
       }
       
       try {
         const decoded = verifyToken(message.token);
         const device = getDevice(decoded.deviceId);
         
         if (!device || device.status !== 'active') {
           ws.send(JSON.stringify({ type: 'error', message: 'Device revoked' }));
           ws.close();
           return;
         }
         
         client.role = 'intercom';
         client.deviceId = decoded.deviceId;
         client.buildingId = decoded.buildingId;
       } catch (err) {
         ws.send(JSON.stringify({ type: 'error', message: 'Invalid token' }));
         ws.close();
         return;
       }
     }
     // ... existing registration logic
     break;
   ```

---

## Step 7.5 - Add Setup Screen to Intercom App

1. Install AsyncStorage for persisting the JWT:
   ```bash
   cd vidrom-ai-intercom
   npx expo install @react-native-async-storage/async-storage
   ```

2. Create `DeviceSetupScreen.js`:
   ```javascript
   import React, { useState } from 'react';
   import {
     View,
     Text,
     TextInput,
     TouchableOpacity,
     StyleSheet,
     ActivityIndicator,
     Alert,
   } from 'react-native';
   import AsyncStorage from '@react-native-async-storage/async-storage';
   
   const SERVER_URL = 'http://10.0.0.16:8081';
   
   export default function DeviceSetupScreen({ onProvisioned }) {
     const [code, setCode] = useState('');
     const [loading, setLoading] = useState(false);
   
     const handleProvision = async () => {
       if (code.length !== 6) {
         Alert.alert('Error', 'Enter 6-digit provisioning code');
         return;
       }
   
       setLoading(true);
       try {
         const response = await fetch(`${SERVER_URL}/api/devices/provision`, {
           method: 'POST',
           headers: { 'Content-Type': 'application/json' },
           body: JSON.stringify({ code }),
         });
   
         const data = await response.json();
   
         if (!response.ok) {
           Alert.alert('Error', data.error || 'Provisioning failed');
           return;
         }
   
         // Store token and device info locally
         await AsyncStorage.setItem('deviceToken', data.token);
         await AsyncStorage.setItem('deviceId', data.deviceId);
         await AsyncStorage.setItem('buildingId', data.buildingId);
   
         onProvisioned(data.token);
       } catch (error) {
         Alert.alert('Error', 'Could not connect to server');
       } finally {
         setLoading(false);
       }
     };
   
     return (
       <View style={styles.container}>
         <Text style={styles.title}>Vidrom Intercom Setup</Text>
         <Text style={styles.subtitle}>Enter provisioning code</Text>
   
         <TextInput
           style={styles.input}
           value={code}
           onChangeText={setCode}
           keyboardType="numeric"
           maxLength={6}
           placeholder="000000"
           placeholderTextColor="#999"
         />
   
         <TouchableOpacity
           style={styles.button}
           onPress={handleProvision}
           disabled={loading}
         >
           {loading ? (
             <ActivityIndicator color="#fff" />
           ) : (
             <Text style={styles.buttonText}>Activate Device</Text>
           )}
         </TouchableOpacity>
       </View>
     );
   }
   
   const styles = StyleSheet.create({
     container: {
       flex: 1,
       justifyContent: 'center',
       alignItems: 'center',
       backgroundColor: '#1a1a2e',
       padding: 20,
     },
     title: {
       fontSize: 28,
       fontWeight: 'bold',
       color: '#fff',
       marginBottom: 10,
     },
     subtitle: {
       fontSize: 16,
       color: '#aaa',
       marginBottom: 30,
     },
     input: {
       backgroundColor: '#16213e',
       color: '#fff',
       fontSize: 32,
       letterSpacing: 10,
       textAlign: 'center',
       padding: 15,
       borderRadius: 10,
       width: 250,
       marginBottom: 20,
       borderWidth: 1,
       borderColor: '#0f3460',
     },
     button: {
       backgroundColor: '#e94560',
       paddingVertical: 15,
       paddingHorizontal: 40,
       borderRadius: 8,
       minWidth: 200,
       alignItems: 'center',
     },
     buttonText: {
       color: '#fff',
       fontSize: 16,
       fontWeight: '600',
     },
   });
   ```

---

## Step 7.6 - Update Intercom App.js for Auth Flow

1. Update `App.js` to check for saved token on startup:
   ```javascript
   import AsyncStorage from '@react-native-async-storage/async-storage';
   import DeviceSetupScreen from './DeviceSetupScreen';
   
   export default function App() {
     const [deviceToken, setDeviceToken] = useState(null);
     const [initializing, setInitializing] = useState(true);
   
     useEffect(() => {
       // Check if device is already provisioned
       const checkProvisioning = async () => {
         const token = await AsyncStorage.getItem('deviceToken');
         if (token) {
           setDeviceToken(token);
         }
         setInitializing(false);
       };
       checkProvisioning();
     }, []);
   
     if (initializing) return null;
   
     // Show setup screen if not provisioned
     if (!deviceToken) {
       return <DeviceSetupScreen onProvisioned={setDeviceToken} />;
     }
   
     // Existing app with token-based WebSocket auth
     // ... pass deviceToken to WebSocket connection
   }
   ```

2. Include token in WebSocket registration:
   ```javascript
   ws.onopen = () => {
     ws.send(JSON.stringify({
       type: 'register',
       role: 'intercom',
       token: deviceToken,
     }));
   };
   ```

---

## Step 7.7 - Testing Checklist

1. **Create Device:**
   - Call `POST /api/admin/devices` with buildingId and name
   - Verify provisioning code is returned (6 digits)

2. **Fresh Device Setup:**
   - Fresh install intercom app
   - Verify setup screen appears
   - Enter provisioning code → verify "Activate Device" succeeds
   - Verify app transitions to main call UI

3. **Persistent Auth:**
   - Restart intercom app
   - Verify it skips setup screen and connects automatically

4. **Invalid Code:**
   - Enter wrong code → verify error message
   - Use expired/already-used code → verify rejection

5. **Device Revocation:**
   - Call `POST /api/admin/devices/:id/revoke`
   - Restart intercom app → verify WebSocket connection is refused

---

## Step 7.8 - Project Structure After Completion

```
vidrom-ai/
├── vidrom-signaling-server/
│   ├── server.js           # Updated: REST endpoints on port 8080 + WS auth
│   ├── auth.js             # New: JWT generation/verification
│   ├── devices.js          # New: device management store
│   └── package.json        # Added: jsonwebtoken, uuid
├── vidrom-ai-intercom/
│   ├── App.js              # Updated: auth flow check
│   ├── DeviceSetupScreen.js # New: provisioning code entry UI
│   └── package.json        # Added: @react-native-async-storage
```

---

## Provisioning Flow Diagram

```
Admin                Server              Intercom
  |                    |                    |
  |-- POST /devices -->|                    |
  |<-- { code: 123456} |                    |
  |                    |                    |
  |    (give code to installer)             |
  |                    |                    |
  |                    |<-- POST /provision -|  (enter code)
  |                    |--- { token: JWT } ->|
  |                    |                    |
  |                    |                    |  (store JWT)
  |                    |                    |
  |                    |<== WebSocket =======|  (register + token)
  |                    |--- registered ----->|
```

---

## Security Notes

- Provisioning codes expire after first use (one-time)
- JWTs have 1-year expiry (configurable)
- Revoked devices are immediately blocked on WebSocket reconnect
- In production: use environment variable for JWT_SECRET, add admin auth to REST endpoints, replace in-memory store with database
- No Express dependency needed — REST endpoints run on the same `http` server as WebSocket (port 8080)
