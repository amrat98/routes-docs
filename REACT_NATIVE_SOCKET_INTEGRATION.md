---
layout: default
title: React Native Socket Integration
description: Complete Socket.io integration guide for React Native driver and user apps
---

# 🚀 Complete Socket.io Integration Guide for React Native

## 📱 Driver App & User App Implementation

This guide provides **ready-to-use** code for integrating real-time location tracking in your React Native apps.

---

## 📦 Installation

### Step 1: Install Required Packages

```bash
# Socket.io client
npm install socket.io-client

# Background geolocation (for Driver app)
npm install react-native-background-geolocation

# Geolocation (for User app)
npm install @react-native-community/geolocation

# Maps
npm install react-native-maps

# Storage
npm install @react-native-async-storage/async-storage

# Permissions
npm install react-native-permissions
```

### Step 2: Link Native Dependencies

```bash
cd ios && pod install && cd ..
```

---

## 🚗 DRIVER APP IMPLEMENTATION

### File Structure
```
src/
├── services/
│   ├── SocketService.js          # Socket connection manager
│   └── BackgroundLocationService.js  # Background tracking
├── screens/
│   ├── TripScreen.js             # Active trip screen
│   └── DriverHomeScreen.js       # Driver home
└── utils/
    └── constants.js              # Configuration
```

---

### 1. Configuration (`src/utils/constants.js`)

```javascript
export const SOCKET_CONFIG = {
  SERVER_URL: 'http://your-server.com:6065', // Replace with your server
  NAMESPACE: '/location',
  RECONNECTION_ATTEMPTS: Infinity,
  RECONNECTION_DELAY: 1000,
  RECONNECTION_DELAY_MAX: 5000,
  TIMEOUT: 20000,
  LOCATION_UPDATE_INTERVAL: 5000, // 5 seconds
  HEARTBEAT_INTERVAL: 30000, // 30 seconds
};

export const GEOLOCATION_CONFIG = {
  desiredAccuracy: 'HIGH',
  distanceFilter: 10, // meters
  stopTimeout: 5, // minutes
  startOnBoot: true,
  stopOnTerminate: false,
  foregroundService: true,
};
```

---

### 2. Socket Service (`src/services/SocketService.js`)

```javascript
import io from 'socket.io-client';
import AsyncStorage from '@react-native-async-storage/async-storage';
import { SOCKET_CONFIG } from '../utils/constants';

class SocketService {
  constructor() {
    this.socket = null;
    this.isConnected = false;
    this.listeners = new Map();
    this.driverId = null;
    this.reconnectAttempts = 0;
  }

  /**
   * Initialize socket connection
   * @param {string} token - Driver JWT token
   * @param {string} driverId - Driver ID
   */
  async connect(token, driverId) {
    try {
      if (this.socket?.connected) {
        console.log('✅ Already connected');
        return;
      }

      this.driverId = driverId;

      // Store for background service
      await AsyncStorage.setItem('driver_token', token);
      await AsyncStorage.setItem('driver_id', driverId);

      this.socket = io(SOCKET_CONFIG.SERVER_URL + SOCKET_CONFIG.NAMESPACE, {
        auth: { token },
        transports: ['websocket'], // Important for background
        reconnection: true,
        reconnectionAttempts: SOCKET_CONFIG.RECONNECTION_ATTEMPTS,
        reconnectionDelay: SOCKET_CONFIG.RECONNECTION_DELAY,
        reconnectionDelayMax: SOCKET_CONFIG.RECONNECTION_DELAY_MAX,
        timeout: SOCKET_CONFIG.TIMEOUT,
      });

      this._setupEventListeners();
      
      console.log('🔌 Socket connecting...');
    } catch (error) {
      console.error('❌ Socket connection error:', error);
      throw error;
    }
  }

  /**
   * Setup default event listeners
   */
  _setupEventListeners() {
    // Connection events
    this.socket.on('connect', () => {
      console.log('🟢 Socket connected:', this.socket.id);
      this.isConnected = true;
      this.reconnectAttempts = 0;
      
      // Register driver
      this.registerDriver(this.driverId);
      
      // Trigger custom connect listeners
      this._triggerListeners('connect', { socketId: this.socket.id });
    });

    this.socket.on('disconnect', (reason) => {
      console.log('🔴 Socket disconnected:', reason);
      this.isConnected = false;
      this._triggerListeners('disconnect', { reason });
    });

    this.socket.on('connect_error', (error) => {
      console.log('❌ Connection error:', error.message);
      this.reconnectAttempts++;
      this._triggerListeners('connect_error', { error, attempts: this.reconnectAttempts });
    });

    this.socket.on('reconnect', (attemptNumber) => {
      console.log('🔄 Reconnected after', attemptNumber, 'attempts');
      this._triggerListeners('reconnect', { attemptNumber });
    });

    // Server acknowledgments
    this.socket.on('connected', (data) => {
      console.log('✅ Server confirmed connection:', data);
      this._triggerListeners('connected', data);
    });

    this.socket.on('driver_registered', (data) => {
      console.log('✅ Driver registered:', data);
      this._triggerListeners('driver_registered', data);
    });

    this.socket.on('location_updated', (data) => {
      console.log('📍 Location confirmed:', data);
      this._triggerListeners('location_updated', data);
    });

    this.socket.on('background_mode_set', (data) => {
      console.log('📱 Background mode:', data);
      this._triggerListeners('background_mode_set', data);
    });

    this.socket.on('session_restored', (data) => {
      console.log('🔄 Session restored:', data);
      this._triggerListeners('session_restored', data);
    });

    this.socket.on('trip_status_changed', (data) => {
      console.log('🚦 Trip status changed:', data);
      this._triggerListeners('trip_status_changed', data);
    });

    // Heartbeat
    this.socket.on('ping', (data) => {
      this.socket.emit('pong', { timestamp: new Date() });
    });

    // Error handling
    this.socket.on('error', (error) => {
      console.error('❌ Socket error:', error);
      this._triggerListeners('error', error);
    });
  }

  /**
   * Register driver with server
   */
  registerDriver(driverId) {
    if (!this.socket?.connected) {
      console.log('⚠️ Cannot register - not connected');
      return;
    }

    this.socket.emit('register_driver', { driverId });
    console.log('🚗 Registering driver:', driverId);
  }

  /**
   * Send location update to server
   */
  updateLocation(locationData) {
    if (!this.socket?.connected) {
      console.log('⚠️ Cannot send location - not connected');
      return false;
    }

    const payload = {
      driverId: this.driverId,
      location: locationData.location, // [longitude, latitude]
      speed: locationData.speed || 0,
      accuracy: locationData.accuracy || 10,
      heading: locationData.heading || 0,
    };

    this.socket.emit('update_location', payload);
    return true;
  }

  /**
   * Notify server when app goes to background
   */
  setBackgroundMode(isBackground) {
    if (!this.socket?.connected) return;

    this.socket.emit('app_background', {
      driverId: this.driverId,
      isBackground,
    });
    
    console.log('📱 Background mode set:', isBackground);
  }

  /**
   * Restore session when app comes back from background
   */
  restoreSession() {
    if (!this.socket?.connected) return;

    this.socket.emit('restore_session', {
      driverId: this.driverId,
    });
    
    console.log('🔄 Restoring session...');
  }

  /**
   * Check connection quality
   */
  checkConnection() {
    if (!this.socket?.connected) return;

    this.socket.emit('connection_check');
  }

  /**
   * Add event listener
   */
  on(event, callback) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, []);
    }
    this.listeners.get(event).push(callback);
  }

  /**
   * Remove event listener
   */
  off(event, callback) {
    if (!this.listeners.has(event)) return;
    
    const callbacks = this.listeners.get(event);
    const index = callbacks.indexOf(callback);
    if (index > -1) {
      callbacks.splice(index, 1);
    }
  }

  /**
   * Trigger custom listeners
   */
  _triggerListeners(event, data) {
    if (!this.listeners.has(event)) return;
    
    this.listeners.get(event).forEach(callback => {
      try {
        callback(data);
      } catch (error) {
        console.error('Error in listener:', error);
      }
    });
  }

  /**
   * Disconnect socket
   */
  disconnect() {
    if (this.socket) {
      this.socket.disconnect();
      this.socket = null;
      this.isConnected = false;
      console.log('🔌 Socket disconnected');
    }
  }

  /**
   * Get connection status
   */
  getStatus() {
    return {
      connected: this.isConnected,
      socketId: this.socket?.id,
      driverId: this.driverId,
    };
  }
}

export default new SocketService();
```

---

### 3. Background Location Service (`src/services/BackgroundLocationService.js`)

```javascript
import BackgroundGeolocation from 'react-native-background-geolocation';
import SocketService from './SocketService';
import { GEOLOCATION_CONFIG, SOCKET_CONFIG } from '../utils/constants';

class BackgroundLocationService {
  constructor() {
    this.isTracking = false;
    this.locationUpdateInterval = null;
  }

  /**
   * Initialize and start background tracking
   */
  async start(driverId, token) {
    if (this.isTracking) {
      console.log('⚠️ Already tracking');
      return;
    }

    try {
      // Connect socket first
      await SocketService.connect(token, driverId);

      // Configure background geolocation
      await BackgroundGeolocation.ready({
        // Geolocation Config
        desiredAccuracy: BackgroundGeolocation.DESIRED_ACCURACY_HIGH,
        distanceFilter: GEOLOCATION_CONFIG.distanceFilter,
        
        // Activity Recognition
        stopTimeout: GEOLOCATION_CONFIG.stopTimeout,
        
        // Application config
        debug: __DEV__, // Enable in development
        logLevel: BackgroundGeolocation.LOG_LEVEL_VERBOSE,
        stopOnTerminate: GEOLOCATION_CONFIG.stopOnTerminate,
        startOnBoot: GEOLOCATION_CONFIG.startOnBoot,
        
        // Android Foreground Service
        foregroundService: true,
        notification: {
          title: 'Trip in Progress',
          text: 'Tracking your location',
          color: '#4CAF50',
          smallIcon: 'ic_notification',
          largeIcon: 'ic_launcher',
        },
        
        // iOS Background
        preventSuspend: true,
        heartbeatInterval: 60,
        
        // Disable HTTP posting (we use socket)
        autoSync: false,
        maxRecordsToPersist: 0,
      });

      // Listen to location updates
      BackgroundGeolocation.onLocation(this._handleLocationUpdate.bind(this), (error) => {
        console.error('❌ Location error:', error);
      });

      // Listen to motion changes
      BackgroundGeolocation.onMotionChange((event) => {
        console.log('🚶 Motion changed:', event.isMoving ? 'Moving' : 'Stationary');
      });

      // Listen to provider changes (GPS on/off)
      BackgroundGeolocation.onProviderChange((provider) => {
        console.log('📡 Provider changed:', provider);
        if (!provider.enabled) {
          // GPS turned off - notify driver
          console.log('⚠️ GPS is disabled!');
        }
      });

      // Start tracking
      await BackgroundGeolocation.start();
      this.isTracking = true;
      
      console.log('✅ Background location tracking started');

    } catch (error) {
      console.error('❌ Failed to start tracking:', error);
      throw error;
    }
  }

  /**
   * Handle location updates
   */
  _handleLocationUpdate(location) {
    console.log('📍 Location update:', {
      lat: location.coords.latitude,
      lng: location.coords.longitude,
      speed: location.coords.speed,
      accuracy: location.coords.accuracy,
    });

    // Send to server via socket
    const success = SocketService.updateLocation({
      location: [location.coords.longitude, location.coords.latitude],
      speed: location.coords.speed || 0,
      accuracy: location.coords.accuracy || 10,
      heading: location.coords.heading || 0,
    });

    if (!success) {
      console.log('⚠️ Failed to send location - socket not connected');
    }
  }

  /**
   * Stop background tracking
   */
  async stop() {
    if (!this.isTracking) return;

    try {
      await BackgroundGeolocation.stop();
      SocketService.disconnect();
      this.isTracking = false;
      
      console.log('🛑 Background location tracking stopped');
    } catch (error) {
      console.error('❌ Error stopping tracking:', error);
    }
  }

  /**
   * Get current position once
   */
  async getCurrentPosition() {
    try {
      const location = await BackgroundGeolocation.getCurrentPosition({
        timeout: 30,
        maximumAge: 5000,
        enableHighAccuracy: true,
      });
      
      return {
        latitude: location.coords.latitude,
        longitude: location.coords.longitude,
        speed: location.coords.speed || 0,
        accuracy: location.coords.accuracy || 10,
        heading: location.coords.heading || 0,
      };
    } catch (error) {
      console.error('❌ Error getting current position:', error);
      throw error;
    }
  }

  /**
   * Check if tracking is active
   */
  isActive() {
    return this.isTracking;
  }
}

export default new BackgroundLocationService();
```

---

### 4. Driver Trip Screen (`src/screens/TripScreen.js`)

```javascript
import React, { useEffect, useState } from 'react';
import {
  View,
  Text,
  Button,
  StyleSheet,
  Alert,
  Linking,
  AppState,
  Platform,
} from 'react-native';
import BackgroundLocationService from '../services/BackgroundLocationService';
import SocketService from '../services/SocketService';

const TripScreen = ({ route, navigation }) => {
  const { trip, driverToken, driverId } = route.params;
  
  const [isTracking, setIsTracking] = useState(false);
  const [socketStatus, setSocketStatus] = useState('Disconnected');
  const [locationCount, setLocationCount] = useState(0);
  const [appState, setAppState] = useState(AppState.currentState);

  useEffect(() => {
    startTracking();

    // Listen to app state changes (background/foreground)
    const subscription = AppState.addEventListener('change', handleAppStateChange);

    // Socket event listeners
    SocketService.on('connect', () => setSocketStatus('Connected'));
    SocketService.on('disconnect', () => setSocketStatus('Disconnected'));
    SocketService.on('location_updated', handleLocationConfirmed);
    SocketService.on('session_restored', handleSessionRestored);

    return () => {
      subscription.remove();
      stopTracking();
    };
  }, []);

  const startTracking = async () => {
    try {
      await BackgroundLocationService.start(driverId, driverToken);
      setIsTracking(true);
      Alert.alert('Success', 'Location tracking started');
    } catch (error) {
      console.error('Failed to start tracking:', error);
      Alert.alert('Error', 'Failed to start location tracking');
    }
  };

  const stopTracking = async () => {
    await BackgroundLocationService.stop();
    setIsTracking(false);
  };

  const handleAppStateChange = (nextAppState) => {
    if (appState.match(/inactive|background/) && nextAppState === 'active') {
      // App came to foreground
      console.log('📱 App came to FOREGROUND');
      SocketService.setBackgroundMode(false);
      SocketService.restoreSession();
    } else if (nextAppState.match(/inactive|background/)) {
      // App went to background
      console.log('📱 App went to BACKGROUND');
      SocketService.setBackgroundMode(true);
    }
    setAppState(nextAppState);
  };

  const handleLocationConfirmed = (data) => {
    setLocationCount(prev => prev + 1);
    console.log('✅ Location confirmed by server:', data);
  };

  const handleSessionRestored = (data) => {
    console.log('🔄 Session restored:', data);
    Alert.alert('Session Restored', 'Your trip session has been restored');
  };

  const openGoogleMaps = async () => {
    const { latitude, longitude } = trip.pickupLocation;
    const url = Platform.select({
      ios: `maps://app?daddr=${latitude},${longitude}`,
      android: `google.navigation:q=${latitude},${longitude}`,
    });

    const supported = await Linking.canOpenURL(url);
    
    if (supported) {
      await Linking.openURL(url);
      // Background tracking continues automatically!
    } else {
      // Fallback to browser
      const webUrl = `https://www.google.com/maps/dir/?api=1&destination=${latitude},${longitude}`;
      Linking.openURL(webUrl);
    }
  };

  const completeTrip = async () => {
    Alert.alert(
      'Complete Trip',
      'Are you sure you want to complete this trip?',
      [
        { text: 'Cancel', style: 'cancel' },
        {
          text: 'Complete',
          onPress: async () => {
            await stopTracking();
            navigation.goBack();
          },
        },
      ]
    );
  };

  return (
    <View style={styles.container}>
      <Text style={styles.title}>Trip in Progress</Text>
      
      <View style={styles.statusContainer}>
        <Text style={styles.statusLabel}>Socket Status:</Text>
        <Text style={[
          styles.statusValue,
          { color: socketStatus === 'Connected' ? '#4CAF50' : '#f44336' }
        ]}>
          {socketStatus}
        </Text>
      </View>

      <View style={styles.statusContainer}>
        <Text style={styles.statusLabel}>Tracking:</Text>
        <Text style={[
          styles.statusValue,
          { color: isTracking ? '#4CAF50' : '#f44336' }
        ]}>
          {isTracking ? 'Active' : 'Stopped'}
        </Text>
      </View>

      <View style={styles.statusContainer}>
        <Text style={styles.statusLabel}>Updates Sent:</Text>
        <Text style={styles.statusValue}>{locationCount}</Text>
      </View>

      <View style={styles.tripInfo}>
        <Text style={styles.label}>Pickup:</Text>
        <Text style={styles.value}>{trip.pickupAddress}</Text>
        
        <Text style={styles.label}>Destination:</Text>
        <Text style={styles.value}>{trip.destinationAddress}</Text>
      </View>

      <View style={styles.buttonContainer}>
        <Button
          title="Navigate to Pickup"
          onPress={openGoogleMaps}
          color="#4CAF50"
        />
        
        <Button
          title={isTracking ? 'Stop Tracking' : 'Start Tracking'}
          onPress={isTracking ? stopTracking : startTracking}
          color={isTracking ? '#f44336' : '#2196F3'}
        />
        
        <Button
          title="Complete Trip"
          onPress={completeTrip}
          color="#FF9800"
        />
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 20,
    backgroundColor: '#fff',
  },
  title: {
    fontSize: 24,
    fontWeight: 'bold',
    marginBottom: 20,
    textAlign: 'center',
  },
  statusContainer: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    marginBottom: 10,
    paddingVertical: 10,
    borderBottomWidth: 1,
    borderBottomColor: '#eee',
  },
  statusLabel: {
    fontSize: 16,
    fontWeight: '600',
  },
  statusValue: {
    fontSize: 16,
    fontWeight: 'bold',
  },
  tripInfo: {
    marginVertical: 20,
    padding: 15,
    backgroundColor: '#f5f5f5',
    borderRadius: 8,
  },
  label: {
    fontSize: 14,
    fontWeight: '600',
    marginTop: 10,
    color: '#666',
  },
  value: {
    fontSize: 16,
    marginTop: 5,
  },
  buttonContainer: {
    marginTop: 20,
    gap: 10,
  },
});

export default TripScreen;
```

---

## 👤 USER APP IMPLEMENTATION

### File Structure
```
src/
├── services/
│   └── UserSocketService.js    # Socket for tracking driver
├── screens/
│   ├── TrackTripScreen.js      # Track driver location
│   └── TripDetailsScreen.js    # Trip info
└── components/
    └── DriverMap.js            # Map with driver marker
```

---

### 1. User Socket Service (`src/services/UserSocketService.js`)

```javascript
import io from 'socket.io-client';
import { SOCKET_CONFIG } from '../utils/constants';

class UserSocketService {
  constructor() {
    this.socket = null;
    this.isConnected = false;
    this.listeners = new Map();
    this.currentTripId = null;
  }

  /**
   * Connect and start tracking a trip
   */
  async connectAndTrack(token, userId, tripId) {
    try {
      if (this.socket?.connected && this.currentTripId === tripId) {
        console.log('✅ Already tracking this trip');
        return;
      }

      // Disconnect existing connection
      if (this.socket) {
        this.disconnect();
      }

      this.currentTripId = tripId;

      this.socket = io(SOCKET_CONFIG.SERVER_URL + SOCKET_CONFIG.NAMESPACE, {
        auth: { token },
        transports: ['websocket'],
        reconnection: true,
        reconnectionAttempts: SOCKET_CONFIG.RECONNECTION_ATTEMPTS,
        reconnectionDelay: SOCKET_CONFIG.RECONNECTION_DELAY,
        reconnectionDelayMax: SOCKET_CONFIG.RECONNECTION_DELAY_MAX,
        timeout: SOCKET_CONFIG.TIMEOUT,
      });

      this._setupEventListeners();
      
      // Wait for connection
      await new Promise((resolve, reject) => {
        const timeout = setTimeout(() => {
          reject(new Error('Connection timeout'));
        }, 10000);

        this.socket.once('connect', () => {
          clearTimeout(timeout);
          
          // Start tracking the trip
          this.socket.emit('track_trip', { tripId, userId });
          resolve();
        });

        this.socket.once('connect_error', (error) => {
          clearTimeout(timeout);
          reject(error);
        });
      });

      console.log('🟢 Connected and tracking trip:', tripId);
    } catch (error) {
      console.error('❌ Failed to connect:', error);
      throw error;
    }
  }

  /**
   * Setup event listeners
   */
  _setupEventListeners() {
    this.socket.on('connect', () => {
      console.log('🟢 User socket connected:', this.socket.id);
      this.isConnected = true;
      this._triggerListeners('connect', { socketId: this.socket.id });
    });

    this.socket.on('disconnect', (reason) => {
      console.log('🔴 User socket disconnected:', reason);
      this.isConnected = false;
      this._triggerListeners('disconnect', { reason });
    });

    this.socket.on('reconnect', (attemptNumber) => {
      console.log('🔄 Reconnected after', attemptNumber, 'attempts');
      
      // Re-track trip after reconnection
      if (this.currentTripId) {
        this.socket.emit('track_trip', { tripId: this.currentTripId });
      }
    });

    // Tracking started confirmation
    this.socket.on('tracking_started', (data) => {
      console.log('✅ Tracking started:', data);
      this._triggerListeners('tracking_started', data);
    });

    // Driver location updates (MAIN EVENT)
    this.socket.on('location_update', (data) => {
      console.log('📍 Driver location update:', {
        lat: data.location[1],
        lng: data.location[0],
        speed: data.speed,
        heading: data.heading,
      });
      this._triggerListeners('location_update', data);
    });

    // Trip status changes
    this.socket.on('trip_status_changed', (data) => {
      console.log('🚦 Trip status changed:', data);
      this._triggerListeners('trip_status_changed', data);
    });

    // Heartbeat
    this.socket.on('ping', () => {
      this.socket.emit('pong', { timestamp: new Date() });
    });

    // Errors
    this.socket.on('error', (error) => {
      console.error('❌ Socket error:', error);
      this._triggerListeners('error', error);
    });
  }

  /**
   * Stop tracking trip
   */
  stopTracking() {
    if (this.socket?.connected) {
      this.socket.emit('untrack_trip');
      this.currentTripId = null;
      console.log('🛑 Stopped tracking');
    }
  }

  /**
   * Add event listener
   */
  on(event, callback) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, []);
    }
    this.listeners.get(event).push(callback);
  }

  /**
   * Remove event listener
   */
  off(event, callback) {
    if (!this.listeners.has(event)) return;
    
    const callbacks = this.listeners.get(event);
    const index = callbacks.indexOf(callback);
    if (index > -1) {
      callbacks.splice(index, 1);
    }
  }

  /**
   * Trigger listeners
   */
  _triggerListeners(event, data) {
    if (!this.listeners.has(event)) return;
    
    this.listeners.get(event).forEach(callback => {
      try {
        callback(data);
      } catch (error) {
        console.error('Error in listener:', error);
      }
    });
  }

  /**
   * Disconnect
   */
  disconnect() {
    if (this.socket) {
      this.stopTracking();
      this.socket.disconnect();
      this.socket = null;
      this.isConnected = false;
      this.currentTripId = null;
      console.log('🔌 User socket disconnected');
    }
  }

  /**
   * Get status
   */
  getStatus() {
    return {
      connected: this.isConnected,
      socketId: this.socket?.id,
      trackingTripId: this.currentTripId,
    };
  }
}

export default new UserSocketService();
```

---

### 2. Track Trip Screen (`src/screens/TrackTripScreen.js`)

```javascript
import React, { useEffect, useState, useRef } from 'react';
import {
  View,
  Text,
  StyleSheet,
  ActivityIndicator,
  Alert,
} from 'react-native';
import MapView, { Marker, Polyline, PROVIDER_GOOGLE } from 'react-native-maps';
import UserSocketService from '../services/UserSocketService';

const TrackTripScreen = ({ route, navigation }) => {
  const { trip, userToken, userId } = route.params;
  
  const mapRef = useRef(null);
  const [loading, setLoading] = useState(true);
  const [connected, setConnected] = useState(false);
  const [driverLocation, setDriverLocation] = useState(null);
  const [tripData, setTripData] = useState(trip);
  const [locationHistory, setLocationHistory] = useState([]);

  useEffect(() => {
    startTracking();

    // Socket event listeners
    UserSocketService.on('connect', handleConnect);
    UserSocketService.on('disconnect', handleDisconnect);
    UserSocketService.on('location_update', handleLocationUpdate);
    UserSocketService.on('trip_status_changed', handleTripStatusChanged);
    UserSocketService.on('tracking_started', handleTrackingStarted);

    return () => {
      UserSocketService.disconnect();
    };
  }, []);

  const startTracking = async () => {
    try {
      await UserSocketService.connectAndTrack(userToken, userId, trip._id);
    } catch (error) {
      console.error('Failed to start tracking:', error);
      Alert.alert('Error', 'Failed to connect to tracking service');
      setLoading(false);
    }
  };

  const handleConnect = () => {
    setConnected(true);
    console.log('✅ Connected to tracking service');
  };

  const handleDisconnect = ({ reason }) => {
    setConnected(false);
    console.log('🔴 Disconnected:', reason);
  };

  const handleTrackingStarted = (data) => {
    setLoading(false);
    console.log('✅ Tracking started:', data);
  };

  const handleLocationUpdate = (data) => {
    console.log('📍 Received location update:', data);

    const newLocation = {
      latitude: data.location[1],
      longitude: data.location[0],
      speed: data.speed || 0,
      heading: data.heading || 0,
      accuracy: data.accuracy || 10,
    };

    setDriverLocation(newLocation);

    // Add to history for polyline
    setLocationHistory(prev => [...prev.slice(-50), newLocation]); // Keep last 50 points

    // Update trip data if provided
    if (data.status) {
      setTripData(prev => ({
        ...prev,
        status: data.status,
        is_cancelled_by_driver: data.is_cancelled_by_driver,
        cancellation_reason: data.cancellation_reason,
      }));
    }

    // Animate map to driver location
    if (mapRef.current && newLocation) {
      mapRef.current.animateCamera({
        center: newLocation,
        zoom: 16,
        heading: data.heading || 0,
      }, { duration: 1000 });
    }
  };

  const handleTripStatusChanged = (data) => {
    console.log('🚦 Trip status changed:', data);
    
    setTripData(prev => ({
      ...prev,
      ...data.trip,
    }));

    if (data.status === 'Cancelled') {
      Alert.alert(
        'Trip Cancelled',
        data.metadata?.cancellation_reason || 'Trip has been cancelled',
        [{ text: 'OK', onPress: () => navigation.goBack() }]
      );
    } else if (data.status === 'Completed') {
      Alert.alert(
        'Trip Completed',
        'Your trip has been completed',
        [{ text: 'OK', onPress: () => navigation.goBack() }]
      );
    }
  };

  if (loading) {
    return (
      <View style={styles.loadingContainer}>
        <ActivityIndicator size="large" color="#4CAF50" />
        <Text style={styles.loadingText}>Connecting to driver...</Text>
      </View>
    );
  }

  const pickupLocation = {
    latitude: tripData.startingPoint?.coordinates?[1] || 0,
    longitude: tripData.startingPoint?.coordinates?.[0] || 0,
  };

  const destinationLocation = {
    latitude: tripData.destinationPoint?.coordinates?.[1] || 0,
    longitude: tripData.destinationPoint?.coordinates?.[0] || 0,
  };

  return (
    <View style={styles.container}>
      {/* Map */}
      <MapView
        ref={mapRef}
        provider={PROVIDER_GOOGLE}
        style={styles.map}
        initialRegion={{
          ...pickupLocation,
          latitudeDelta: 0.01,
          longitudeDelta: 0.01,
        }}
        showsUserLocation
        showsMyLocationButton
        showsCompass
      >
        {/* Pickup Marker */}
        <Marker
          coordinate={pickupLocation}
          title="Pickup Location"
          description={tripData.startingPointAddress}
          pinColor="green"
        />

        {/* Destination Marker */}
        <Marker
          coordinate={destinationLocation}
          title="Destination"
          description={tripData.destinationPointAddress}
          pinColor="red"
        />

        {/* Driver Marker */}
        {driverLocation && (
          <Marker
            coordinate={driverLocation}
            title="Driver"
            description={`Speed: ${Math.round(driverLocation.speed * 3.6)} km/h`}
            rotation={driverLocation.heading}
            anchor={{ x: 0.5, y: 0.5 }}
          >
            <View style={styles.driverMarker}>
              <Text style={styles.carIcon}>🚗</Text>
            </View>
          </Marker>
        )}

        {/* Path traveled by driver */}
        {locationHistory.length > 1 && (
          <Polyline
            coordinates={locationHistory}
            strokeColor="#4CAF50"
            strokeWidth={3}
            lineDashPattern={[1]}
          />
        )}
      </MapView>

      {/* Status Bar */}
      <View style={styles.statusBar}>
        <View style={styles.statusItem}>
          <Text style={styles.statusLabel}>Connection:</Text>
          <View style={[
            styles.statusIndicator,
            { backgroundColor: connected ? '#4CAF50' : '#f44336' }
          ]} />
        </View>

        <View style={styles.statusItem}>
          <Text style={styles.statusLabel}>Trip Status:</Text>
          <Text style={styles.statusValue}>{tripData.status}</Text>
        </View>

        {driverLocation && (
          <View style={styles.statusItem}>
            <Text style={styles.statusLabel}>Speed:</Text>
            <Text style={styles.statusValue}>
              {Math.round(driverLocation.speed * 3.6)} km/h
            </Text>
          </View>
        )}
      </View>

      {/* Trip Info */}
      <View style={styles.tripInfo}>
        <Text style={styles.tripInfoTitle}>Trip Details</Text>
        
        <View style={styles.infoRow}>
          <Text style={styles.infoLabel}>From:</Text>
          <Text style={styles.infoValue} numberOfLines={1}>
            {tripData.startingPointAddress}
          </Text>
        </View>

        <View style={styles.infoRow}>
          <Text style={styles.infoLabel}>To:</Text>
          <Text style={styles.infoValue} numberOfLines={1}>
            {tripData.destinationPointAddress}
          </Text>
        </View>

        <View style={styles.infoRow}>
          <Text style={styles.infoLabel}>Distance:</Text>
          <Text style={styles.infoValue}>{tripData.distance} km</Text>
        </View>

        <View style={styles.infoRow}>
          <Text style={styles.infoLabel}>Fare:</Text>
          <Text style={styles.infoValue}>₹{tripData.fare?.total || 0}</Text>
        </View>
      </View>
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
  },
  loadingContainer: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    backgroundColor: '#fff',
  },
  loadingText: {
    marginTop: 10,
    fontSize: 16,
    color: '#666',
  },
  map: {
    flex: 1,
  },
  driverMarker: {
    width: 40,
    height: 40,
    justifyContent: 'center',
    alignItems: 'center',
  },
  carIcon: {
    fontSize: 30,
  },
  statusBar: {
    position: 'absolute',
    top: 50,
    left: 10,
    right: 10,
    backgroundColor: 'white',
    borderRadius: 8,
    padding: 10,
    flexDirection: 'row',
    justifyContent: 'space-around',
    shadowColor: '#000',
    shadowOffset: { width: 0, height: 2 },
    shadowOpacity: 0.25,
    shadowRadius: 3.84,
    elevation: 5,
  },
  statusItem: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  statusLabel: {
    fontSize: 12,
    color: '#666',
    marginRight: 5,
  },
  statusValue: {
    fontSize: 12,
    fontWeight: 'bold',
  },
  statusIndicator: {
    width: 10,
    height: 10,
    borderRadius: 5,
  },
  tripInfo: {
    position: 'absolute',
    bottom: 0,
    left: 0,
    right: 0,
    backgroundColor: 'white',
    borderTopLeftRadius: 20,
    borderTopRightRadius: 20,
    padding: 20,
    shadowColor: '#000',
    shadowOffset: { width: 0, height: -2 },
    shadowOpacity: 0.25,
    shadowRadius: 3.84,
    elevation: 5,
  },
  tripInfoTitle: {
    fontSize: 18,
    fontWeight: 'bold',
    marginBottom: 15,
  },
  infoRow: {
    flexDirection: 'row',
    justifyContent: 'space-between',
    marginBottom: 10,
  },
  infoLabel: {
    fontSize: 14,
    color: '#666',
    flex: 1,
  },
  infoValue: {
    fontSize: 14,
    fontWeight: '600',
    flex: 2,
    textAlign: 'right',
  },
});

export default TrackTripScreen;
```

---

## 🔧 Platform-Specific Configuration

### Android

#### 1. AndroidManifest.xml
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    
    <!-- Permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.WAKE_LOCK" />
    
    <application>
        <!-- Google Maps API Key -->
        <meta-data
            android:name="com.google.android.geo.API_KEY"
            android:value="YOUR_GOOGLE_MAPS_API_KEY"/>
    </application>
</manifest>
```

#### 2. Request Permissions (Driver App)
```javascript
import { PermissionsAndroid, Platform } from 'react-native';

export const requestLocationPermissions = async () => {
  if (Platform.OS === 'android') {
    try {
      const granted = await PermissionsAndroid.requestMultiple([
        PermissionsAndroid.PERMISSIONS.ACCESS_FINE_LOCATION,
        PermissionsAndroid.PERMISSIONS.ACCESS_COARSE_LOCATION,
        PermissionsAndroid.PERMISSIONS.ACCESS_BACKGROUND_LOCATION,
      ]);

      if (granted['android.permission.ACCESS_FINE_LOCATION'] === 'granted' &&
          granted['android.permission.ACCESS_BACKGROUND_LOCATION'] === 'granted') {
        console.log('✅ All permissions granted');
        return true;
      } else {
        console.log('❌ Permissions denied');
        return false;
      }
    } catch (err) {
      console.error('Permission error:', err);
      return false;
    }
  }
  return true;
};
```

---

### iOS

#### Info.plist
```xml
<key>NSLocationAlwaysAndWhenInUseUsageDescription</key>
<string>We need your location to track trips even when you use Google Maps</string>

<key>NSLocationWhenInUseUsageDescription</key>
<string>We need your location to show your position during the trip</string>

<key>NSLocationAlwaysUsageDescription</key>
<string>We need your location to track trips in the background</string>

<key>UIBackgroundModes</key>
<array>
    <string>location</string>
    <string>fetch</string>
</array>

<key>NSAppTransportSecurity</key>
<dict>
    <key>NSAllowsArbitraryLoads</key>
    <true/>
</dict>
```

---

## 📊 Testing

### Test Driver App
```javascript
// 1. Start trip
await BackgroundLocationService.start(driverId, token);

// 2. Check console logs
// ✅ Should see: "Background location tracking started"
// ✅ Should see: "Socket connected"
// ✅ Should see: "Driver registered"
// ✅ Should see: "Location update" every 5 seconds

// 3. Open Google Maps
openGoogleMaps();

// 4. Check console logs
// ✅ Should see: "App went to BACKGROUND"
// ✅ Should see: "Location update" continues
// ✅ Socket should stay connected

// 5. Return to app
// ✅ Should see: "App came to FOREGROUND"
// ✅ Should see: "Session restored"
```

### Test User App
```javascript
// 1. Start tracking
await UserSocketService.connectAndTrack(token, userId, tripId);

// 2. Check console logs
// ✅ Should see: "Connected and tracking trip"
// ✅ Should see: "Tracking started"
// ✅ Should see: "Driver location update" every 5 seconds

// 3. Check map
// ✅ Driver marker should update smoothly
// ✅ Speed should change
// ✅ Heading should rotate marker
```

---

## 🎯 Server Events Reference

### Driver Events

| Event | Direction | Payload | Response |
|-------|-----------|---------|----------|
| `register_driver` | Driver → Server | `{ driverId }` | `driver_registered` |
| `update_location` | Driver → Server | `{ driverId, location, speed, heading, accuracy }` | `location_updated` |
| `app_background` | Driver → Server | `{ driverId, isBackground }` | `background_mode_set` |
| `restore_session` | Driver → Server | `{ driverId }` | `session_restored` |
| `location_updated` | Server → Driver | `{ success, cached, queued, processingTime }` | - |

### User Events

| Event | Direction | Payload | Response |
|-------|-----------|---------|----------|
| `track_trip` | User → Server | `{ tripId, userId }` | `tracking_started` |
| `untrack_trip` | User → Server | `{}` | `tracking_stopped` |
| `location_update` | Server → User | `{ driverId, location, speed, heading, tripData }` | - |
| `trip_status_changed` | Server → User | `{ tripId, status, metadata }` | - |

### Common Events

| Event | Direction | Description |
|-------|-----------|-------------|
| `connect` | Server → Client | Socket connected |
| `disconnect` | Server → Client | Socket disconnected |
| `reconnect` | Server → Client | Socket reconnected |
| `ping` | Server → Client | Heartbeat check |
| `pong` | Client → Server | Heartbeat response |
| `error` | Server → Client | Error occurred |

---

## 🚨 Common Issues & Solutions

### Issue: Socket disconnects in background

**Solution:**
```javascript
// Use WebSocket transport only
{
  transports: ['websocket'], // NOT ['polling', 'websocket']
  reconnection: true,
  reconnectionAttempts: Infinity,
}
```

### Issue: Location stops after a few minutes

**Solution:**
```javascript
// Enable foreground service (Android)
{
  foregroundService: true,
  notification: {
    title: 'Trip in Progress',
    text: 'Tracking your location',
  },
}

// iOS - add location to UIBackgroundModes in Info.plist
```

### Issue: Permission denied

**Solution:**
```javascript
// Request background location permission explicitly
const granted = await PermissionsAndroid.request(
  PermissionsAndroid.PERMISSIONS.ACCESS_BACKGROUND_LOCATION
);
```

### Issue: Map doesn't update smoothly

**Solution:**
```javascript
// Use animateCamera instead of setRegion
mapRef.current.animateCamera({
  center: newLocation,
  zoom: 16,
}, { duration: 1000 });
```

---

## 📈 Performance Optimization

### 1. Throttle Location Updates (Driver)
```javascript
let lastUpdate = 0;
const MIN_UPDATE_INTERVAL = 5000; // 5 seconds

_handleLocationUpdate(location) {
  const now = Date.now();
  if (now - lastUpdate < MIN_UPDATE_INTERVAL) {
    return; // Skip this update
  }
  lastUpdate = now;
  
  SocketService.updateLocation(location);
}
```

### 2. Limit Location History (User)
```javascript
// Keep only last 50 points for polyline
setLocationHistory(prev => [...prev.slice(-50), newLocation]);
```

### 3. Debounce Map Updates
```javascript
import { debounce } from 'lodash';

const updateMapDebounced = debounce((location) => {
  mapRef.current?.animateCamera({ center: location });
}, 1000);
```

---

## 🎉 Ready to Deploy!

Your apps are now ready with Uber-like real-time location tracking!

### Checklist:
- ✅ Driver app sends location every 5 seconds
- ✅ Background service keeps running when driver uses Google Maps
- ✅ Socket stays connected in background
- ✅ User sees smooth real-time updates
- ✅ Session restoration on app resume
- ✅ Trip status updates (cancellation, completion)
- ✅ Heartbeat mechanism for connection health
- ✅ Auto-reconnection on disconnect

**Happy Coding! 🚀**
