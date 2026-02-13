# Using Your Own AWS Cognito Setup

This guide covers all changes required to use your own AWS Cognito User Pool instead of the default test pool.

## Overview

Logseq sync has **two authentication points**:

1. **Server-side** (sync server): Validates JWT tokens from sync requests
2. **Client-side** (Logseq app): Authenticates users and obtains JWT tokens

**Most users only need to configure the server** - users can still log in with Logseq's default auth, and your server will validate their tokens.

**Only configure the client** if you want a completely isolated auth system where users log in directly with your custom Cognito pool.

---

## Part 1: Create AWS Cognito User Pool

### Step 1: Create the User Pool

1. Go to [AWS Console](https://console.aws.amazon.com/) → Cognito
2. Click **"Create user pool"**
3. Configure authentication:
   - **Sign-in options:** Email
   - **Password requirements:** Cognito defaults (or customize)
   - **MFA:** Optional (recommended for production)
   - **Email provider:** Choose (SES recommended for production, Cognito for testing)

4. Configure app integration:
   - **User pool name:** `logseq-sync-prod` (or your choice)
   - **Domain:** Not required for password auth
   
5. Create app client:
   - **App client name:** `logseq-sync-client`
   - **Authentication flows:** 
     - ✅ Enable `ALLOW_USER_PASSWORD_AUTH`
     - ✅ Enable `ALLOW_REFRESH_TOKEN_AUTH`
   - **Token expiration:** Default (or customize)

6. Click **"Create user pool"**

### Step 2: Note Your Credentials

After creation, go to your User Pool in AWS Console and note:

- **User Pool ID**: Found under "User pool overview" → looks like `us-east-1_xxxxxxxxx`
- **Region**: The AWS region you created the pool in (e.g., `us-east-1`, `us-west-2`)
- **App Client ID**: Go to "App integration" tab → "App clients" → looks like `xxxxxxxxxxxxxxxxxxxxxxxxxx`

### Step 3: Construct Environment Variables

Using the values from Step 2, construct these three environment variables:

```bash
# Pattern:
COGNITO_ISSUER=https://cognito-idp.{REGION}.amazonaws.com/{USER_POOL_ID}
COGNITO_CLIENT_ID={APP_CLIENT_ID}
COGNITO_JWKS_URL=https://cognito-idp.{REGION}.amazonaws.com/{USER_POOL_ID}/.well-known/jwks.json

# Example (replace with your actual values):
COGNITO_ISSUER=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz
COGNITO_CLIENT_ID=7a8b9c0d1e2f3g4h5i6j7k8l9m
COGNITO_JWKS_URL=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz/.well-known/jwks.json
```

**Save these** - you'll use them in Parts 2 and 3.

### Step 4: Create Users

In your Cognito User Pool:

1. Go to **"Users"** tab
2. Click **"Create user"**
3. Enter:
   - Email address
   - Temporary password (user must change on first login)
   - Optionally mark email as verified
4. Click **"Create user"**

Users will need to change their password on first login through Logseq.

### Step 5: Configure OAuth Domain (Required for Complete Isolation)

**⚠️ Required if you want complete isolation from Logseq's authentication.**

To enable token refresh and authentication flows with your own Cognito pool:

1. **Go to AWS Console** → Cognito → Your User Pool
2. Click **"App integration"** tab
3. Scroll down to **"Domain"** section
4. Click **"Actions"** → **"Create Cognito domain"**

**Creating the domain:**
- **Domain prefix**: Choose a unique name (e.g., `logseq-sync-prod`, `my-logseq-2024`)
- AWS validates availability globally
- Format: `{your-prefix}.auth.{region}.amazoncognito.com`
- Click **"Create Cognito domain"**

**Example:**
```
Domain prefix: logseq-sync-prod
Region: us-east-1
Full OAuth domain: logseq-sync-prod.auth.us-east-1.amazoncognito.com
```

5. **Note the OAuth domain** - you'll need it for Part 3 client configuration:
   - `OAUTH-DOMAIN = {your-prefix}.auth.{region}.amazoncognito.com`

**Advanced: Custom Domain**
- If you own a domain, you can use it (e.g., `auth.yourdomain.com`)
- Requires ACM certificate in the same region
- See [AWS Cognito custom domain docs](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-add-custom-domain.html)

---

## Part 2: Configure the Sync Server

Choose your deployment type:

### Option A: VPS Deployment (Production)

If you've set up the server with systemd (see `SELF_HOSTED_SYNC_SETUP.md` Part 2):

```bash
# Navigate to db-sync directory
cd ~/logseq/deps/db-sync

# Edit the environment file
nano systemd.env
```

**Replace** the three Cognito variables with your values from Part 1:

```bash
# Your custom pool (replace with actual values from Part 1 Step 3):
COGNITO_ISSUER=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz
COGNITO_CLIENT_ID=7a8b9c0d1e2f3g4h5i6j7k8l9m
COGNITO_JWKS_URL=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz/.well-known/jwks.json

# Keep these unchanged:
DB_SYNC_PORT=8787
DB_SYNC_DATA_DIR=/var/lib/logseq-sync/data
DB_SYNC_LOG_LEVEL=info
```

Save the file (Ctrl+O, Enter, Ctrl+X in nano).

**Restart the service:**
```bash
sudo systemctl restart logseq-sync

# Verify service is running
sudo systemctl status logseq-sync

# Check logs if needed
sudo journalctl -u logseq-sync -n 50 -f
```

**Done!** Your VPS server now uses your custom Cognito pool.

### Option B: Local Development

If you're running the server locally for testing:

**Method 1: Export before running (recommended)**
```bash
cd deps/db-sync

# Export your custom Cognito values
export COGNITO_ISSUER="https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz"
export COGNITO_CLIENT_ID="7a8b9c0d1e2f3g4h5i6j7k8l9m"
export COGNITO_JWKS_URL="https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz/.well-known/jwks.json"

# Run the server
./start.sh
```

**Method 2: Modify start.sh (persistent)**
```bash
cd deps/db-sync
nano start.sh
```

Replace lines 8-10 with your values:
```bash
# Before (default test pool):
: "${COGNITO_ISSUER:=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_dtagLnju8}"
: "${COGNITO_CLIENT_ID:=69cs1lgme7p8kbgld8n5kseii6}"
: "${COGNITO_JWKS_URL:=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_dtagLnju8/.well-known/jwks.json}"

# After (your custom pool):
: "${COGNITO_ISSUER:=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz}"
: "${COGNITO_CLIENT_ID:=7a8b9c0d1e2f3g4h5i6j7k8l9m}"
: "${COGNITO_JWKS_URL:=https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz/.well-known/jwks.json}"
```

Save and run:
```bash
./start.sh
```

---

## Part 3: Configure the Logseq Client (Optional)

**⚠️ IMPORTANT:** This section is **OPTIONAL** and only needed if:
- You want users to log in directly with your custom Cognito pool
- You're running a completely isolated system
- You don't want users authenticated through Logseq's main auth system

**For most self-hosted users**: Skip this section. Your server configuration from Part 2 is sufficient - users can log in with Logseq's default authentication, and your server will validate their tokens.

---

### When to Configure the Client

Configure the client if you need:
- ✅ Complete isolation from Logseq's auth servers
- ✅ Custom user management in your own Cognito pool
- ✅ Different authentication requirements (MFA, password policies, etc.)

Skip if:
- ❌ You're OK with users logging in through Logseq's standard auth
- ❌ You only want to run your own sync server, not manage authentication
- ❌ You're testing locally and don't mind the test pool

---

### Client Configuration Steps

#### Step 1: Edit Client Configuration

Edit `src/main/frontend/config.cljs`:

```clojure
(goog-define ENABLE-DB-SYNC-LOCAL false)
(defonce db-sync-local? ENABLE-DB-SYNC-LOCAL)

;; Add these additional configuration variables
(goog-define ENABLE-CUSTOM-COGNITO false)
(defonce custom-cognito? ENABLE-CUSTOM-COGNITO)

;; Update the Cognito configuration section (around line 27-48)
;; BEFORE: Production vs Test configuration only

;; AFTER: Add custom Cognito option
(if custom-cognito?
  ;; Your custom Cognito pool (COMPLETE ISOLATION)
  (do (def LOGIN-URL
        "https://logseq-sync-prod.auth.us-east-1.amazoncognito.com/login?client_id=7a8b9c0d1e2f3g4h5i6j7k8l9m&response_type=code&scope=email+openid+phone&redirect_uri=logseq%3A%2F%2Fauth-callback")
      (def API-DOMAIN "api.logseq.com")  ; Optional: Leave as is unless you hosted your own API server
      (def COGNITO-IDP "https://cognito-idp.us-east-1.amazonaws.com/")
      (def COGNITO-CLIENT-ID "7a8b9c0d1e2f3g4h5i6j7k8l9m")  ; Your App Client ID from Part 1 Step 2
      (def REGION "us-east-1")  ; Your AWS region from Part 1 Step 2
      (def USER-POOL-ID "us-east-1_ABC123xyz")  ; Your User Pool ID from Part 1 Step 2
      (def IDENTITY-POOL-ID "us-east-1:xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx")  ; Optional: only if you need AWS SDK access
      (def OAUTH-DOMAIN "logseq-sync-prod.auth.us-east-1.amazoncognito.com")  ; Your OAuth domain from Part 1 Step 5
      (def PUBLISH-API-BASE "https://logseq.io"))  ; Keep default unless you self-host publishing too
  
  ;; Existing production/test configuration (keep unchanged)
  (if ENABLE-FILE-SYNC-PRODUCTION
    (do (def LOGIN-URL
          "https://logseq-prod.auth.us-east-1.amazoncognito.com/login?client_id=3c7np6bjtb4r1k1bi9i049ops5&response_type=code&scope=email+openid+phone&redirect_uri=logseq%3A%2F%2Fauth-callback")
        ;; ... rest of existing config ...
        )))
```

**Replace these values with your actual credentials:**

| Variable | Replace With | Where to Get It |
|----------|-------------|-----------------|
| `LOGIN-URL` OAuth domain | `logseq-sync-prod.auth.us-east-1.amazoncognito.com` | Part 1 Step 5 (OAuth domain you created) |
| `LOGIN-URL` client_id | `7a8b9c0d1e2f3g4h5i6j7k8l9m` | Part 1 Step 2 (App Client ID) |
| `COGNITO-IDP` | Keep `https://cognito-idp.{REGION}.amazonaws.com/` | Only change `{REGION}` to your region |
| `COGNITO-CLIENT-ID` | `7a8b9c0d1e2f3g4h5i6j7k8l9m` | Part 1 Step 2 (App Client ID) |
| `REGION` | `us-east-1` | Part 1 Step 2 (Your AWS region) |
| `USER-POOL-ID` | `us-east-1_ABC123xyz` | Part 1 Step 2 (User Pool ID) |
| `OAUTH-DOMAIN` | `logseq-sync-prod.auth.us-east-1.amazoncognito.com` | Part 1 Step 5 (OAuth domain you created) |

**Example with real values:**
```clojure
;; Example: us-east-1 region, custom Cognito pool
(if custom-cognito?
  (do (def LOGIN-URL
        "https://logseq-sync-prod.auth.us-east-1.amazoncognito.com/login?client_id=7a8b9c0d1e2f3g4h5i6j7k8l9m&response_type=code&scope=email+openid+phone&redirect_uri=logseq%3A%2F%2Fauth-callback")
      (def API-DOMAIN "api.logseq.com")
      (def COGNITO-IDP "https://cognito-idp.us-east-1.amazonaws.com/")
      (def COGNITO-CLIENT-ID "7a8b9c0d1e2f3g4h5i6j7k8l9m")
      (def REGION "us-east-1")
      (def USER-POOL-ID "us-east-1_ABC123xyz")
      (def IDENTITY-POOL-ID "")  ; Leave empty unless you need AWS SDK access
      (def OAUTH-DOMAIN "logseq-sync-prod.auth.us-east-1.amazoncognito.com")
      (def PUBLISH-API-BASE "https://logseq.io"))
  ...)
```

#### Step 2: Update Sync Server URL (if needed)

If your sync server is on a custom domain (not localhost):

```clojure
;; Around line 56-64
(defonce db-sync-ws-url
  (if custom-cognito?
    "wss://your-sync-server.com/sync/%s"  ; Your production URL
    (if db-sync-local?
      "ws://127.0.0.1:8787/sync/%s"
      "wss://logseq-sync-prod.logseq.workers.dev/sync/%s")))

(defonce db-sync-http-base
  (if custom-cognito?
    "https://your-sync-server.com"  ; Your production URL
    (if db-sync-local?
      "http://127.0.0.1:8787"
      "https://logseq-sync-prod.logseq.workers.dev")))
```

#### Step 3: Rebuild the Client

From the repository root:

```bash
# Install dependencies if not already done
yarn install

# Build with custom Cognito flag
yarn watch  # For development with hot-reload
# OR
yarn release  # For production build

# When running yarn watch or building, enable the flag:
ENABLE_CUSTOM_COGNITO=true yarn watch
# OR for release:
ENABLE_CUSTOM_COGNITO=true yarn release
```

**For shadow-cljs direct build:**
```bash
npx shadow-cljs compile app --config-merge '{:closure-defines {frontend.config/ENABLE-CUSTOM-COGNITO true}}'
```

#### Step 4: Test Authentication

1. Launch the rebuilt Logseq client
2. Try logging in with a user from your Cognito pool
3. Check browser console for authentication flow
4. Verify sync connection to your server

**Debug authentication issues:**
```bash
# Server logs (VPS):
sudo journalctl -u logseq-sync -f

# Server logs (local):
# Check terminal where start.sh is running

# Client logs:
# Open browser DevTools → Console
# Look for "auth" or "cognito" related messages
```

---

## Part 4: Testing Your Setup

### Test Server Authentication

```bash
# Get a test JWT token from your Cognito pool first
# (Use AWS CLI or authenticate through Logseq client)

# Test sync server authentication
curl -H "Authorization: Bearer YOUR_JWT_TOKEN" \
     http://localhost:8787/health

# Should return 200 OK if authentication works
```

### Test Full Sync Flow

1. **Create a user** in your Cognito pool (Part 1, Step 4)
2. **Start the sync server** with your Cognito config (Part 2)
3. **Launch Logseq client** (with custom config if you did Part 3)
4. **Log in** with your Cognito user credentials
5. **Create a DB graph** (File → New graph → Database graph)
6. **Enable sync** (Three-dot menu → Use Logseq Sync)
7. **Check server logs** for sync activity

**Expected server log output:**
```
[INFO] Client connected: user@example.com
[DEBUG] Validating token from issuer: https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz
[DEBUG] Token validated successfully
[INFO] Sync session started for graph: my-graph
```

---

## Troubleshooting

### Server Issues

**Error: "Invalid token"**
```bash
# Check server configuration
sudo systemctl show logseq-sync --property=Environment | grep COGNITO

# Verify values match your Cognito pool
# Should show your COGNITO_ISSUER, COGNITO_CLIENT_ID, COGNITO_JWKS_URL
```

**Error: "Unable to fetch JWKS"**
```bash
# Test JWKS URL directly
curl https://cognito-idp.us-east-1.amazonaws.com/us-east-1_ABC123xyz/.well-known/jwks.json

# Should return JSON with keys
# Check firewall isn't blocking outbound HTTPS
```

### Client Issues (if you configured Part 3)

**Error: "User pool does not exist"**
- Verify `USER-POOL-ID` in `config.cljs` matches AWS console exactly
- Check region is correct
- Ensure pool wasn't deleted

**Error: "Invalid client id"**
- Verify `COGNITO-CLIENT-ID` matches App Client ID from AWS console
- Check you're using the right app client

**Login page doesn't appear**
- Rebuild client with `ENABLE_CUSTOM_COGNITO=true`
- Check browser console for errors
- Verify `LOGIN-URL` is correct

### Network Issues

**Can't reach sync server**
```bash
# Test server is accessible
curl http://your-server:8787/health

# Check firewall rules (VPS)
sudo ufw status

# Verify port 8787 is open
sudo netstat -tlnp | grep 8787
```

### Authentication Flow Issues

**Users can't log in**
1. Verify user exists in Cognito pool (AWS Console → Users tab)
2. Check user status is "CONFIRMED"
3. Verify email is confirmed (or disable email verification requirement)
4. Check authentication flows are enabled on app client:
   - `ALLOW_USER_PASSWORD_AUTH`
   - `ALLOW_REFRESH_TOKEN_AUTH`

---

## Security Best Practices

### Production Deployment

1. **Use MFA**: Enable Multi-Factor Authentication in Cognito
2. **Strong passwords**: Configure password policies in Cognito
3. **SSL/TLS**: Use HTTPS for sync server (see nginx reverse proxy setup)
4. **Firewall**: Restrict access to sync server port
5. **Token expiration**: Configure appropriate token lifetimes in Cognito
6. **Monitoring**: Set up CloudWatch alarms for Cognito events

### Token Management

- **Access token lifetime**: 60 minutes (default, adjust as needed)
- **Refresh token lifetime**: 30 days (default, adjust as needed)
- **ID token lifetime**: 60 minutes (default, adjust as needed)

Configure in AWS Console → Your User Pool → App integration → App clients → Edit token expiration.

---

## Summary Checklist

- [ ] **Part 1**: Created AWS Cognito User Pool
- [ ] **Part 1**: Noted User Pool ID, Region, App Client ID
- [ ] **Part 1**: Constructed environment variables (COGNITO_ISSUER, COGNITO_CLIENT_ID, COGNITO_JWKS_URL)
- [ ] **Part 1**: Created test user(s)
- [ ] **Part 2**: Updated sync server configuration (VPS: systemd.env, Local: start.sh or exports)
- [ ] **Part 2**: Restarted server and verified it's running
- [ ] **Part 3** (Optional): Modified `config.cljs` with custom Cognito values
- [ ] **Part 3** (Optional): Rebuilt Logseq client with `ENABLE_CUSTOM_COGNITO=true`
- [ ] **Part 4**: Tested authentication and sync flow
- [ ] **Part 4**: Verified server logs show successful authentication

---

## Related Documentation

- [SELF_HOSTED_SYNC_SETUP.md](SELF_HOSTED_SYNC_SETUP.md) - Complete sync server setup guide
- [AWS Cognito Documentation](https://docs.aws.amazon.com/cognito/) - Official AWS docs
- [Logseq DB Sync Protocol](docs/agent-guide/db-sync/protocol.md) - Sync protocol details

---

## Questions?

Check existing issues or create a new one at: https://github.com/logseq/logseq/issues
