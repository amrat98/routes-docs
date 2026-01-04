---
layout: default
title: Background Mode Quick Start
description: Quick start guide for implementing background location tracking in your driver app
---

# 🚀 Quick Start: Background Location for Driver App

## Problem Solved
✅ Socket stays connected when driver opens Google Maps  
✅ Location updates continue in background  
✅ Users see real-time driver location even when driver navigates in Google Maps  

---

## Mobile App Changes (3 Steps)

### Step 1: Install Package

**React Native:**
```bash
npm install react-native-background-geolocation
```

**Flutter:**
```yaml
# pubspec.yaml
background_locator_2: ^2.0.5
```

---

### Step 2: Configure Permissions

**Android - AndroidManifest.xml:**
```xml
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
<uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
```

**iOS - Info.plist:**
```xml
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>We need your location to track trips in background</string>

<key>UIBackgroundModes</key>
<array>
    <string>location</string>
</array>
```

---

### Step 3: Implement Background Service

**React Native:**
```javascript
import BackgroundGeolocation from 'react-native-background-geolocation';
import io from 'socket.io-client';

// Initialize socket (same as before)
const socket = io('http://your-server.com:6065/location', {
  auth: { token: driverToken },
  transports: ['websocket'],
  reconnection: true,
  reconnectionAttempts: Infinity,
});

// Start background tracking when trip starts
BackgroundGeolocation.ready({
  desiredAccuracy: BackgroundGeolocation.DESIRED_ACCURACY_HIGH,
  distanceFilter: 10,
  stopOnTerminate: false,  // ⭐ Keep running after app closed
  startOnBoot: true,
  foregroundService: true, // ⭐ CRITICAL for Android
  notification: {
    title: 'Trip in Progress',
    text: 'Tracking your location',
  },
}).then(() => {
  BackgroundGeolocation.start();
});

// Listen to location updates
BackgroundGeolocation.onLocation((location) => {
  // Send to server (works in background!)
  socket.emit('update_location', {
    driverId: driverId,
    location: [location.coords.longitude, location.coords.latitude],
    speed: location.coords.speed || 0,
    accuracy: location.coords.accuracy,
    heading: location.coords.heading,
  });
});

// When driver clicks "Navigate"
const openGoogleMaps = (userLat, userLng) => {
  const url = `https://www.google.com/maps/dir/?api=1&destination=${userLat},${userLng}`;
  Linking.openURL(url);
  // Background service continues automatically! ✅
};

// Stop tracking when trip ends
const stopTracking = () => {
  BackgroundGeolocation.stop();
  socket.disconnect();
};
```

---

## New Socket Events (Server Already Supports!)

### 1. App Going to Background
```javascript
// Mobile: Notify server when app goes to background
socket.emit('app_background', {
  driverId: driverId,
  isBackground: true
});

// Server confirms
socket.on('background_mode_set', (data) => {
  console.log(data.message); // "Background mode active - location tracking continues"
});
```

### 2. Session Restoration (when app comes back)
```javascript
// Mobile: Restore session when app comes to foreground
socket.emit('restore_session', {
  driverId: driverId
});

// Server sends back active trip info
socket.on('session_restored', (data) => {
  console.log(data.activeTrip); // Current trip details
  console.log(data.message); // "Session restored successfully"
});
```

---

## Complete Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    DRIVER APP FLOW                          │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1. Trip Starts                                             │
│     ↓                                                       │
│  2. Start Background Geolocation                           │
│     ↓                                                       │
│  3. Socket connects and sends location every 5s            │
│     ↓                                                       │
│  4. Driver clicks "Navigate to Pickup"                     │
│     ↓                                                       │
│  5. App emits: app_background(isBackground: true)          │
│     ↓                                                       │
│  6. Google Maps opens (app in background)                  │
│     ↓                                                       │
│  7. Background service continues sending location          │
│     ↓ (Socket stays connected!)                            │
│  8. User sees driver moving in real-time ✅                 │
│     ↓                                                       │
│  9. Driver switches back to your app                       │
│     ↓                                                       │
│  10. App emits: restore_session(driverId)                  │
│     ↓                                                       │
│  11. Server sends back active trip details                 │
│     ↓                                                       │
│  12. App UI updates with current trip                      │
│     ↓                                                       │
│  13. Trip completes                                        │
│     ↓                                                       │
│  14. Stop background tracking                              │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Server Changes (Already Done! ✅)

Your server now supports:

1. **Background Mode Detection**
   - Server knows when app is in background
   - Logs: "Driver X app went to BACKGROUND"

2. **Session Restoration**
   - Re-registers socket mapping
   - Returns active trip details
   - Seamless reconnection

3. **Continuous Location Updates**
   - Works same in background and foreground
   - Uses intelligent batching under high load
   - Redis caching for fast queries

---

## Testing

### 1. Test Background Mode
```bash
# Start trip in app
# Click "Navigate to Pickup"
# Check server logs:
✅ "Driver 123 app went to BACKGROUND"
✅ "📍 Location updated for driver 123"
✅ "Broadcasting to users tracking driver 123"
```

### 2. Test Session Restoration
```bash
# Switch back to app from Google Maps
# Check server logs:
✅ "🔄 Restoring session: Driver 123 reconnected"
✅ "Session restored for driver 123"
```

### 3. Monitor Metrics
```bash
curl http://localhost:6065/api/system/metrics

# Check:
- activeConnections: Should stay constant
- updatesPerSecond: Should continue in background
- batchModeActive: true if >50 updates/sec
```

---

## Key Points

1. **Foreground Service Required (Android)**
   - Shows persistent notification
   - Prevents Android from killing the service
   - User knows tracking is active

2. **Always Location Permission (iOS)**
   - User must grant "Always" permission
   - Required for background tracking
   - Explained in permission dialog

3. **Socket Reconnection**
   - Automatic reconnection enabled
   - Infinite attempts (critical for background)
   - Uses WebSocket transport only

4. **Battery Optimization**
   - User must disable battery optimization for your app
   - Otherwise Android may kill background service
   - Show prompt in app to guide user

---

## Common Issues & Solutions

### Issue: Location stops after 5 minutes
**Solution:** 
- Enable foreground service (Android)
- Add location to UIBackgroundModes (iOS)
- Disable battery optimization

### Issue: Socket disconnects in background
**Solution:**
- Use WebSocket transport only
- Enable reconnection: true
- Set reconnectionAttempts: Infinity

### Issue: Google Maps opens but location stops
**Solution:**
- Verify background location permission granted
- Check foreground service is running
- Use BackgroundGeolocation library (not just Geolocation)

---

## Battery Impact

**Optimized Settings:**
```javascript
{
  distanceFilter: 20,        // Update every 20 meters (not every move)
  desiredAccuracy: MEDIUM,    // Don't need GPS accuracy for navigation
  stopTimeout: 5,             // Stop tracking if stationary for 5 min
  heartbeatInterval: 60,      // Reduce heartbeat frequency
}
```

**Battery Usage:** ~5-10% per hour (acceptable for trip duration)

---

## User Experience

### Before (Without Background Service)
```
Driver opens Google Maps
  ↓
Socket disconnects ❌
  ↓
Location updates stop ❌
  ↓
User sees driver frozen on map ❌
  ↓
Poor experience 😞
```

### After (With Background Service)
```
Driver opens Google Maps
  ↓
Background service continues ✅
  ↓
Socket stays connected ✅
  ↓
Location updates every 5s ✅
  ↓
User sees smooth real-time movement ✅
  ↓
Uber-like experience! 🎉
```

---

## Next Steps

1. ✅ Server is ready (no changes needed!)
2. 📱 Implement background service in mobile app
3. 🧪 Test with real devices
4. 📊 Monitor metrics endpoint
5. 🚀 Deploy to production

---

## Need Help?

Check these files:
- `BACKGROUND_LOCATION_SERVICE.md` - Complete implementation guide
- `HYBRID_ARCHITECTURE_GUIDE.md` - Server architecture
- `SOCKET_LOCATION_TRACKING_GUIDE.md` - Socket implementation details

**Your backend is production-ready! Just add the mobile background service and you're done.** 🎯
