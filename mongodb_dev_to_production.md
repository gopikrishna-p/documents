Here’s your **complete “User Migration Guide: Development to Production”** rewritten and formatted as a **professional Markdown document** — suitable for project documentation or README files.

---

# 🧭 User Migration Guide: Development → Production (MongoDB)

This guide explains how to safely **migrate MongoDB user data** from your **development environment** to a **production server** using `mongodump/mongorestore` or `mongoexport/mongoimport`.

---

## 🚀 Option 1: MongoDB Export/Import (Recommended)

### **Step 1: Export from Development**

#### 🔹 Export Using `mongodump` (Binary Format)

```bash
# Export specific collection (users)
mongodump --uri="mongodb://localhost:27017" \
--db=angel_ir_platform \
--collection=users \
--out=./backup
```
or
```bash
# Export users collection
mongodump --uri="mongodb://localhost:27017" --db=angel_ir_platform --collection=users --out=./backup
```

This creates a binary backup inside the `./backup` directory, which can be restored directly to another MongoDB instance.

---

#### 🔹 Export Using `mongoexport` (JSON Format)

```bash
# Export collection as JSON for readability
mongoexport --uri="mongodb://localhost:27017" \
--db=angel_ir_platform \
--collection=users \
--out=users.json \
--jsonArray
```
or
```bash
# Or export as JSON (more readable)
mongoexport --uri="mongodb://localhost:27017" --db=angel_ir_platform --collection=users --out=users.json --jsonArray
```

---

### **Step 2: Import to Production**

#### 🔹 Import Using `mongorestore`

```bash
mongorestore --uri="mongodb://PRODUCTION_HOST:27017" \
--db=angel_ir_platform \
--collection=users \
./backup/angel_ir_platform/users.bson
```
or
```bash
# Using mongorestore
mongorestore --uri="mongodb://PRODUCTION_HOST:27017" --db=angel_ir_platform --collection=users ./backup/angel_ir_platform/users.bson
```


---

#### 🔹 Import Using `mongoimport` (JSON File)

```bash
mongoimport --uri="mongodb://PRODUCTION_HOST:27017" \
--db=angel_ir_platform \
--collection=users \
--file=users.json \
--jsonArray
```
or
```bash
# Or using mongoimport (if you exported as JSON)
mongoimport --uri="mongodb://PRODUCTION_HOST:27017" --db=angel_ir_platform --collection=users --file=users.json --jsonArray
```


> ⚠️ **Note:** Replace `PRODUCTION_HOST` with the actual production server’s IP address or domain name.
To find that PRODUCTION_HOST use
```bash
sudo netstat -tulnp | grep mongod
```
---
### **Step 1: Export All Collections (Development Server)**
#### 🔹 Export Entire Database (Recommended)
```bash
mongodump --uri="mongodb://localhost:27017" \
--db=angel_ir_platform \
--out=./backup
```
### **Step 2: Import to Production Server**
#### 🔹 Using `mongorestore` (BSON Backup)

```bash
mongorestore --uri="mongodb://PRODUCTION_HOST:27017" \
--db=angel_ir_platform \
--drop \
./backup/angel_ir_platform
```
PRODUCTION_HOST: 127.0.0.1

**Explanation:**

* `--drop` → Drops (deletes) existing collections **before restoring**, ensuring duplicates are replaced.
* `--db=angel_ir_platform` → Restores into the same database name.
* Replace `PRODUCTION_HOST` with the **actual IP or hostname** of your production server.

✅ This ensures all data is overwritten cleanly.

---


## 🧩 Basic MongoDB Commands

### **Database Operations**

```js
show dbs              // Show all databases
use mydb              // Switch or create a new database
db                    // Show current database
show collections      // Show all collections in the current database
```

---

### **Insert Operations**

```js
db.users.insertOne({ name: "Gopi", age: 24, city: "Bangalore" })

db.users.insertMany([
  { name: "Teja", age: 22, city: "Vijayawada" },
  { name: "Aravind", age: 25, city: "Chennai" }
])
```

---

### **Query Operations**

```js
db.users.find()                        // Get all documents
db.users.find().pretty()               // Pretty print
db.users.find({ name: "Gopi" })        // Filter by field
db.users.find({ age: { $gt: 22 } })    // age > 22
db.users.find({ age: { $gte: 22, $lte: 25 } }) // 22 ≤ age ≤ 25
db.users.find({ city: { $in: ["Bangalore", "Chennai"] } })
db.users.find({}, { name: 1, _id: 0 }) // Show only name field
```

---

### **Update Operations**

```js
db.users.updateOne(
  { name: "Gopi" },
  { $set: { age: 25, city: "Hyderabad" } }
)

db.users.updateMany(
  { city: "Bangalore" },
  { $set: { verified: true } }
)
```

---

### **Delete Operations**

```js
db.users.deleteOne({ name: "Teja" })
db.users.deleteMany({ city: "Vijayawada" })
```

---

### **Sorting, Limiting & Counting**

```js
db.users.find().sort({ age: 1 })       // 1 = ascending, -1 = descending
db.users.find().limit(3)               // Limit to first 3 documents
db.users.countDocuments({ city: "Hyderabad" })
```

---

### **Database & Collection Management**

```js
db.dropDatabase()                      // Delete current database
db.users.drop()                        // Drop collection
db.users.stats()                       // Collection statistics
db.serverStatus()                      // MongoDB server info
```

---

## 🧰 Example Import Command (Using IP Address)

If your production server IP is `192.168.1.50`, use:

```bash
mongoimport --uri="mongodb://192.168.1.50:27017" \
--db=angel_ir_platform \
--collection=users \
--file=users.json \
--jsonArray
```

---

## 🔍 Checking MongoDB Listening Port

To verify MongoDB is running and listening on port `27017`:

```bash
sudo netstat -tulnp | grep mongod
```

Expected output:

```
tcp   LISTEN 0  4096  127.0.0.1:27017  0.0.0.0:*  users:(("mongod",pid=817,fd=14))
```

If you see `127.0.0.1:27017`, MongoDB is only accessible locally.
To allow remote access, update your `/etc/mongod.conf` file:

```yaml
net:
  bindIp: 0.0.0.0
```

Then restart MongoDB:

```bash
sudo systemctl restart mongod
```

---

## 🛡️ Recommended Security Settings

* Enable authentication:

  ```yaml
  security:
    authorization: enabled
  ```
* Create an admin user:

  ```js
  use admin
  db.createUser({
    user: "admin",
    pwd: "StrongPassword123",
    roles: [ { role: "root", db: "admin" } ]
  })
  ```
* Allow port 27017 through firewall:

  ```bash
  sudo ufw allow 27017
  ```

---

## ✅ Summary

| Task                 | Command                                   |              |
| -------------------- | ----------------------------------------- | ------------ |
| Export (Binary)      | `mongodump`                               |              |
| Export (JSON)        | `mongoexport`                             |              |
| Import (Binary)      | `mongorestore`                            |              |
| Import (JSON)        | `mongoimport`                             |              |
| Verify Port          | `sudo netstat -tulnp                      | grep mongod` |
| Enable Remote Access | Edit `/etc/mongod.conf → bindIp: 0.0.0.0` |              |

---

> 📝 **Tip:** Always take a backup before migration and test the imported data in production with a staging environment first.

---
