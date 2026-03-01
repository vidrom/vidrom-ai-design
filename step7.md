# Step 6 - Intercom Device Authentication

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

## Step 6.1 - Add JWT Support to Signaling Server

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

## Step 6.2 - Add Device Management Storage

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

## Step 6.3 - Add REST Endpoints to Signaling Server

1. Install Express (or add HTTP handling alongside WebSocket):
   ```bash
   npm install express cors
   ```

2. Add REST endpoints to `server.js`:
   ```javascript
   const express = require('express');
   const cors = require('cors');
   const app = express();
   app.use(cors());
   app.use(express.json());
   
   // Admin: Create a new device (returns provisioning code)
   app.post('/api/admin/devices', (req, res) => {
     const { buildingId, name } = req.body;
     if (!buildingId || !name) {
       return res.status(400).json({ error: 'buildingId and name required' });
     }
     const { deviceId, code } = createDevice(buildingId, name);
     res.json({ deviceId, provisioningCode: code });
   });
   
   // Device: Exchange provisioning code for JWT
   app.post('/api/devices/provision', (req, res) => {
     const { code } = req.body;
     if (!code) {
       return res.status(400).json({ error: 'Provisioning code required' });
     }
     
     const device = validateProvisioningCode(code);
     if (!device) {
       return res.status(401).json({ error: 'Invalid or expired code' });
     }
     
     const token = generateDeviceToken(device.deviceId, device.buildingId);
     res.json({
       token,
       deviceId: device.deviceId,
       buildingId: device.buildingId,
     });
   });
   
   // Admin: Revoke a device
   app.post('/api/admin/devices/:deviceId/revoke', (req, res) => {
     revokeDevice(req.params.deviceId);
     res.json({ success: true });
   });
   
   // Start HTTP server on port 8081 (WebSocket remains on 8080)
   app.listen(8081, () => {
     console.log('REST API running on port 8081');
   });
   ```

---

## Step 6.4 - Authenticate WebSocket Connections

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

## Step 6.5 - Add Setup Screen to Intercom App

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

## Step 6.6 - Update Intercom App.js for Auth Flow

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

## Step 6.7 - Testing Checklist

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

## Step 6.8 - Project Structure After Completion

```
vidrom-ai/
├── vidrom-signaling-server/
│   ├── server.js           # Updated: REST endpoints + WS auth
│   ├── auth.js             # New: JWT generation/verification
│   ├── devices.js          # New: device management store
│   └── package.json        # Added: jsonwebtoken, express, cors
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
