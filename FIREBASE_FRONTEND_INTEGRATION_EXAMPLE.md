# 🔥 Firebase Frontend Integration Guide

## Complete Implementation for Getting JWT Tokens

### **1. Install Firebase SDK**

```bash
# For React/Web
npm install firebase

# For React Native
npm install @react-native-firebase/app @react-native-firebase/auth
```

### **2. Firebase Configuration**

```javascript
// firebase.js
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';

const firebaseConfig = {
  apiKey: "your-api-key",
  authDomain: "freelancing-platform-v2.firebaseapp.com",
  projectId: "freelancing-platform-v2",
  storageBucket: "freelancing-platform-v2.firebasestorage.app",
  messagingSenderId: "your-sender-id",
  appId: "your-app-id"
};

const app = initializeApp(firebaseConfig);
export const auth = getAuth(app);
```

### **3. Authentication Service**

```javascript
// authService.js
import { auth } from './firebase';
import { signInWithPhoneNumber, RecaptchaVerifier } from 'firebase/auth';

class AuthService {
  constructor() {
    this.recaptchaVerifier = null;
  }

  // Initialize reCAPTCHA
  initRecaptcha(containerId) {
    this.recaptchaVerifier = new RecaptchaVerifier(containerId, {
      size: 'invisible',
      callback: (response) => {
        console.log('reCAPTCHA solved');
      }
    }, auth);
  }

  // Send OTP
  async sendOTP(phoneNumber) {
    try {
      if (!this.recaptchaVerifier) {
        throw new Error('reCAPTCHA not initialized');
      }

      const confirmationResult = await signInWithPhoneNumber(
        auth, 
        phoneNumber, 
        this.recaptchaVerifier
      );
      
      return confirmationResult;
    } catch (error) {
      console.error('Error sending OTP:', error);
      throw error;
    }
  }

  // Verify OTP and get Firebase ID Token
  async verifyOTP(confirmationResult, otp) {
    try {
      const result = await confirmationResult.confirm(otp);
      const user = result.user;
      
      // Get Firebase ID Token
      const firebaseIdToken = await user.getIdToken();
      
      return {
        user,
        firebaseIdToken,
        phoneNumber: user.phoneNumber
      };
    } catch (error) {
      console.error('Error verifying OTP:', error);
      throw error;
    }
  }

  // Get backend JWT token
  async getBackendJWT(firebaseIdToken, role = 'client') {
    try {
      const response = await fetch('https://freelancing-platform-backend-backup.onrender.com/api/real-firebase-auth/authenticate', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          idToken: firebaseIdToken,
          role: role
        })
      });

      const data = await response.json();
      
      if (!data.success) {
        throw new Error(data.message);
      }

      return {
        backendJWT: data.data.token,
        user: data.data.user,
        firebase: data.data.firebase
      };
    } catch (error) {
      console.error('Error getting backend JWT:', error);
      throw error;
    }
  }

  // Complete authentication flow
  async authenticate(phoneNumber, otp, role = 'client') {
    try {
      // Step 1: Send OTP
      const confirmationResult = await this.sendOTP(phoneNumber);
      
      // Step 2: Verify OTP and get Firebase ID Token
      const { firebaseIdToken, user } = await this.verifyOTP(confirmationResult, otp);
      
      // Step 3: Get backend JWT token
      const { backendJWT } = await this.getBackendJWT(firebaseIdToken, role);
      
      return {
        firebaseIdToken,
        backendJWT,
        user,
        success: true
      };
    } catch (error) {
      return {
        error: error.message,
        success: false
      };
    }
  }
}

export default new AuthService();
```

### **4. React Component Example**

```jsx
// LoginComponent.jsx
import React, { useState, useEffect } from 'react';
import AuthService from './authService';

const LoginComponent = () => {
  const [phoneNumber, setPhoneNumber] = useState('+919009009000');
  const [otp, setOtp] = useState('909090');
  const [role, setRole] = useState('client');
  const [loading, setLoading] = useState(false);
  const [result, setResult] = useState(null);

  useEffect(() => {
    // Initialize reCAPTCHA
    AuthService.initRecaptcha('recaptcha-container');
  }, []);

  const handleLogin = async () => {
    setLoading(true);
    setResult(null);

    try {
      const authResult = await AuthService.authenticate(phoneNumber, otp, role);
      setResult(authResult);
    } catch (error) {
      setResult({ error: error.message, success: false });
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      <h2>Firebase Authentication Test</h2>
      
      <div>
        <label>Phone Number:</label>
        <input 
          type="text" 
          value={phoneNumber} 
          onChange={(e) => setPhoneNumber(e.target.value)}
          placeholder="+919009009000"
        />
      </div>

      <div>
        <label>OTP:</label>
        <input 
          type="text" 
          value={otp} 
          onChange={(e) => setOtp(e.target.value)}
          placeholder="909090"
        />
      </div>

      <div>
        <label>Role:</label>
        <select value={role} onChange={(e) => setRole(e.target.value)}>
          <option value="client">Client</option>
          <option value="freelancer">Freelancer</option>
        </select>
      </div>

      <button onClick={handleLogin} disabled={loading}>
        {loading ? 'Authenticating...' : 'Login with Firebase'}
      </button>

      {/* reCAPTCHA container */}
      <div id="recaptcha-container"></div>

      {result && (
        <div>
          <h3>Authentication Result:</h3>
          {result.success ? (
            <div>
              <p>✅ Authentication Successful!</p>
              <p><strong>Firebase ID Token:</strong> {result.firebaseIdToken.substring(0, 50)}...</p>
              <p><strong>Backend JWT Token:</strong> {result.backendJWT.substring(0, 50)}...</p>
              <p><strong>User ID:</strong> {result.user._id}</p>
              <p><strong>Phone:</strong> {result.user.phone}</p>
              <p><strong>Role:</strong> {result.user.role}</p>
            </div>
          ) : (
            <div>
              <p>❌ Authentication Failed:</p>
              <p>{result.error}</p>
            </div>
          )}
        </div>
      )}
    </div>
  );
};

export default LoginComponent;
```

### **5. Using JWT Token for API Calls**

```javascript
// apiService.js
class ApiService {
  constructor() {
    this.baseURL = 'https://freelancing-platform-backend-backup.onrender.com';
    this.token = null;
  }

  setToken(token) {
    this.token = token;
  }

  async makeRequest(endpoint, options = {}) {
    const url = `${this.baseURL}${endpoint}`;
    const headers = {
      'Content-Type': 'application/json',
      ...options.headers
    };

    if (this.token) {
      headers.Authorization = `Bearer ${this.token}`;
    }

    const response = await fetch(url, {
      ...options,
      headers
    });

    const data = await response.json();
    
    if (!response.ok) {
      throw new Error(data.message || 'API request failed');
    }

    return data;
  }

  // Example API calls
  async getProfile() {
    return this.makeRequest('/api/client/profile');
  }

  async postJob(jobData) {
    return this.makeRequest('/api/client/jobs', {
      method: 'POST',
      body: JSON.stringify(jobData)
    });
  }

  async getJobs() {
    return this.makeRequest('/api/freelancer/jobs/available');
  }
}

export default new ApiService();
```

### **6. Complete Usage Example**

```javascript
// App.js
import React, { useState } from 'react';
import AuthService from './authService';
import ApiService from './apiService';

const App = () => {
  const [isAuthenticated, setIsAuthenticated] = useState(false);
  const [user, setUser] = useState(null);

  const handleLogin = async () => {
    const result = await AuthService.authenticate('+919009009000', '909090', 'client');
    
    if (result.success) {
      // Set JWT token for API calls
      ApiService.setToken(result.backendJWT);
      
      // Update app state
      setIsAuthenticated(true);
      setUser(result.user);
      
      console.log('✅ Ready to make API calls with JWT token!');
    } else {
      console.error('❌ Authentication failed:', result.error);
    }
  };

  const testApiCall = async () => {
    try {
      const profile = await ApiService.getProfile();
      console.log('Profile:', profile);
    } catch (error) {
      console.error('API call failed:', error);
    }
  };

  return (
    <div>
      {!isAuthenticated ? (
        <button onClick={handleLogin}>
          Login with Firebase
        </button>
      ) : (
        <div>
          <h2>Welcome, {user?.phone}!</h2>
          <button onClick={testApiCall}>
            Test API Call
          </button>
        </div>
      )}
    </div>
  );
};

export default App;
```

## 🎯 Key Points

1. **Firebase ID Token**: Generated by Firebase SDK after OTP verification
2. **Backend JWT Token**: Generated by your backend after verifying Firebase ID token
3. **Use Backend JWT**: For all subsequent API calls to your backend
4. **Token Storage**: Store the backend JWT token securely (localStorage, AsyncStorage, etc.)
5. **Token Refresh**: Firebase ID tokens expire, but your backend JWT can have longer expiry

## 🔧 Testing

1. Add `+919009009000` to Firebase testing list
2. Set OTP to `909090`
3. Use the above code to authenticate
4. Get both Firebase ID token and backend JWT token
5. Use backend JWT token for all API calls

This setup will give you complete Firebase authentication with JWT tokens for your app! 🚀
