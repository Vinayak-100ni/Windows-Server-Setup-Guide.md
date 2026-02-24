# 🍃 MongoDB Restore Process on Windows Server

> Complete step-by-step guide to restore a MongoDB database dump on Windows Server.

---

## 📋 Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1: Install MongoDB Server](#step-1-install-mongodb-server)
- [Step 2: Install MongoDB Database Tools](#step-2-install-mongodb-database-tools)
- [Step 3: Start MongoDB](#step-3-start-mongodb)
- [Step 4: Prepare Dump Files](#step-4-prepare-dump-files)
- [Step 5: Restore the Database](#step-5-restore-the-database)
- [Step 6: Verify the Restore](#step-6-verify-the-restore)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Windows Server (2008 R2 or later)
- MongoDB ZIP file (e.g. `mongodb-windows-8.2.5.zip`)
- MongoDB Database Tools ZIP
- MongoDB Shell (mongosh) ZIP
- Your database dump folder (containing `.bson` and `.metadata.json` files)

---

## Step 1: Install MongoDB Server

1. Extract the MongoDB ZIP to:
   ```
   C:\Program Files\MongoDB\Server\8.2\
   ```

2. Create required data and log directories:
   ```cmd
   mkdir C:\data\db
   mkdir C:\data\log
   ```

---

## Step 2: Install MongoDB Database Tools

MongoDB Tools (`mongorestore`, `mongodump`, etc.) must be installed **separately**.

1. Download from: https://www.mongodb.com/try/download/database-tools
   - Platform: **Windows**
   - Package: **ZIP**

2. Extract to:
   ```
   C:\Program Files\MongoDB\Tools\
   ```

3. Verify the structure:
   ```
   C:\Program Files\MongoDB\Tools\
   └── bin\
       ├── mongorestore.exe
       ├── mongodump.exe
       └── ...
   ```

---

## Step 3: Start MongoDB

Open **Command Prompt as Administrator** and run:

```cmd
"C:\Program Files\MongoDB\Server\8.2\bin\mongod.exe" --dbpath "C:\data\db"
```

> ⚠️ **Keep this window open.** Closing it will stop MongoDB.

### Optional: Install as a Windows Service

```cmd
"C:\Program Files\MongoDB\Server\8.2\bin\mongod.exe" --install --serviceName "MongoDB" --dbpath "C:\data\db" --logpath "C:\data\log\mongod.log"

net start MongoDB
```

---

## Step 4: Prepare Dump Files

If your dump ZIP contains `.bson` files **directly** (flat structure), you must place them inside a subfolder named after your database.

### Create the correct folder structure

```cmd
mkdir "C:\Users\Administrator\Downloads\dump\DigiSparshDBUAT"

move "C:\Users\Administrator\Downloads\DigiSparshDBUAT\*" "C:\Users\Administrator\Downloads\dump\DigiSparshDBUAT\"
```

### Expected structure

```
C:\Users\Administrator\Downloads\dump\
└── DigiSparshDBUAT\
    ├── abhalogs.bson
    ├── abhalogs.metadata.json
    ├── approvedloans.bson
    ├── approvedloans.metadata.json
    └── ... (all other collections)
```

---

## Step 5: Restore the Database

Open a **second** Command Prompt as Administrator and run:

```cmd
"C:\Program Files\MongoDB\Tools\bin\mongorestore.exe" --drop "C:\Users\Administrator\Downloads\dump"
```

> ⚠️ Do **NOT** add a trailing backslash `\` at the end of the path — it causes a syntax error.

### With Authentication

```cmd
"C:\Program Files\MongoDB\Tools\bin\mongorestore.exe" --drop --username admin --password yourpassword --authenticationDatabase admin "C:\Users\Administrator\Downloads\dump"
```

### Expected Output

```
2026-02-24T09:XX:XX  restoring DigiSparshDBUAT.approvedloans
2026-02-24T09:XX:XX  restoring DigiSparshDBUAT.users
...
2026-02-24T09:XX:XX  X document(s) restored successfully. 0 document(s) failed to restore.
```

---

## Step 6: Verify the Restore

1. Download MongoDB Shell from: https://www.mongodb.com/try/download/shell
2. Extract and run:

```cmd
"C:\Users\Administrator\Downloads\mongosh\bin\mongosh.exe"
```

3. Inside the shell, verify:

```js
show dbs
use DigiSparshDBUAT
show collections
```

You should see **DigiSparshDBUAT** listed with all its collections. ✅

---

## Troubleshooting

### ❌ `The system cannot find the path specified`
**Cause:** Incorrect path to `mongorestore.exe` or dump folder.  
**Fix:** Check paths using:
```cmd
dir "C:\Program Files\MongoDB\Tools\bin"
dir "C:\Users\Administrator\Downloads\"
```

---

### ❌ `Filename, directory name, or volume label syntax is incorrect`
**Cause:** Trailing backslash `\` at end of the dump path.  
**Fix:** Remove the trailing backslash:
```cmd
# ❌ Wrong
"C:\Users\Administrator\Downloads\dump\"

# ✅ Correct
"C:\Users\Administrator\Downloads\dump"
```

---

### ❌ `don't know what to do with file — skipping`
**Cause:** `.bson` files are directly in the folder without a database subfolder.  
**Fix:** Move all files into a subfolder named after your database (see [Step 4](#step-4-prepare-dump-files)).

---

### ❌ `Data directory C:\data\db not found`
**Cause:** The data directory doesn't exist.  
**Fix:**
```cmd
mkdir C:\data\db
```

---

### ❌ `The service name is invalid` (net start MongoDB)
**Cause:** MongoDB is not installed as a Windows service.  
**Fix:** Either install it as a service (see [Step 3](#step-3-start-mongodb)) or start `mongod.exe` manually.

---

### ❌ `mongorestore is not recognized`
**Cause:** MongoDB Tools not installed or not in PATH.  
**Fix:** Use the full path to `mongorestore.exe` as shown in [Step 5](#step-5-restore-the-database), or add `C:\Program Files\MongoDB\Tools\bin` to your system PATH.

---

## 📁 Project Structure Summary

```
MongoDB Setup
├── Server          → C:\Program Files\MongoDB\Server\8.2\bin\mongod.exe
├── Tools           → C:\Program Files\MongoDB\Tools\bin\mongorestore.exe
├── Shell           → C:\Users\Administrator\Downloads\mongosh\bin\mongosh.exe
├── Data            → C:\data\db
├── Logs            → C:\data\log
└── Dump            → C:\Users\Administrator\Downloads\dump\DigiSparshDBUAT\
```

---

## 📝 Notes

- Always run **CMD as Administrator** for MongoDB commands
- You need **two CMD windows** — one for the server, one for the restore
- MongoDB version used: **8.2.5**
- Database restored: **DigiSparshDBUAT**

---

*Guide based on actual restore session — Windows Server 2026, MongoDB 8.2.5*
