# 🗂️ Project Configuration Summary

## 📁 `app/backend/server.py`

```python
if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=5010)
```

---

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

---

## 📁 `app/frontend/package.json`

```json
"scripts": {
  "start": "HOST=0.0.0.0 PORT=8010 craco start",
  "build": "craco build",
  "test": "craco test"
}
```

---

## 📁 `app/backend/.env`

```env
MONGO_URL="mongodb://localhost:27017"
DB_NAME="spearon_finance_dashboard"
```

---

## 📁 `app/frontend/.env`

```env
REACT_APP_BACKEND_URL=/server
WDS_SOCKET_PORT=443
```

