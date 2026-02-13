# Self-Hosted DB Sync Setup Guide

This guide covers setting up Logseq's self-hosted DB sync server, first locally for testing, then deploying to a VPS with authentication.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Part 1: Local Testing](#part-1-local-testing)
- [Part 2: VPS Deployment](#part-2-vps-deployment)
- [Part 3: Authentication Setup](#part-3-authentication-setup)
- [Troubleshooting](#troubleshooting)

**📘 Additional Guides:**
- [CUSTOM_COGNITO_SETUP.md](CUSTOM_COGNITO_SETUP.md) - Complete guide for using your own AWS Cognito User Pool

---

## ⚠️ Important: Environment Variables & Port Configuration

**CRITICAL: DB Sync Only Works with DB Graphs (SQLite-based), NOT Markdown/File Graphs!**

When testing locally, you **MUST**:
1. **Create a NEW DB graph** (not use an existing markdown graph)
2. **Check "Use Logseq Sync?"** when creating the graph
3. **Be logged in** to your Logseq account

**Key Points:**
1. **Use `start.sh` for local testing** - it has everything you need hardcoded
2. **The server port is 8787** - matches config.cljs (line 59)
3. **Use `yarn dev-electron-app`** - not `yarn electron-start` or `yarn electron:dev`
4. **Set `DB_SYNC_LOG_LEVEL=debug`** to see detailed logs
5. **`ENABLE_DB_SYNC_LOCAL=true` when building client** - enables sync UI for local testing

**Quick Start Commands (3 Terminals Required):**
```bash
# Terminal 1: Start sync server
cd deps/db-sync
yarn install                # First time only
yarn build:node-adapter     # First time only, or after code changes
./start.sh                  # Uses port 8787 by default

# Terminal 2: Build Logseq client with local sync enabled
cd /Users/ashutosh/Documents/GITS/logseq
# Use cross-env for cross-platform compatibility
yarn cross-env ENABLE_DB_SYNC_LOCAL=true yarn watch

# Terminal 3: Run Logseq desktop app (after Terminal 2 build completes)
cd /Users/ashutosh/Documents/GITS/logseq
yarn dev-electron-app
```

---

## Prerequisites

### Required Software
- **Node.js**: >= 22.20.0
- **Yarn**: Latest version
- **Java**: OpenJDK 20+ (for building Logseq client)
- **Clojure**: 1.12.4+

### Check Installation
```bash
node --version    # Should be >= 22.20.0
yarn --version
java --version
clojure --version
```

---

## Part 1: Local Testing

### Step 1: Build the Node.js Adapter Server

```bash
cd /Users/ashutosh/Documents/GITS/logseq/deps/db-sync
yarn install
yarn build:node-adapter
```

This creates the server in `deps/db-sync/dist/node-adapter/`.

### Step 2: Understand Configuration

**How the Node.js adapter gets its configuration:**

1. **`start.sh`** - Has hardcoded defaults (lines 8-10), easiest for local testing
2. **Shell exports** - Export vars before running, overrides start.sh defaults
3. **Inline vars** - Pass directly when running node: `DB_SYNC_PORT=8787 node ...`

**For local testing:**

**Option A: Use start.sh script** (RECOMMENDED - Easiest):

The `start.sh` script has **hardcoded default values** inside it:
```bash
cd /Users/ashutosh/Documents/GITS/logseq/deps/db-sync
cat start.sh  # View lines 8-10 for hardcoded Cognito defaults
```

Hardcoded defaults in `start.sh` (lines 8-10):
```bash
: "${COGNITO_ISSUER:=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_dtagLnju8}"
: "${COGNITO_CLIENT_ID:=69cs1lgme7p8kbgld8n5kseii6}"
: "${COGNITO_JWKS_URL:=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_dtagLnju8/.well-known/jwks.json}"
```

This means:
- Port: `8787`
- Log level: `debug`
- Cognito: Test pool (`us-east-1_dtagLnju8`)
- Data directory: `./data`

**Option B: Override defaults by exporting before running start.sh**:

```bash
# Export these BEFORE running start.sh to override its hardcoded defaults
export DB_SYNC_PORT=8787
export DB_SYNC_DATA_DIR=./sync-data
export DB_SYNC_LOG_LEVEL=debug  # Options: trace, debug, info, warn, error

# To use different Cognito pool:
export COGNITO_ISSUER=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_YOUR_POOL
export COGNITO_CLIENT_ID=your-client-id
export COGNITO_JWKS_URL=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_YOUR_POOL/.well-known/jwks.json

# Then run start.sh - it will use your exported values instead of its defaults
./start.sh
```

**Option C: Inline environment variables** (skip start.sh entirely):

```bash
# Pass all vars inline when running node directly
DB_SYNC_PORT=8787 \
DB_SYNC_LOG_LEVEL=debug \
COGNITO_ISSUER=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_dtagLnju8 \
COGNITO_CLIENT_ID=69cs1lgme7p8kbgld8n5kseii6 \
COGNITO_JWKS_URL=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_dtagLnju8/.well-known/jwks.json \
node worker/dist/node-adapter.js
```

**Summary:** `start.sh` has everything hardcoded, so just run `./start.sh` for defaults.

### Step 3: Start the Node.js Adapter Server

Choose one of these methods based on your Step 2 choice:

**Option A: Using start.sh script (RECOMMENDED):**

```bash
cd /Users/ashutosh/Documents/GITS/logseq/deps/db-sync
./start.sh
```

**Option B: Override start.sh defaults** (after exporting your custom values):

```bash
# After exporting custom values (see Step 2 Option B)
cd /Users/ashutosh/Documents/GITS/logseq/deps/db-sync
./start.sh
```

**Option C: Inline vars with node directly** (skip start.sh):

```bash
cd /Users/ashutosh/Documents/GITS/logseq/deps/db-sync
DB_SYNC_PORT=8787 DB_SYNC_LOG_LEVEL=debug \
COGNITO_ISSUER=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_dtagLnju8 \
COGNITO_CLIENT_ID=69cs1lgme7p8kbgld8n5kseii6 \
COGNITO_JWKS_URL=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_dtagLnju8/.well-known/jwks.json \
node worker/dist/node-adapter.js
```

**Expected Output:**
```
Starting Logseq sync...
Logseq sync listening on port 8787
[DEBUG] Server initialized with SQLite storage
[DEBUG] Data directory: /path/to/deps/db-sync/data
```

**Troubleshooting:**
- **"port already in use"**: Change to a different port or kill the process using port 8787
- **"Cannot find module"**: Run `yarn install` first
- **No debug logs**: Make sure you set `DB_SYNC_LOG_LEVEL=debug`
- **Permission denied on start.sh**: Run `chmod +x start.sh` first

**Keep this terminal running.**

### Step 4: Build Logseq Client with Local Sync Enabled

Open a **new terminal** (keep Terminal 1 running with the server):

```bash
cd /Users/ashutosh/Documents/GITS/logseq

# Set environment variable and start development build
# Using cross-env for cross-platform compatibility
yarn cross-env ENABLE_DB_SYNC_LOCAL=true yarn watch

# Alternative (Unix/macOS only):
# ENABLE_DB_SYNC_LOCAL=true yarn watch
```

**What this does:**
- Compiles ClojureScript with local sync URLs
- Connects to `ws://127.0.0.1:8787` instead of production
- Watches for file changes (auto-recompiles on edit)
- Compiles app, db-worker, inference-worker, and electron targets

**Expected Output:**
```
[:app] Compiling ...
[:db-worker] Compiling ...
[:inference-worker] Compiling ...
[:electron] Compiling ...
[:app] Build completed. (XXX files, XXX compiled, 0 warnings, XX.XXs)
```

**Build time:** First build takes 2-5 minutes. Subsequent changes rebuild in seconds.

**Wait for "Build completed successfully"** message before proceeding to Step 5.

**Troubleshooting:**
- **"command not found: clojure"**: Install Clojure CLI tools
- **"Java not found"**: Install Java 20+ (OpenJDK recommended)
- **Build errors**: Run `yarn clean` then try again
- **Slow build**: Normal for first build; subsequent builds are faster
- **On Windows**: If inline env vars don't work, use `yarn cross-env ENABLE_DB_SYNC_LOCAL=true yarn watch`

**Keep this terminal running** - it will auto-rebuild when you edit source files.

### Step 5: Launch Logseq Desktop App

Open a **third terminal** (keep Terminals 1 & 2 running):

```bash
cd /Users/ashutosh/Documents/GITS/logseq
yarn dev-electron-app
```

**What this does:**
- Launches Electron with development settings
- Opens Logseq desktop app
- Connects to your local sync server

**The app should open** within 10-30 seconds.

**Troubleshooting:**
- **App doesn't open**: Check Terminal 2 build completed successfully
- **White screen**: Wait 30 seconds or check browser console (View > Toggle Developer Tools)
- **"electron command not found"**: Run `yarn install` in the root directory
- **App crashes on startup**: Check for build errors in Terminal 2

### Step 6: Test Local Sync End-to-End

Now test that sync is working properly:

#### 6.1 Create a New DB Graph with Sync Enabled

⚠️ **CRITICAL**: DB sync ONLY works with **DB graphs** (SQLite-based), NOT traditional markdown/file graphs!

1. In the Logseq app, **log in to your Logseq account** (required for sync authentication)
2. Click **"Add a DB graph"** (or look for "Create new graph" button)
3. In the "Create a new graph" dialog:
   - **Enter graph name**: e.g., "test-sync-graph"
   - **Check "Use Logseq Sync?"** checkbox (this enables RTC sync)
   - Optionally check "Encrypt graph data" for E2EE
   - Click **"Submit"**
4. Wait for the graph to be created (10-30 seconds)

**Important Notes:**
- Don't try to use an existing markdown graph - sync won't work
- The "Use Logseq Sync?" checkbox appears because `ENABLE_DB_SYNC_LOCAL=true` bypasses the RTC group check
- DB graphs use SQLite format and have URLs like `logseq_db_test-graph`
- File/markdown graphs are directory paths and don't support sync

#### 6.2 Verify Server Connection

Check **Terminal 1** (server logs). You should see:
```
[DEBUG] WebSocket connection from 127.0.0.1
[DEBUG] Client connected: <connection-id>
[DEBUG] Authenticating user...
[DEBUG] User authenticated: <user-id>
```

If you **don't see these logs**:
- Make sure you set `DB_SYNC_LOG_LEVEL=debug` when starting the server
- Check the app's developer console (View > Toggle Developer Tools > Console tab)
- Look for WebSocket connection errors

#### 6.3 Enable Sync in Logseq

1. **Open Settings** → **Sync** tab
2. **Check if sync is available** - you should see sync options
3. **Enable sync for the graph** (if not already enabled)

#### 6.4 Make Changes and Verify Sync

1. **Create a new page**: Type `[[Test Page]]` and press Enter
2. **Add some blocks**: Type content and press Enter
3. **Check server logs** in Terminal 1:
   ```
   [DEBUG] Received message: type=:sync/push-transaction
   [DEBUG] Processing transaction for graph: <graph-id>
   [DEBUG] Transaction saved: <tx-id>
   [DEBUG] Broadcasting updates to 1 clients
   ```

#### 6.5 Test Multi-Device Sync (Optional but Recommended)

To thoroughly test sync, open a second Logseq instance:

1. **Open a 4th terminal**:
   ```bash
   cd /Users/ashutosh/Documents/GITS/logseq
   yarn dev-electron-app
   ```

2. **Open the SAME graph** in the second instance (use the same folder)

3. **Make changes in Instance 1**:
   - Edit a block
   - Create a new page
   
4. **Check Instance 2** - changes should appear automatically within 1-2 seconds

5. **Make changes in Instance 2** and verify they appear in Instance 1

6. **Check server logs** - you should see:
   ```
   [DEBUG] Broadcasting updates to 2 clients
   [DEBUG] Client received update: <client-id-1>
   [DEBUG] Client received update: <client-id-2>
   ```

#### 6.6 Verify Sync Data Persistence

Your sync data is stored locally in the server's data directory:

```bash
# Check sync data files (in a new terminal)
ls -la /Users/ashutosh/Documents/GITS/logseq/deps/db-sync/data/

# You should see:
# - SQLite database files (.db)
# - Graph metadata files
# - Transaction logs
```

#### 6.7 Test Sync After Restart

1. **Close all Logseq instances**
2. **Keep server running** (Terminal 1)
3. **Reopen Logseq**: `yarn dev-electron-app`
4. **Open the same graph**
5. **Verify all your changes are still there**

This confirms data persistence is working correctly.

### Step 7: Inspect Sync Data (Optional)

Your sync data is stored locally by default in `deps/db-sync/data/`:

```bash
# View data directory structure
ls -la /Users/ashutosh/Documents/GITS/logseq/deps/db-sync/data/

# Example output:
# graphs/              - Per-graph directories
# ├── <graph-id-1>/
# │   ├── graph.db     - SQLite database with transactions
# │   └── assets/      - Attached files/images
# └── <graph-id-2>/
#     └── graph.db
```

**Inspect database contents** (requires sqlite3):
```bash
# Install sqlite3 if needed
brew install sqlite3  # macOS

# Query graph metadata
sqlite3 deps/db-sync/data/graphs/<graph-id>/graph.db "SELECT * FROM transactions LIMIT 5;"

# Check transaction count
sqlite3 deps/db-sync/data/graphs/<graph-id>/graph.db "SELECT COUNT(*) FROM transactions;"
```

**Common queries:**
```bash
# List all tables
sqlite3 deps/db-sync/data/graphs/<graph-id>/graph.db ".tables"

# Show schema
sqlite3 deps/db-sync/data/graphs/<graph-id>/graph.db ".schema"

# Recent transactions
sqlite3 deps/db-sync/data/graphs/<graph-id>/graph.db "SELECT * FROM transactions ORDER BY created_at DESC LIMIT 10;"
```

### Step 8: Monitor Server Logs and Debug

The server's log level determines what you see. By default, `start.sh` uses `debug` level.

**Log Levels (from most to least verbose):**
- `trace` - Everything including raw message payloads (VERY verbose)
- `debug` - Connection details, sync operations, WebSocket messages  
- `info` - Startup and important events (production default)
- `warn` - Warnings only
- `error` - Errors only

**Enable Debug Logging:**

If you're not seeing much output, make sure debug logging is enabled:

```bash
# Method 1: Edit start.sh (permanent)
nano deps/db-sync/start.sh
# Change: : "${DB_SYNC_LOG_LEVEL:=info}"
# To:     : "${DB_SYNC_LOG_LEVEL:=debug}"

# Method 2: Inline when starting (temporary)
cd deps/db-sync
DB_SYNC_LOG_LEVEL=debug ./start.sh

# Method 3: Export before starting (temporary)
export DB_SYNC_LOG_LEVEL=debug
cd deps/db-sync
./start.sh
```

**What You'll See with Debug Logging:**

**On Server Startup:**
```
Starting Logseq sync...
[DEBUG] Environment: NODE_ENV=undefined
[DEBUG] Configuration loaded:
[DEBUG] - Port: 8787
[DEBUG] - Data directory: /path/to/deps/db-sync/data
[DEBUG] - Storage driver: sqlite
[DEBUG] - Assets driver: filesystem
[DEBUG] Server initialized with SQLite storage
Logseq sync listening on port 8787
```

**On Client Connection:**
```
[DEBUG] WebSocket connection from 127.0.0.1:54321
[DEBUG] Connection ID: conn-abc123
[DEBUG] Waiting for authentication...
[DEBUG] Received auth token
[DEBUG] Token payload: {sub: "user-id-xxx", aud: "client-id", ...}
[DEBUG] User authenticated: user-id-xxx
[DEBUG] Client registered for graph: my-graph-id
```

**During Sync Operations:**
```
[DEBUG] Received message: {:op :sync/push-transaction :graph-id "..." :tx-id 123}
[DEBUG] Processing transaction for graph: my-graph-id
[DEBUG] Transaction contains 5 datoms
[DEBUG] Writing to SQLite: tx-id=123, size=2.4KB
[DEBUG] Transaction saved successfully
[DEBUG] Broadcasting to 2 connected clients
[DEBUG] Sending update to client: conn-abc123
[DEBUG] Update delivered successfully
```

**On Client Disconnect:**
```
[DEBUG] WebSocket closed: conn-abc123
[DEBUG] Reason: Normal closure
[DEBUG] Unregistering client from graph: my-graph-id
[DEBUG] Active connections: 1
```

**Enable Trace Logging (Maximum Detail):**

For deep debugging, use trace level:
```bash
DB_SYNC_LOG_LEVEL=trace ./start.sh
```

This shows:
- Raw WebSocket message payloads
- Full transaction data
- Detailed authentication flow
- All database queries

⚠️ **Warning**: Trace logging is VERY verbose and may contain sensitive data.

**Check Environment Variables Are Being Read:**

```bash
# Before starting server, verify:
echo $DB_SYNC_PORT          # Should show 8787
echo $DB_SYNC_LOG_LEVEL     # Should show debug

# Or check in Node.js:
node -e "console.log('Port:', process.env.DB_SYNC_PORT)"
```

**Tail Logs in Real-Time:**

If running server in background:
```bash
# If using systemd (for VPS deployment)
sudo journalctl -u logseq-sync -f

# If logging to file
tail -f /var/log/logseq-sync.log
```

---

### Step 9: Common Issues During Local Testing

#### Issue: "Sync option not showing in UI" or "Use Logseq Sync? checkbox missing"

**Symptoms:** When creating a new graph, you don't see the "Use Logseq Sync?" checkbox

**Root Cause:** Sync UI only appears for users in the RTC group or when `ENABLE_DB_SYNC_LOCAL=true`

**Solutions:**
1. **Verify local sync is enabled**:
   ```bash
   # Check compiled output
   grep "ENABLE_DB_SYNC_LOCAL" /Users/ashutosh/Documents/GITS/logseq/static/js/main.js
   # Should show: ENABLE_DB_SYNC_LOCAL=true
   ```

2. **Rebuild with the flag**:
   ```bash
   # Stop current build (Ctrl+C in Terminal 2)
   # Rebuild with flag
   yarn cross-env ENABLE_DB_SYNC_LOCAL=true yarn watch
   # Wait for "Build completed"
   ```

3. **Verify the rtc-group bypass** (already applied in this repo):
   - The `user.cljs` file has been updated to enable sync UI when `ENABLE_DB_SYNC_LOCAL=true`
   - File: [src/main/frontend/handler/user.cljs](src/main/frontend/handler/user.cljs)

4. **Must create a NEW DB graph**:
   - Only DB graphs support sync (not markdown/file graphs)
   - Click "Add a DB graph" when creating
   - If button says "Add a graph", look for "Add a DB graph" option instead

#### Issue: "Existing graph doesn't sync"

**Symptoms:** Opened an existing graph but sync doesn't work

**Root Cause:** Sync ONLY works with **DB graphs**, not markdown/file graphs

**How to identify your graph type:**
- **DB graph**: URL like `logseq_db_my-graph` (SQLite-based)
- **File graph**: Directory path like `/Users/you/Documents/my-graph` (Markdown files)

**Solution:**
- You **must create a NEW DB graph** with sync enabled
- Existing markdown graphs cannot be converted to DB graphs
- To migrate content:
  1. Export pages from old graph (Settings > Export)
  2. Import into new DB graph (Settings > Import)

#### Issue: "Port 8787 already in use"

**Symptoms:** Server fails to start with "EADDRINUSE" error

**Solutions:**
```bash
# Find what's using the port
lsof -i :8787

# Kill the process
kill -9 <PID>

# Or use a different port
DB_SYNC_PORT=8788 ./start.sh
# Then update config.cljs to match (requires rebuild):
# "ws://127.0.0.1:8788/sync/%s"
```

#### Issue: App connects but no sync happens

**Symptoms:** No error messages, but changes don't sync between instances

**Debug steps:**
1. **Check server logs** - Set `DB_SYNC_LOG_LEVEL=debug` and restart server
2. **Check app console**:
   - In Logseq: View > Toggle Developer Tools > Console
   - Look for WebSocket errors or sync errors
3. **Verify authentication**:
   ```
   # In app console, check:
   localStorage.getItem('auth-id-token')  # Should have a JWT token
   ```
4. **Check network tab**:
   - View > Toggle Developer Tools > Network tab
   - Filter by "WS" to see WebSocket connections
   - Should show connection to `ws://127.0.0.1:8787/sync/<graph-id>`

#### Issue: "WebSocket connection failed"

**Symptoms:** App shows sync errors or "Unable to connect to sync server"

**Solutions:**
1. **Verify server is running**: Check Terminal 1 - should show "listening on port 8787"
2. **Check port matches**: Server port (8787) must match [config.cljs](src/main/frontend/config.cljs)
3. **Firewall**: Make sure localhost traffic is allowed
4. **Browser/Electron security**: Try disabling any security extensions
5. **Check URL format**: Should be `ws://` not `wss://` for local testing

#### Issue: Authentication errors

**Symptoms:** "Unauthorized" or "Invalid token" errors

**Solutions:**
1. **Using test Cognito**: The default `start.sh` uses a test Cognito pool that should work out of the box
2. **Token expired**: Tokens expire after 1 hour. Restart the app to get a new token.
3. **Wrong Cognito credentials**: Verify COGNITO_CLIENT_ID and COGNITO_ISSUER match between:
   - Server environment variables
   - Client config (if customized)
4. **Test without auth** (for debugging only):
   ```bash
   # Start server without authentication
   DISABLE_AUTH=true DB_SYNC_LOG_LEVEL=debug ./start.sh
   
   # Note: This is INSECURE - only for local debugging
   ```

#### Issue: Build fails with "command not found"

**Symptoms:** `yarn watch` fails with "clojure: command not found" or "java: command not found"

**Solutions:**
```bash
# Install Clojure
brew install clojure/tools/clojure  # macOS
# or follow: https://clojure.org/guides/install_clojure

# Install Java (OpenJDK 20+)
brew install openjdk@20  # macOS

# Verify installations
clojure --version  # Should be 1.12.4+
java --version     # Should be 20+
```

#### Issue: Changes sync slowly or with delay

**Symptoms:** Changes take 5-10+ seconds to appear on other clients

**Solutions:**
1. **Check CPU usage**: Shadow-cljs watch mode can be CPU intensive during development
2. **Network tab**: Check for reconnection issues in browser dev tools
3. **Server logs**: Look for backpressure or queue warnings
4. **Database locks**: Check if SQLite is having lock contention
   ```bash
   sqlite3 deps/db-sync/data/graphs/<graph-id>/graph.db "PRAGMA integrity_check;"
   ```

#### Issue: "Cannot find module 'better-sqlite3'"

**Symptoms:** Server crashes on startup with module not found error

**Solutions:**
```bash
cd deps/db-sync

# Clean and reinstall
rm -rf node_modules
yarn install

# Rebuild native modules
yarn rebuild better-sqlite3

# Try starting again
./start.sh
```

#### Issue: App shows "white screen" after launch

**Symptoms:** Electron window opens but shows blank white page

**Solutions:**
1. **Wait 30-60 seconds** - first load can be slow
2. **Check Terminal 2**: Make sure build completed without errors
3. **Check developer console**: View > Toggle Developer Tools > Console for errors
4. **Clear app data**:
   ```bash
   # macOS
   rm -rf ~/Library/Application\ Support/Logseq
   
   # Linux  
   rm -rf ~/.config/Logseq
   
   # Windows
   # Delete: %APPDATA%\Logseq
   ```
5. **Rebuild from scratch**:
   ```bash
   yarn clean
   yarn install
   ENABLE_DB_SYNC_LOCAL=true yarn watch
   ```

#### Issue: Multiple graph instances cause conflicts

**Symptoms:** Sync works but data gets corrupted or duplicated

**Solutions:**
1. **Don't open the same graph in standard and dev Logseq simultaneously**
2. **Use different graphs for production vs development** testing
3. **Check for duplicate processes**:
   ```bash
   ps aux | grep -i logseq
   # Kill any unexpected processes
   ```

#### Issue: Server crashes with "SQLITE_BUSY" error

**Symptoms:** Server logs show database locked errors

**Solutions:**
1. **Close all Logseq instances** connecting to that graph
2. **Wait 30 seconds** for locks to clear
3. **Check for zombie connections**:
   ```bash
   lsof deps/db-sync/data/graphs/*/graph.db
   ```
4. **Restart server** with clean slate:
   ```bash
   pkill node  # Kill all node processes
   ./start.sh
   ```

---

### Step 10: Stopping the Local Development Environment

When you're done testing:

**1. Close Logseq instances:**
- Close all Electron app windows
- Or press `Ctrl+C` in Terminal 3 (and Terminal 4 if you opened a second instance)

**2. Stop the watch build (Terminal 2):**
```bash
# Press Ctrl+C in Terminal 2
# This stops the ClojureScript watch compilation
```

**3. Stop the sync server (Terminal 1):**
```bash
# Press Ctrl+C in Terminal 1
# This gracefully shuts down the server and closes database connections
```

**Clean shutdown order:**
1. Close app(s) first (prevents "connection lost" errors)
2. Stop watch build (no longer needed)
3. Stop server last (allows final sync operations to complete)

**If processes don't stop:**
```bash
# Force kill all Java processes (ClojureScript builds)
pkill java

# Force kill all Node processes (sync server)
pkill node

# Force kill Electron
pkill Electron
```

**Verify everything is stopped:**
```bash
# Check no processes are using port 8787
lsof -i :8787  # Should return nothing

# Check for lingering Logseq processes
ps aux | grep -i logseq

# Check for lingering Node processes
ps aux | grep -i node
```

---

## Part 2: VPS Deployment

### Step 1: Prepare Your VPS

**Requirements:**
- Ubuntu 22.04+ (or similar Linux distribution)
- 1GB+ RAM
- Node.js 22+
- Open port: 8787 (or your chosen port)

**SSH into your VPS:**
```bash
ssh your-user@your-vps-ip
```

### Step 2: Install Prerequisites on VPS

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Node.js 22
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Install build essentials (required for better-sqlite3)
sudo apt install -y build-essential python3

# Install yarn
npm install -g yarn

# Verify
node --version
yarn --version
```

### Step 3: Deploy Node.js Adapter to VPS

**Option A: Copy built files**
```bash
# On your local machine
cd /Users/ashutosh/Documents/GITS/logseq/deps/db-sync
tar czf node-adapter.tar.gz dist/node-adapter package.json

# Copy to VPS
scp node-adapter.tar.gz your-user@your-vps-ip:~/
```

**Option B: Clone and build on VPS**
```bash
# On VPS
git clone https://github.com/logseq/logseq.git
cd logseq/deps/db-sync
yarn install
yarn build:node-adapter
```

### Step 4: Create Systemd Environment File

**On VPS, create an environment file that systemd will read:**

```bash
# On VPS
cd ~/logseq/deps/db-sync

# Create environment file for systemd
cat > systemd.env << 'EOF'
# Server Configuration
DB_SYNC_PORT=8787
DB_SYNC_DATA_DIR=/var/lib/logseq-sync/data
DB_SYNC_LOG_LEVEL=info

# Authentication (Test Cognito pool - change for production)
COGNITO_ISSUER=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_dtagLnju8
COGNITO_CLIENT_ID=69cs1lgme7p8kbgld8n5kseii6
COGNITO_JWKS_URL=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_dtagLnju8/.well-known/jwks.json
EOF

# Create data directory
sudo mkdir -p /var/lib/logseq-sync/data
sudo chown $USER:$USER /var/lib/logseq-sync/data
```

**Note:** Systemd will load this file via `EnvironmentFile=` in the service file (Step 5).

### Step 5: Set Up Systemd Service

Create a systemd service for automatic startup:

```bash
sudo nano /etc/systemd/system/logseq-sync.service
```

Add this content:
```ini
[Unit]
Description=Logseq DB Sync Node Adapter
After=network.target

[Service]
Type=simple
User=your-user
WorkingDirectory=/home/your-user/logseq/deps/db-sync
EnvironmentFile=/home/your-user/logseq/deps/db-sync/systemd.env
ExecStart=/usr/bin/node dist/node-adapter/server.js
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Create data directory and enable service:**
```bash
# Create data directory
sudo mkdir -p /var/lib/logseq-sync/data
sudo chown $USER:$USER /var/lib/logseq-sync/data

# Enable and start the service
sudo systemctl daemon-reload
sudo systemctl enable logseq-sync
sudo systemctl start logseq-sync
sudo systemctl status logseq-sync
```

### Step 6: Configure Firewall

```bash
# Allow port 57414
```bash
# Allow port 8787
sudo ufw allow 8787/tcp
sudo ufw status
```

### Step 7: Set Up Reverse Proxy with SSL (Recommended)

Install nginx:
```bash
sudo apt install -y nginx certbot python3-certbot-nginx
```

Create nginx configuration:
```bash
sudo nano /etc/nginx/sites-available/logseq-sync
```

Add this content:
```nginx
# HTTP server (will redirect to HTTPS)
server {
    listen 80;
    server_name sync.yourdomain.com;  # Replace with your domain
    
    location / {
        return 301 https://$server_name$request_uri;
    }
}

# HTTPS server
server {
    listen 443 ssl http2;
    server_name sync.yourdomain.com;  # Replace with your domain
    
    # SSL certificates (certbot will add these)
    # ssl_certificate /etc/letsencrypt/live/sync.yourdomain.com/fullchain.pem;
    # ssl_certificate_key /etc/letsencrypt/live/sync.yourdomain.com/privkey.pem;
    
    # WebSocket support
    location /sync/ {
        proxy_pass http://127.0.0.1:8787;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # WebSocket timeout
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
    }
    
    # HTTP API
    location / {
        proxy_pass http://127.0.0.1:8787;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable the site:
```bash
sudo ln -s /etc/nginx/sites-available/logseq-sync /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

Get SSL certificate:
```bash
sudo certbot --nginx -d sync.yourdomain.com
```

### Step 8: Configure Logseq Client for VPS

You have two options:

**Option A: Modify config.cljs permanently**

Edit `src/main/frontend/config.cljs`:
```clojure
(defonce db-sync-ws-url
  (if db-sync-local?
    "wss://sync.yourdomain.com/sync/%s"  ; Your VPS domain
    "wss://logseq-sync-prod.logseq.workers.dev/sync/%s"))

(defonce db-sync-http-base
  (if db-sync-local?
    "https://sync.yourdomain.com"  ; Your VPS domain
    "https://logseq-sync-prod.logseq.workers.dev"))
```

Then build:
```bash
yarn cross-env ENABLE_DB_SYNC_LOCAL=true yarn watch
```

**Option B: Build with IP address for testing**

For testing without domain, modify temporarily:
```clojure
(defonce db-sync-ws-url
  (if db-sync-local?
    "ws://YOUR_VPS_IP:8787/sync/%s"  ; Your VPS IP
    "wss://logseq-sync-prod.logseq.workers.dev/sync/%s"))

(defonce db-sync-http-base
  (if db-sync-local?
    "http://YOUR_VPS_IP:8787"  ; Your VPS IP
    "https://logseq-sync-prod.logseq.workers.dev"))
```

---

## Part 3: Authentication Setup

By default, the Node.js adapter uses Logseq's test Cognito pool, which means **anyone can authenticate**. Here's how to secure it:

**📘 For detailed step-by-step instructions on using your own AWS Cognito User Pool,** see the complete guide: [CUSTOM_COGNITO_SETUP.md](CUSTOM_COGNITO_SETUP.md)

The guide below provides quick reference for authentication options. For a comprehensive walkthrough including AWS Cognito setup, server configuration, optional client configuration, and troubleshooting, use the dedicated guide above.

### Option 1: Disable Cognito Authentication (Simple, IP-Restricted)

**Best for:** Private VPS with firewall/VPN access

Edit the systemd environment file:
```bash
# Add to systemd.env
DISABLE_AUTH=true

# Lock down to specific IPs in firewall instead
```

Configure firewall to only allow your IP addresses:
```bash
# Example: Only allow home and office IPs
sudo ufw delete allow 8787/tcp
sudo ufw allow from 203.0.113.1 to any port 8787  # Your home IP
sudo ufw allow from 203.0.113.2 to any port 8787  # Your office IP
```

Or use VPN (recommended):
```bash
# Install WireGuard
sudo apt install wireguard

# Configure WireGuard (see WireGuard docs)
# Then only allow connections from VPN network
sudo ufw allow from 10.0.0.0/24 to any port 8787
```

### Option 2: Create Your Own AWS Cognito Pool (Recommended)

**Best for:** Multiple users, proper authentication

#### Step 1: Create AWS Cognito User Pool

1. Go to [AWS Console](https://console.aws.amazon.com/) → Cognito
2. Click "Create user pool"
3. **Sign-in options:** Email
4. **Password requirements:** Cognito defaults
5. **MFA:** Optional (recommended)
6. **Email provider:** Choose (SES or Cognito)
7. **User pool name:** `logseq-sync-prod`
8. **App client name:** `logseq-sync-client`
9. **App client settings:**
   - Enable "ALLOW_USER_PASSWORD_AUTH"
   - Enable "ALLOW_REFRESH_TOKEN_AUTH"
10. Click "Create user pool"

#### Step 2: Note Your Credentials

After creation, you need these values:
- **User Pool ID**: `us-east-1_xxxxxxxxx` 
- **Client ID**: `xxxxxxxxxxxxxxxxxxxxxxxxxx`
- **Region**: `us-east-1` (or your chosen region)

**Then construct the environment variables:**
- `COGNITO_ISSUER` = `https://cognito-idp.{REGION}.amazonaws.com/{USER_POOL_ID}`
- `COGNITO_CLIENT_ID` = Your Client ID from above
- `COGNITO_JWKS_URL` = `https://cognito-idp.{REGION}.amazonaws.com/{USER_POOL_ID}/.well-known/jwks.json`

**Example:**
- User Pool ID: `us-east-1_ABC123xyz`
- Region: `us-east-1`
- Client ID: `7a8b9c0d1e2f3g4h5i6j7k8l9m`

Becomes:
- `COGNITO_ISSUER=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz`
- `COGNITO_CLIENT_ID=7a8b9c0d1e2f3g4h5i6j7k8l9m`
- `COGNITO_JWKS_URL=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz/.well-known/jwks.json`

#### Step 3: Create Users

In your Cognito User Pool:
1. Go to "Users" tab
2. Click "Create user"
3. Enter email and temporary password
4. User will need to change password on first login

#### Step 4: Update VPS Configuration

Now that you have your Cognito credentials from AWS, update your VPS server configuration:

```bash
# Navigate to db-sync directory
cd ~/logseq/deps/db-sync

# Edit the environment file
nano systemd.env
```

**Replace** the existing Cognito test pool values with your custom values:

```bash
# Before (test pool defaults):
COGNITO_ISSUER=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_123456789
COGNITO_CLIENT_ID=abcdefghijklmnopqrstuvwxyz
COGNITO_JWKS_URL=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_123456789/.well-known/jwks.json

# After (your custom pool):
COGNITO_ISSUER=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_xxxxxxxxx
COGNITO_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxxxx
COGNITO_JWKS_URL=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_xxxxxxxxx/.well-known/jwks.json
```

Save the file (Ctrl+O, Enter, Ctrl+X in nano).

**Restart the service** to apply changes:
```bash
sudo systemctl restart logseq-sync

# Check service status
sudo systemctl status logseq-sync

# Verify new configuration loaded (optional)
sudo systemctl show logseq-sync --property=Environment
```

Your server now validates tokens from your custom Cognito pool!

#### Step 5: Configure Logseq Client (Optional)

**Note:** If you're using the **test Cognito pool** or users log in via Logseq's main auth system, you **don't need to change the client**. The server will validate tokens from any Cognito pool you configure.

**Only needed if:** You want users to authenticate directly with your custom Cognito pool in the Logseq login UI.

Edit `src/main/frontend/config.cljs`:

```clojure
;; Add new section for custom Cognito
(if ENABLE-DB-SYNC-LOCAL
  ;; Your custom Cognito pool
  (do (def COGNITO-IDP "https://cognito-idp.us-east-1.amazonaws.com/")
      (def COGNITO-CLIENT-ID "xxxxxxxxxxxxxxxxxxxxxxxxxx")  ; Your client ID
      (def REGION "us-east-1")  ; Your region
      (def USER-POOL-ID "us-east-1_xxxxxxxxx"))  ; Your pool ID
  
  ;; Production (existing code)
  (if ENABLE-FILE-SYNC-PRODUCTION
    (do (def COGNITO-IDP "https://cognito-idp.us-east-1.amazonaws.com/")
        ;; ... existing production config
        )))
```

**Most users won't need this** - the server config is usually sufficient.

### Option 3: API Key Authentication (Custom Implementation)

**Best for:** Simple single-user or programmatic access

Edit `deps/db-sync/src/node-adapter/server.ts`:

```typescript
// Add API key middleware
const API_KEY = process.env.API_KEY || 'generate-random-key-here';

function authenticateApiKey(req: Request, res: Response, next: Function) {
  const apiKey = req.headers['x-api-key'];
  if (apiKey === API_KEY) {
    next();
  } else {
    res.status(401).json({ error: 'Unauthorized' });
  }
}

// Apply to all routes
app.use(authenticateApiKey);
```

Add to systemd environment file:
```bash
echo "API_KEY=your-super-secret-random-api-key-here" >> systemd.env
```

**Then modify Logseq client to send the API key** (requires more code changes).

---

## Testing Your Setup

### Test Authentication
```bash
# Test server is running
curl http://your-vps-ip:8787/

# Test WebSocket (requires wscat)
npm install -g wscat
wscat -c ws://your-vps-ip:8787/sync/test-graph
```

### Monitor Logs
```bash
# On VPS
sudo journalctl -u logseq-sync -f

# Check data directory
ls -la /var/lib/logseq-sync/data/
```

### Test Sync
1. Open Logseq on Device 1
2. Sign in (if using Cognito)
3. Create/modify content
4. Open Logseq on Device 2 with same account
5. Verify changes appear automatically

---

## Troubleshooting

### Server Won't Start
```bash
# Check logs
sudo journalctl -u logseq-sync -n 50

# Check if port is in use
sudo lsof -i :8787

# Test manually
cd ~/logseq/deps/db-sync
node dist/node-adapter/server.js
```

### Connection Refused
```bash
# Check firewall
sudo ufw status

# Check server is listening
sudo netstat -tlnp | grep 8787

# Test from local machine
telnet your-vps-ip 8787
```

### WebSocket Connection Fails
- Check nginx WebSocket configuration
- Verify `Upgrade` headers are being proxied
- Check SSL certificate is valid
- Look at browser console for errors

### Authentication Errors
```bash
# Check Cognito credentials loaded by systemd
sudo systemctl show logseq-sync -p Environment

# Or check the file directly
cat ~/logseq/deps/db-sync/systemd.env | grep COGNITO
```

### Data Not Syncing
- Check server logs for errors
- Verify client is using correct URLs
- Check network connectivity
- Verify authentication succeeded (check auth token in browser storage)

### Build Errors
```bash
# Clean and rebuild
yarn clean
yarn install
yarn build:node-adapter

# For client
ENABLE_DB_SYNC_LOCAL=true yarn watch
```

---

## Security Best Practices

1. **Always use HTTPS/WSS in production** (nginx + Let's Encrypt)
2. **Use strong authentication** (Cognito or VPN)
3. **Regular backups** of `/var/lib/logseq-sync/data/`
4. **Monitor access logs** for suspicious activity
5. **Keep Node.js and dependencies updated**
6. **Use firewall rules** to restrict access
7. **Consider rate limiting** to prevent abuse

---

## Backup Your Data

```bash
# On VPS - automated backup script
cat > /usr/local/bin/backup-logseq-sync.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/var/backups/logseq-sync"
DATA_DIR="/var/lib/logseq-sync/data"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"
tar czf "$BACKUP_DIR/sync-data-$DATE.tar.gz" "$DATA_DIR"

# Keep only last 7 days
find "$BACKUP_DIR" -name "sync-data-*.tar.gz" -mtime +7 -delete
EOF

chmod +x /usr/local/bin/backup-logseq-sync.sh

# Add to crontab (daily at 2 AM)
(crontab -l 2>/dev/null; echo "0 2 * * * /usr/local/bin/backup-logseq-sync.sh") | crontab -
```

---

## Next Steps

1. **Test locally** following Part 1
2. **Deploy to VPS** following Part 2
3. **Set up authentication** following Part 3
4. **Configure backups** using the script above
5. **Monitor and maintain** your server

## Questions?

- Check the logs first: `sudo journalctl -u logseq-sync -f`
- Review [deps/db-sync/README.md](deps/db-sync/README.md)
- Check Logseq Discord #self-hosting channel

