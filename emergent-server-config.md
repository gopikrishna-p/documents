# 🗂️ Project Configuration Summary

## 📁 `app/backend/server.py`

```python
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=5010)
```



## 📁 `app/frontend/craco.config.js`

```js
devServer: (devServerConfig) => {
  devServerConfig.proxy = {
    '/server': {
      target: 'http://0.0.0.0:5010',
      changeOrigin: true,
      pathRewrite: { '^/server': '' },
    },
  };
  return devServerConfig;
},
```



## 📁 `app/frontend/package.json`

```json
"scripts": {
  "start": "HOST=0.0.0.0 PORT=8010 craco start",
  "build": "craco build",
  "test": "craco test"
}
```



## 📁 `app/backend/.env`

```env
MONGO_URL="mongodb://localhost:27017"
DB_NAME="spearon_finance_dashboard"
```



## 📁 `app/frontend/.env`

```env
REACT_APP_BACKEND_URL=/server
WDS_SOCKET_PORT=443
```


```env
REACT_APP_BACKEND_URL = /server
WDS_SOCKET_PORT = 443
const API = process.env.NODE_ENV === 'development' ? '/api' : `${BACKEND_URL}/api`;
const API = `${process.env.REACT_APP_BACKEND_URL}/api`;
```


## `Replace the existing getBackendUrl and API setup in your Dashboard component with this`

```env
// 

const getBackendUrl = () => {
  const hostname = window.location.hostname;
  
  // Production environment detection
  if (hostname !== 'localhost' && hostname !== '127.0.0.1') {
    return `${window.location.origin}/server`;
  }
  
  // Local development - use environment variable or fallback
  return process.env.REACT_APP_BACKEND_URL || 'http://localhost:5020';
};

const BACKEND_URL = getBackendUrl();
const API = `${BACKEND_URL}/api`;

console.log('Environment:', {
  hostname: window.location.hostname,
  BACKEND_URL,
  API,
  env: process.env.REACT_APP_BACKEND_URL
});

// Rest of your component logic remains the same...
```


