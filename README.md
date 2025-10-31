# FileTransferApp
# Secure File Transfer System - Project Plan

**Course:** Computer Networks  
**Team Size:** 3 Members  
**Submission Deadline:** This Thursday  
**Platform:** Linux (C Socket Programming + Next.js/React GUI)

---

## 1. Project Title and Objectives

### 1.1 Project Title
**Secure File Transfer System with User Permission Management**

### 1.2 Project Objectives

1. Develop a robust client-server file transfer application using C socket programming on Linux platform
2. Implement comprehensive user management with secure authentication and session handling
3. Create an intuitive graphical user interface using Next.js/React for seamless user experience
4. Support efficient file and directory operations including upload, download, search, and management
5. Implement Linux-style permission management for secure file sharing and access control
6. Handle large file transfers efficiently with streaming and chunking mechanisms

---

# Operational Flow: Next.js + Firebase OAuth + C File Transfer

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Browser                             │
│                     (Next.js Frontend)                           │
└──────────────┬──────────────────────────────────────────────────┘
               │
               │ HTTPS
               │
┌──────────────▼──────────────────────────────────────────────────┐
│                    Next.js Backend                               │
│                  (API Routes + SSR)                              │
└──────┬───────────────────────────────┬──────────────────────────┘
       │                               │
       │ Firebase Admin SDK            │ Child Process / IPC
       │                               │
┌──────▼───────────────┐      ┌────────▼──────────────────────────┐
│  Firebase Services   │      │   C File Transfer Process         │
│  - Authentication    │      │   - TCP Client/Server             │
│  - Firestore DB      │      │   - File Operations               │
│  - Storage (optional)│      │                                   │
└──────────────────────┘      └───────────────────────────────────┘
```

## Technology Stack

### Frontend Layer
- **Framework:** Next.js 14+ (App Router)
- **UI Components:** React components
- **Styling:** Tailwind CSS / Material-UI
- **File Upload:** HTML5 File API, drag-and-drop

### Authentication Layer
- **Provider:** Google OAuth 2.0
- **Platform:** Firebase Authentication
- **Session Management:** NextAuth.js or Firebase Auth SDK

### Backend Layer
- **API:** Next.js API Routes (App Router)
- **Runtime:** Node.js
- **C Integration:** Child process execution or native addons

### Database Layer
- **Service:** Firebase Firestore
- **Data Storage:**
  - User profiles
  - File transfer history
  - Transfer metadata
  - Connection logs

### File Transfer Layer
- **Protocol:** TCP (existing C code)
- **Execution:** Spawned C processes
- **Communication:** stdio/IPC between Node.js and C

## Detailed Operational Flow

### Phase 1: User Authentication

#### Step 1.1: Initial Access
```
User → Next.js App
  1. User visits application URL
  2. Next.js checks for existing session (cookies/JWT)
  3. If no session: redirect to login page
  4. If session exists: validate with Firebase
```

#### Step 1.2: Google OAuth Login
```
User → Google OAuth → Firebase Auth → Next.js
  1. User clicks "Sign in with Google"
  2. Next.js redirects to Google OAuth consent screen
  3. User authorizes application
  4. Google returns authorization code
  5. Firebase Auth exchanges code for tokens
  6. Firebase creates/updates user record
  7. Next.js receives Firebase ID token
  8. Next.js creates session (httpOnly cookie)
  9. Redirect to dashboard
```

#### Step 1.3: Session Storage
```
Firebase Authentication
  ├── User UID (unique identifier)
  ├── Email
  ├── Display Name
  ├── Photo URL
  └── Custom Claims (role, permissions)

Next.js Session
  ├── Firebase ID Token (JWT)
  ├── Refresh Token
  └── Session expiry
```

### Phase 2: User Data Management

#### Step 2.1: Firestore Schema
```
users/
  {uid}/
    - email: string
    - displayName: string
    - photoURL: string
    - createdAt: timestamp
    - lastLogin: timestamp
    - transferCount: number

transfers/
  {transferId}/
    - userId: string (reference to users)
    - fileName: string
    - fileSize: number
    - status: "pending" | "transferring" | "completed" | "failed"
    - direction: "send" | "receive"
    - startTime: timestamp
    - endTime: timestamp
    - clientIP: string
    - serverIP: string
    - port: number
    - error: string (if failed)

sessions/
  {sessionId}/
    - userId: string
    - cProcessPID: number
    - socketInfo: object
    - createdAt: timestamp
    - status: "active" | "closed"
```

#### Step 2.2: Data Synchronization
```
1. User logs in → Create/update user document in Firestore
2. User initiates transfer → Create transfer document
3. C process updates → Next.js updates Firestore in real-time
4. User views history → Query transfers collection
```

### Phase 3: File Transfer Initiation

#### Step 3.1: File Selection (Frontend)
```javascript
// User Flow
User Action → File Input/Drag-Drop
  ↓
File Validation
  - Check file size limits
  - Check file type
  - Check user permissions
  ↓
Display Preview
  - File name
  - File size
  - Transfer mode (send/receive)
  ↓
User Confirms
```

#### Step 3.2: Transfer Request (API Route)
```javascript
// Next.js API Route: /api/transfer/send
POST /api/transfer/send
Headers: {
  Authorization: Bearer {firebaseToken}
}
Body: {
  fileName: "document.txt",
  fileSize: 2048,
  mode: "send",
  targetIP: "127.0.0.1",
  targetPort: 8080
}

Response: {
  transferId: "uuid",
  status: "initiated",
  processId: 12345
}
```

### Phase 4: C Process Integration

#### Step 4.1: Node.js to C Communication

**Option A: Child Process (Recommended)**
```javascript
// Next.js API Route Handler
import { spawn } from 'child_process';
import path from 'path';

export async function POST(request) {
  // 1. Verify Firebase token
  const user = await verifyToken(request);

  // 2. Parse request
  const { fileName, mode, targetIP, targetPort } = await request.json();

  // 3. Create Firestore transfer record
  const transferDoc = await createTransferRecord(user.uid, fileName);

  // 4. Spawn C process
  const cClient = spawn('./client', [], {
    cwd: path.join(process.cwd(), 'c-programs'),
    env: {
      ...process.env,
      FILE_NAME: fileName,
      TARGET_IP: targetIP,
      TARGET_PORT: targetPort
    }
  });

  // 5. Handle C process output
  cClient.stdout.on('data', (data) => {
    console.log(`C Client: ${data}`);
    updateTransferProgress(transferDoc.id, data.toString());
  });

  cClient.stderr.on('data', (data) => {
    console.error(`C Client Error: ${data}`);
    updateTransferError(transferDoc.id, data.toString());
  });

  cClient.on('close', (code) => {
    if (code === 0) {
      markTransferComplete(transferDoc.id);
    } else {
      markTransferFailed(transferDoc.id, `Exit code: ${code}`);
    }
  });

  return Response.json({ transferId: transferDoc.id, pid: cClient.pid });
}
```

**Option B: Native Addon (Advanced)**
```javascript
// Using node-gyp to create N-API bindings
const fileTransfer = require('./build/Release/filetransfer.node');

export async function POST(request) {
  const result = await fileTransfer.sendFile({
    fileName: 'send.txt',
    targetIP: '127.0.0.1',
    targetPort: 8080
  });
  return Response.json(result);
}
```

#### Step 4.2: Modified C Code Architecture

**Updated client.c with environment variables:**
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <arpa/inet.h>

#define SIZE 1024

// Read configuration from environment or arguments
void read_config(char **filename, char **ip, int *port) {
    char *env_file = getenv("FILE_NAME");
    char *env_ip = getenv("TARGET_IP");
    char *env_port = getenv("TARGET_PORT");

    *filename = env_file ? env_file : "send.txt";
    *ip = env_ip ? env_ip : "127.0.0.1";
    *port = env_port ? atoi(env_port) : 8080;
}

// Send progress updates to stdout (Node.js reads this)
void report_progress(const char *status, int bytes_sent) {
    printf("PROGRESS:%s:%d\n", status, bytes_sent);
    fflush(stdout);
}

int main(int argc, char *argv[]) {
    char *filename, *ip;
    int port;

    read_config(&filename, &ip, &port);

    // Report start
    report_progress("STARTED", 0);

    // ... existing socket and file transfer code ...

    // Report progress during transfer
    while(fgets(data, SIZE, fp) != NULL) {
        int len = strlen(data);
        if (send(sockfd, data, len, 0) == -1) {
            report_progress("ERROR", total_sent);
            exit(1);
        }
        total_sent += len;
        report_progress("SENDING", total_sent);
        bzero(data, SIZE);
    }

    report_progress("COMPLETED", total_sent);
    return 0;
}
```

### Phase 5: Real-time Progress Updates

#### Step 5.1: WebSocket Connection
```javascript
// Frontend: app/transfer/page.tsx
'use client';
import { useEffect, useState } from 'react';
import { db } from '@/lib/firebase';
import { doc, onSnapshot } from 'firebase/firestore';

export default function TransferPage({ transferId }) {
  const [progress, setProgress] = useState(null);

  useEffect(() => {
    // Real-time listener on Firestore
    const unsubscribe = onSnapshot(
      doc(db, 'transfers', transferId),
      (doc) => {
        setProgress(doc.data());
      }
    );

    return () => unsubscribe();
  }, [transferId]);

  return (
    <div>
      <h2>Transfer Progress</h2>
      <p>Status: {progress?.status}</p>
      <p>Bytes Sent: {progress?.bytesSent}</p>
      <ProgressBar value={progress?.percentage} />
    </div>
  );
}
```

### Phase 6: Complete User Journey

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. USER AUTHENTICATION                                           │
└─────────────────────────────────────────────────────────────────┘
    User opens app
      ↓
    Next.js checks session
      ↓
    No session → Redirect to /login
      ↓
    Click "Sign in with Google"
      ↓
    Google OAuth flow
      ↓
    Firebase creates user session
      ↓
    Next.js sets httpOnly cookie
      ↓
    Redirect to /dashboard

┌─────────────────────────────────────────────────────────────────┐
│ 2. DASHBOARD & FILE SELECTION                                    │
└─────────────────────────────────────────────────────────────────┘
    User sees dashboard
      ↓
    Firestore loads:
      - User profile
      - Transfer history
      - Active connections
      ↓
    User clicks "Send File"
      ↓
    File picker opens
      ↓
    User selects file
      ↓
    Frontend validates:
      - File size < 100MB
      - File type allowed
      ↓
    User enters:
      - Target IP (or selects peer)
      - Target Port (default 8080)
      ↓
    User clicks "Start Transfer"

┌─────────────────────────────────────────────────────────────────┐
│ 3. TRANSFER INITIATION                                           │
└─────────────────────────────────────────────────────────────────┘
    Frontend sends POST /api/transfer/send
      ↓
    Next.js API Route:
      1. Verify Firebase token
      2. Check user permissions
      3. Create Firestore transfer doc
      4. Save file to temp directory
      5. Spawn C client process
      6. Return transferId to frontend
      ↓
    Frontend navigates to /transfer/[id]

┌─────────────────────────────────────────────────────────────────┐
│ 4. FILE TRANSFER EXECUTION                                       │
└─────────────────────────────────────────────────────────────────┘
    C Client Process:
      1. Read environment config
      2. Create socket
      3. Connect to target
      4. Send progress to stdout
      ↓
    Node.js reads C stdout:
      - "PROGRESS:SENDING:1024"
      - "PROGRESS:SENDING:2048"
      ↓
    Node.js updates Firestore:
      transfers/{id} → { bytesSent: 2048, status: "transferring" }
      ↓
    Frontend Firestore listener:
      - Receives real-time updates
      - Updates progress bar
      - Shows transfer speed
      ↓
    C Process completes:
      - stdout: "PROGRESS:COMPLETED:5120"
      - exit code 0
      ↓
    Node.js updates Firestore:
      transfers/{id} → { status: "completed", endTime: now() }

┌─────────────────────────────────────────────────────────────────┐
│ 5. POST-TRANSFER                                                 │
└─────────────────────────────────────────────────────────────────┘
    Frontend shows completion message
      ↓
    User clicks "View History"
      ↓
    Firestore query:
      transfers.where('userId', '==', uid)
               .orderBy('startTime', 'desc')
      ↓
    Display transfer list:
      - File names
      - Transfer times
      - Success/failure status
      - Download logs
```

## Security Considerations

### Authentication Security
```
1. Firebase Token Validation
   - Verify token on every API request
   - Check token expiration
   - Validate custom claims

2. Session Management
   - HttpOnly cookies (prevent XSS)
   - Secure flag (HTTPS only)
   - SameSite=Strict (prevent CSRF)
   - Short expiration (1 hour)
   - Refresh token rotation

3. API Route Protection
   middleware.ts:
   - Check authentication
   - Rate limiting
   - CORS configuration
```

### File Transfer Security
```
1. Input Validation
   - Sanitize file names
   - Check file size limits
   - Validate IP addresses
   - Whitelist allowed ports

2. File Storage
   - Temporary file directory with restricted permissions
   - Auto-cleanup after transfer
   - Virus scanning (optional)

3. Process Isolation
   - C process runs with limited permissions
   - Timeout limits (kill hung processes)
   - Resource limits (CPU, memory)

4. Network Security
   - Option to use TLS/SSL wrapper
   - IP whitelist/blacklist
   - Port restrictions
```

## Database Schema Details

### Firestore Collections

```javascript
// users collection
{
  uid: "firebase-user-id",
  email: "user@example.com",
  displayName: "John Doe",
  photoURL: "https://...",
  createdAt: Timestamp,
  lastLogin: Timestamp,
  settings: {
    defaultPort: 8080,
    autoAccept: false,
    notifyOnTransfer: true
  },
  stats: {
    totalTransfers: 42,
    totalBytesSent: 1048576,
    totalBytesReceived: 2097152
  }
}

// transfers collection
{
  id: "transfer-uuid",
  userId: "firebase-user-id",
  fileName: "document.txt",
  fileSize: 5120,
  filePath: "/tmp/uploads/document.txt",
  direction: "send", // or "receive"
  peer: {
    ip: "127.0.0.1",
    port: 8080
  },
  status: "completed", // pending, transferring, completed, failed
  progress: {
    bytesSent: 5120,
    bytesReceived: 5120,
    percentage: 100,
    speed: 1024 // bytes per second
  },
  timestamps: {
    created: Timestamp,
    started: Timestamp,
    completed: Timestamp
  },
  process: {
    pid: 12345,
    exitCode: 0
  },
  error: null // or error message
}

// logs collection (optional)
{
  id: "log-uuid",
  transferId: "transfer-uuid",
  timestamp: Timestamp,
  level: "info", // info, warning, error
  message: "Connection established",
  source: "c-client" // or "next-api"
}
```

## API Endpoints

### Authentication Endpoints
```
POST /api/auth/login
  - Initiates Google OAuth flow

POST /api/auth/callback
  - Handles OAuth callback

POST /api/auth/logout
  - Clears session

GET /api/auth/session
  - Returns current user session
```

### Transfer Endpoints
```
POST /api/transfer/send
  Body: { fileName, fileSize, targetIP, targetPort }
  Response: { transferId, processId }

POST /api/transfer/receive
  Body: { port }
  Response: { transferId, processId }

GET /api/transfer/[id]
  Response: { transfer details }

DELETE /api/transfer/[id]
  - Cancel active transfer

GET /api/transfer/history
  Query: { limit, offset }
  Response: { transfers[] }
```

### User Endpoints
```
GET /api/user/profile
  Response: { user data }

PATCH /api/user/profile
  Body: { settings }

GET /api/user/stats
  Response: { transfer statistics }
```

## Frontend Pages Structure

```
app/
├── (auth)/
│   ├── login/
│   │   └── page.tsx          # Google OAuth login page
│   └── callback/
│       └── page.tsx          # OAuth callback handler
│
├── (dashboard)/
│   ├── layout.tsx            # Protected layout (requires auth)
│   ├── page.tsx              # Dashboard home
│   ├── transfer/
│   │   ├── send/
│   │   │   └── page.tsx      # Send file interface
│   │   ├── receive/
│   │   │   └── page.tsx      # Receive file interface
│   │   └── [id]/
│   │       └── page.tsx      # Transfer progress page
│   ├── history/
│   │   └── page.tsx          # Transfer history
│   └── profile/
│       └── page.tsx          # User profile settings
│
└── api/
    ├── auth/
    │   ├── login/route.ts
    │   ├── callback/route.ts
    │   └── logout/route.ts
    ├── transfer/
    │   ├── send/route.ts
    │   ├── receive/route.ts
    │   └── [id]/route.ts
    └── user/
        ├── profile/route.ts
        └── stats/route.ts
```

## Environment Configuration

### .env.local
```bash
# Firebase Configuration
NEXT_PUBLIC_FIREBASE_API_KEY=your-api-key
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
NEXT_PUBLIC_FIREBASE_PROJECT_ID=your-project-id
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=your-project.appspot.com
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=123456789
NEXT_PUBLIC_FIREBASE_APP_ID=your-app-id

# Firebase Admin (Server-side only)
FIREBASE_ADMIN_PROJECT_ID=your-project-id
FIREBASE_ADMIN_CLIENT_EMAIL=firebase-adminsdk@your-project.iam.gserviceaccount.com
FIREBASE_ADMIN_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n..."

# Google OAuth
GOOGLE_CLIENT_ID=your-client-id.apps.googleusercontent.com
GOOGLE_CLIENT_SECRET=your-client-secret

# Application
NEXT_PUBLIC_APP_URL=http://localhost:3000
C_PROGRAMS_PATH=/path/to/c-programs

# File Transfer Settings
MAX_FILE_SIZE=104857600  # 100MB
TEMP_UPLOAD_DIR=/tmp/file-transfers
DEFAULT_TRANSFER_PORT=8080
```

## Deployment Considerations

### Development Setup
```bash
# 1. Compile C programs
cd c-programs
gcc -o client client.c
gcc -o server server.c

# 2. Install Node.js dependencies
npm install

# 3. Configure Firebase
# - Create Firebase project
# - Enable Google OAuth
# - Download service account key

# 4. Run development server
npm run dev
```

### Production Deployment

**Option 1: Vercel + Separate C Server**
```
Next.js → Vercel (serverless)
C Programs → Separate VPS/EC2 instance
Communication → HTTP API between Next.js and C server
```

**Option 2: VPS/EC2 (Recommended)**
```
Single server running:
  - Next.js (Node.js process)
  - C programs (spawned processes)
  - Nginx (reverse proxy)
```

**Option 3: Docker Containers**
```yaml
# docker-compose.yml
services:
  nextjs:
    build: .
    ports:
      - "3000:3000"
    environment:
      - FIREBASE_CONFIG=${FIREBASE_CONFIG}
    volumes:
      - ./c-programs:/app/c-programs

  c-server:
    build: ./c-programs
    network_mode: "host"
```

## Error Handling & Edge Cases

### Scenarios to Handle
```
1. User loses internet during transfer
   → Implement retry logic
   → Save partial transfer state

2. C process crashes
   → Catch exit codes
   → Log error to Firestore
   → Notify user

3. Target server unavailable
   → Connection timeout
   → Display user-friendly error

4. File size exceeds limit
   → Validate before upload
   → Show error message

5. Firebase quota exceeded
   → Implement caching
   → Rate limiting

6. Concurrent transfers
   → Queue system
   → Limit simultaneous transfers
```

## Performance Optimization

### Frontend
- Code splitting (dynamic imports)
- Image optimization (Next.js Image)
- Lazy loading components
- Memoization for expensive renders

### Backend
- Connection pooling
- Caching with Redis (optional)
- Batch Firestore writes
- Stream large files (avoid loading in memory)

### C Programs
- Non-blocking I/O (epoll/select)
- Buffer optimization
- Memory management

## Future Enhancements

### Phase 2 Features
- End-to-end encryption
- Peer-to-peer discovery
- Resume interrupted transfers
- Multi-file transfers
- Compression
- Transfer scheduling

### Phase 3 Features
- Mobile apps (React Native)
- WebRTC data channels (browser-to-browser)
- CDN integration for large files
- Admin dashboard
- Analytics and monitoring

## Success Metrics

### Technical Metrics
- Transfer success rate > 99%
- Average transfer speed > 1 MB/s
- API response time < 200ms
- Page load time < 2s
- C process spawn time < 100ms

### User Metrics
- Authentication success rate > 95%
- User retention (weekly active users)
- Average transfers per user
- Error rate < 1%

## Conclusion

This architecture combines modern web technologies (Next.js, Firebase) with existing C file transfer code, providing:

- **Scalable** web interface with real-time updates
- **Secure** authentication via Google OAuth
- **Reliable** file transfers using proven TCP code
- **Observable** system with comprehensive logging
- **Extensible** design for future enhancements

The key integration point is the Next.js API routes spawning C processes and capturing their output for real-time progress updates stored in Firestore.


## 3. System Analysis and Design (UML Diagrams)

### 3.1 Use Case Diagram

**Actors:**
- Anonymous User
- Registered User
- System Administrator (optional)

**Use Cases:**

| Actor | Use Cases |
|-------|-----------|
| Anonymous User | • Register Account<br>• Login to System |
| Registered User | • Upload File/Directory<br>• Download File/Directory<br>• Browse File System<br>• Search Files<br>• Rename/Delete/Copy/Move Files<br>• Create/Rename/Delete Directories<br>• Set File/Folder Permissions<br>• View Activity Logs |

### 3.2 Sequence Diagrams

#### 3.2.1 Authentication Sequence
```
Client          Server          Database
  |               |                 |
  |--CONNECT----->|                 |
  |               |                 |
  |--AUTH_REQ---->|                 |
  |               |--VALIDATE------>|
  |               |<--USER_DATA-----|
  |               |                 |
  |               |--GENERATE_JWT-->|
  |<--AUTH_RESP---|                 |
  |   (token)     |                 |
```

#### 3.2.2 File Upload Sequence
```
Client          Server          FileSystem
  |               |                 |
  |--UPLOAD_INIT->|                 |
  |               |--CHECK_PERM---->|
  |<--READY-------|                 |
  |               |                 |
  |--CHUNK_1----->|                 |
  |<--ACK_1-------|                 |
  |--CHUNK_2----->|                 |
  |<--ACK_2-------|                 |
  |     ...       |                 |
  |--COMPLETE---->|                 |
  |               |--SAVE_FILE----->|
  |<--SUCCESS-----|                 |
```

#### 3.2.3 Search and Download Sequence
```
Client          Server          Database
  |               |                 |
  |--SEARCH_REQ-->|                 |
  |               |--QUERY_FILES--->|
  |               |<--RESULTS-------|
  |               |--FILTER_PERMS-->|
  |<--SEARCH_RES--|                 |
  |               |                 |
  |--DOWNLOAD---->|                 |
  |               |--VALIDATE------>|
  |<--CHUNK_1-----|                 |
  |<--CHUNK_2-----|                 |
  |     ...       |                 |
```

### 3.3 Class Diagram (C Server Components)

```
┌─────────────────┐
│   SocketServer  │
├─────────────────┤
│ - port          │
│ - max_clients   │
├─────────────────┤
│ + init()        │
│ + listen()      │
│ + accept_conn() │
└─────────────────┘
        │
        ├──────────────────┐
        │                  │
┌───────▼──────┐   ┌───────▼──────┐
│ AuthHandler  │   │ FileHandler  │
├──────────────┤   ├──────────────┤
│ - db_conn    │   │ - storage    │
├──────────────┤   ├──────────────┤
│ + register() │   │ + upload()   │
│ + login()    │   │ + download() │
│ + validate() │   │ + search()   │
└──────────────┘   └──────────────┘

┌─────────────────┐
│ PermissionMgr   │
├─────────────────┤
│ + check_perm()  │
│ + set_perm()    │
│ + get_owner()   │
└─────────────────┘
```

---

## 4. Application-Layer Protocol Design

### 4.1 Protocol Overview

This application implements a **simple TCP-based file transfer protocol** for transferring text files between client and server. The protocol operates on **port 8080** and uses raw TCP socket communication without additional protocol headers or authentication mechanisms.

### 4.2 Protocol Architecture

**Transport Layer:** TCP (Transmission Control Protocol)
**Port:** 8080
**IP Address:** 127.0.0.1 (localhost)
**Connection Model:** One-to-one (single client per server instance)

### 4.3 Message Format

The protocol uses a **raw data stream** approach with no custom headers:

```
+------------------+
|   File Data      |
|   (Raw bytes)    |
+------------------+
```

- **No Protocol Header:** Data is sent directly without metadata
- **Buffer Size:** 1024 bytes per transmission chunk
- **Data Format:** Text data read line-by-line using `fgets()`

### 4.4 Communication Flow

#### **Client-Side Operation:**
1. Establish TCP connection to server (127.0.0.1:8080)
2. Open source file (`send.txt`) for reading
3. Read file data in 1024-byte chunks using `fgets()`
4. Send each chunk via `send()` system call
5. Close connection after complete file transmission

#### **Server-Side Operation:**
1. Bind to port 8080 and listen for incoming connections
2. Accept single client connection
3. Receive data chunks using `recv()` with 1024-byte buffer
4. Write received data to destination file (`recv.txt`)
5. Continue until connection closes (recv returns ≤ 0)

### 4.5 Data Transfer Mechanism

**Transmission Method:** Sequential streaming
- Client reads file line-by-line
- Each read operation transfers up to SIZE (1024) bytes
- Buffer is cleared with `bzero()` between transmissions
- Server writes data immediately to output file

**End-of-Transfer Detection:**
- Connection closure signals end of file transfer
- Server detects when `recv()` returns ≤ 0 bytes

### 4.6 Error Handling

**Client-Side Errors:**
- Socket creation failure
- Connection refused
- File not found (`send.txt`)

**Server-Side Errors:**
- Socket creation failure
- Bind failure (port already in use)
- Listen failure
- File write errors

All errors trigger program termination via `exit(1)`.

### 4.7 Protocol Limitations

- **No Authentication:** No user verification or access control
- **No Encryption:** Data transmitted in plaintext
- **No Integrity Checking:** No checksums or hash verification
- **No File Metadata:** Filename, size, and permissions not transmitted
- **Single File Transfer:** Server handles one file per execution
- **No Resume Capability:** Interrupted transfers cannot be resumed
- **Fixed Filenames:** Hardcoded `send.txt` and `recv.txt`
- **Text-Only Optimization:** Uses `fgets()` which is text-oriented

### 4.8 Technical Specifications

| Parameter | Value |
|-----------|-------|
| Protocol Type | Raw TCP Stream |
| Port | 8080 |
| Buffer Size | 1024 bytes |
| Max Connections | 10 (listen backlog) |
| File I/O Mode | Text mode (line-based) |
| Socket Type | SOCK_STREAM (TCP) |
| Address Family | AF_INET (IPv4) |

---

## 5. Database Design

### 5.1 Database: SQLite

**Rationale:** Lightweight, embedded, serverless, perfect for file system applications

### 5.2 Schema Design

#### Table: users
```sql
CREATE TABLE users (
    user_id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE NOT NULL,
    password_hash TEXT NOT NULL,
    email TEXT UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    storage_quota INTEGER DEFAULT 1073741824  -- 1GB default
);

CREATE INDEX idx_username ON users(username);
```

#### Table: files
```sql
CREATE TABLE files (
    file_id INTEGER PRIMARY KEY AUTOINCREMENT,
    filename TEXT NOT NULL,
    filepath TEXT NOT NULL,
    filesize INTEGER NOT NULL,
    owner_id INTEGER NOT NULL,
    permissions INTEGER NOT NULL,  -- Unix-style: 755, 644, etc.
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    modified_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_directory BOOLEAN DEFAULT 0,
    parent_id INTEGER,
    FOREIGN KEY (owner_id) REFERENCES users(user_id),
    FOREIGN KEY (parent_id) REFERENCES files(file_id)
);

CREATE INDEX idx_owner ON files(owner_id);
CREATE INDEX idx_parent ON files(parent_id);
CREATE INDEX idx_filename ON files(filename);
CREATE UNIQUE INDEX idx_filepath ON files(filepath);
```

#### Table: activity_logs
```sql
CREATE TABLE activity_logs (
    log_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    action TEXT NOT NULL,  -- 'UPLOAD', 'DOWNLOAD', 'DELETE', etc.
    file_id INTEGER,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    ip_address TEXT,
    details TEXT,
    FOREIGN KEY (user_id) REFERENCES users(user_id),
    FOREIGN KEY (file_id) REFERENCES files(file_id)
);

CREATE INDEX idx_user_logs ON activity_logs(user_id);
CREATE INDEX idx_timestamp ON activity_logs(timestamp);
```

#### Table: sessions
```sql
CREATE TABLE sessions (
    session_id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER NOT NULL,
    token TEXT UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    ip_address TEXT,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

CREATE INDEX idx_token ON sessions(token);
```

### 5.3 Permission System

**Permission Encoding (Unix-style):**
- Read (4), Write (2), Execute (1)
- Owner-Group-Others: e.g., 755 = rwxr-xr-x

**Permission Check Algorithm:**
```c
int check_permission(int file_perms, int user_id, int owner_id, char mode) {
    // mode: 'r' = read, 'w' = write, 'x' = execute
    int owner_perms = (file_perms / 100) % 10;
    int group_perms = (file_perms / 10) % 10;
    int other_perms = file_perms % 10;
    
    int required = (mode == 'r') ? 4 : (mode == 'w') ? 2 : 1;
    
    if (user_id == owner_id)
        return (owner_perms & required) != 0;
    // Add group logic here
    return (other_perms & required) != 0;
}
```

---

## 6. Work Assignment and Timeline

### 6.1 Team Structure

**Member 1 - Backend Lead:**
- C socket server implementation
- Protocol implementation
- Database operations
- File I/O operations

**Member 2 - Frontend Lead:**
- Next.js/React UI components
- WebSocket client integration
- File upload/download UI
- State management

**Member 3 - Integration & Testing:**
- Authentication system
- Permission management
- Testing and debugging
- Documentation

### 6.2 Development Timeline (Before Thursday)

#### Day 1 (Monday) - Foundation
**Member 1:**
- [ ] Set up C project structure
- [ ] Implement basic socket server (listen, accept, read/write)
- [ ] Create protocol parser for message format
- [ ] Set up SQLite database with schema

**Member 2:**
- [ ] Initialize Next.js project with TypeScript
- [ ] Set up WebSocket client connection
- [ ] Create basic UI layout (login, file browser)
- [ ] Implement authentication forms

**Member 3:**
- [ ] Design and document protocol specification
- [ ] Create UML diagrams (use case, sequence)
- [ ] Set up testing framework
- [ ] Write authentication logic (bcrypt, JWT)

**End of Day Goal:** Basic client-server connection established

---

#### Day 2 (Tuesday) - Core Features
**Member 1:**
- [ ] Implement AUTH_REQUEST/RESPONSE handlers
- [ ] Implement UPLOAD_INIT, UPLOAD_CHUNK handlers
- [ ] Implement DOWNLOAD_REQUEST, DOWNLOAD_CHUNK handlers
- [ ] Add file chunking logic (64KB chunks)
- [ ] Implement LIST_DIR, MAKE_DIR commands

**Member 2:**
- [ ] Build file browser component with tree view
- [ ] Implement file upload UI (drag-drop, progress bar)
- [ ] Implement file download functionality
- [ ] Add directory navigation
- [ ] Create search interface

**Member 3:**
- [ ] Implement user registration endpoint
- [ ] Implement permission checking logic
- [ ] Add activity logging system
- [ ] Test upload/download flows
- [ ] Create test files and scenarios

**End of Day Goal:** File upload and download working

---

#### Day 3 (Wednesday) - Advanced Features & Polish
**Member 1:**
- [ ] Implement SEARCH_REQUEST handler with SQL queries
- [ ] Implement file operations (rename, delete, copy, move)
- [ ] Implement directory operations
- [ ] Add directory upload/download (ZIP creation)
- [ ] Handle large files (>100MB) efficiently

**Member 2:**
- [ ] Implement permission management UI
- [ ] Add activity logs viewer
- [ ] Implement directory upload/download
- [ ] Add file operation buttons (rename, delete, etc.)
- [ ] Polish UI/UX, add loading states

**Member 3:**
- [ ] Implement SET_PERMISSION/GET_PERMISSION handlers
- [ ] Add error handling throughout system
- [ ] Comprehensive testing of all features
- [ ] Write user documentation
- [ ] Performance testing and optimization

**End of Day Goal:** All features functional

---

#### Day 4 (Thursday Morning) - Final Integration & Documentation
**All Members:**
- [ ] Final integration testing
- [ ] Bug fixes and edge case handling
- [ ] Complete project documentation
- [ ] Prepare presentation slides
- [ ] Create demo scenarios
- [ ] Final code review and cleanup

**Presentation Preparation:**
- [ ] Live demo setup
- [ ] Backup demo video (if live demo fails)
- [ ] Q&A preparation

---

### 6.3 Expected Deliverables

#### Code Deliverables:
1. **C Server Code** (`server/`)
   - `main.c` - Server entry point
   - `socket_handler.c` - Socket operations
   - `protocol.c` - Protocol parsing
   - `auth.c` - Authentication
   - `file_ops.c` - File operations
   - `db.c` - Database operations
   - `permission.c` - Permission management

2. **Next.js Client** (`client/`)
   - `/pages` - React pages
   - `/components` - UI components
   - `/lib/socket.ts` - WebSocket client
   - `/lib/api.ts` - API functions

3. **Database**
   - `schema.sql` - Database schema
   - `init_data.sql` - Test data

#### Documentation:
1. **Project Report** (This document)
2. **API Documentation** (Protocol specification)
3. **User Manual** (How to use the system)
4. **Installation Guide** (Setup instructions)

#### Presentation:
1. PowerPoint slides
2. Live demo
3. Code walkthrough

---

## 7. Current Progress Status

### Completed:
- ✅ Project planning and requirement analysis
- ✅ Protocol design specification
- ✅ Database schema design
- ✅ UML diagrams (use case, sequence)
- ✅ Work assignment and timeline

### In Progress:
- 🔄 Setting up development environment
- 🔄 Basic project structure

### Pending:
- ⏳ All implementation tasks (Days 1-4)
- ⏳ Testing and integration
- ⏳ Final documentation

---

## 8. Technical Architecture

### 8.1 Server Architecture (C)

```
┌─────────────────────────────────────────┐
│         Main Server Process             │
├─────────────────────────────────────────┤
│  Socket Listener (Port 8080)            │
│  │                                       │
│  ├──> Connection Handler (fork/thread)  │
│  │     │                                 │
│  │     ├──> Protocol Parser              │
│  │     ├──> Authentication Handler       │
│  │     ├──> File Operation Handler       │
│  │     └──> Response Generator           │
│  │                                       │
│  └──> Client Connection Pool            │
└─────────────────────────────────────────┘
          ↓                    ↓
    ┌──────────┐        ┌──────────┐
    │ Database │        │   File   │
    │  SQLite  │        │  Storage │
    └──────────┘        └──────────┘
```

### 8.2 Client Architecture (Next.js/React)

```
┌─────────────────────────────────────────┐
│           Next.js App                   │
├─────────────────────────────────────────┤
│  Pages:                                 │
│  ├─ /login                              │
│  ├─ /register                           │
│  └─ /dashboard                          │
│     ├─ File Browser Component           │
│     ├─ Upload Component                 │
│     ├─ Search Component                 │
│     └─ Settings Component               │
├─────────────────────────────────────────┤
│  State Management (Context API)         │
├─────────────────────────────────────────┤
│  WebSocket Client (socket.io-client)    │
└─────────────────────────────────────────┘
          ↓
    ┌──────────┐
    │ C Server │
    │ :8080    │
    └──────────┘
```

### 8.3 File Storage Structure

```
/var/fileserver/
├── users/
│   ├── user_1/
│   │   ├── documents/
│   │   ├── images/
│   │   └── uploads/
│   ├── user_2/
│   └── ...
├── temp/
│   └── upload_sessions/
└── logs/
    └── activity.log
```

---

## 9. Development Guidelines

### 9.1 C Server Best Practices

```c
// Use consistent error handling
#define CHECK_ERROR(expr, msg) \
    if ((expr) < 0) { \
        perror(msg); \
        return -1; \
    }

// Use proper socket options
int opt = 1;
setsockopt(socket_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

// Use non-blocking I/O with select() or poll()
fd_set readfds;
FD_ZERO(&readfds);
FD_SET(socket_fd, &readfds);

// Always close file descriptors
close(client_fd);
```

### 9.2 Protocol Implementation

```c
// Message structure
typedef struct {
    uint32_t magic;      // 0x46545053
    uint8_t version;     // 0x01
    uint16_t command;
    uint32_t sequence;
    uint32_t payload_len;
    uint32_t checksum;
} __attribute__((packed)) MessageHeader;

// Send message
int send_message(int fd, uint16_t cmd, void* payload, size_t len);

// Receive message
int recv_message(int fd, MessageHeader* header, void* payload);
```

### 9.3 Security Checklist

- [ ] Never store plaintext passwords
- [ ] Validate all user inputs
- [ ] Check permissions before every operation
- [ ] Use prepared statements for SQL queries
- [ ] Implement rate limiting
- [ ] Log all security-relevant events
- [ ] Use secure random for tokens
- [ ] Validate JWT tokens on every request

---

## 10. Testing Strategy

### 10.1 Unit Tests

**Server (C):**
- Protocol parser tests
- Authentication logic tests
- Permission checking tests
- File operation tests

**Client (React):**
- Component rendering tests
- API function tests
- State management tests

### 10.2 Integration Tests

- [ ] Complete upload/download cycle
- [ ] Authentication flow
- [ ] Permission enforcement
- [ ] Multi-user scenarios
- [ ] Error handling

### 10.3 Performance Tests

- [ ] Large file transfer (1GB+)
- [ ] Multiple concurrent users (10+)
- [ ] Directory with many files (1000+)
- [ ] Network interruption recovery

### 10.4 Test Scenarios

1. **Basic Flow:**
   - Register → Login → Upload file → Download file → Logout

2. **Permission Test:**
   - User A uploads file with 600 permissions
   - User B attempts to access (should fail)
   - User A changes to 644
   - User B can now read

3. **Large File:**
   - Upload 500MB file
   - Verify chunking works
   - Download and verify checksum matches

4. **Directory Operations:**
   - Upload directory with subdirectories
   - Verify structure maintained
   - Download as ZIP
   - Extract and verify contents

---

## 11. Key Implementation Notes

### 11.1 Socket Programming (C)

**Must Use:**
- `socket()`, `bind()`, `listen()`, `accept()`
- `send()`, `recv()` for data transfer
- `select()` or `poll()` for multiplexing
- Linux platform only (no Winsock)

**Recommended Libraries:**
- SQLite3 for database
- OpenSSL for hashing (bcrypt)
- zlib for compression

### 11.2 Next.js Integration

**WebSocket Connection:**
```typescript
// lib/socket.ts
import { io } from 'socket.io-client';

const socket = io('http://localhost:8080');

socket.on('connect', () => {
  console.log('Connected to server');
});

// Send file chunk
socket.emit('upload_chunk', {
  sessionId,
  chunkNumber,
  data: arrayBuffer
});
```

**File Upload Component:**
```typescript
const handleFileUpload = async (file: File) => {
  const chunkSize = 64 * 1024; // 64KB
  const chunks = Math.ceil(file.size / chunkSize);
  
  for (let i = 0; i < chunks; i++) {
    const start = i * chunkSize;
    const end = Math.min(start + chunkSize, file.size);
    const chunk = file.slice(start, end);
    const buffer = await chunk.arrayBuffer();
    
    await uploadChunk(sessionId, i, buffer);
    setProgress((i + 1) / chunks * 100);
  }
};
```

---

## 12. Scoring Breakdown (Target Points)

| Feature | Points | Priority |
|---------|--------|----------|
| Stream handling | 1 | HIGH |
| Socket I/O on server | 2 | HIGH |
| Account registration | 2 | HIGH |
| Login & session management | 2 | HIGH |
| File upload/download | 2 | HIGH |
| Large file handling | 2 | HIGH |
| Upload/download directories | 3 | MEDIUM |
| File operations (CRUD) | 2 | MEDIUM |
| Directory operations | 2 | MEDIUM |
| File search and selection | 3 | MEDIUM |
| Log activity | 1 | LOW |
| Permission management | 3-7 | HIGH |
| GUI | 3 | HIGH |
| **TOTAL** | **30-34** | |

**Target:** Achieve 28-32 points for excellent grade

---

## 13. Risks and Mitigation

### High Priority Risks:

1. **Risk:** Socket programming complexity
   - **Mitigation:** Start with simple echo server, iterate

2. **Risk:** Large file handling issues
   - **Mitigation:** Implement chunking early, test with 100MB files

3. **Risk:** Permission system complexity
   - **Mitigation:** Start with simple owner-only, expand later

4. **Risk:** Time constraints
   - **Mitigation:** Focus on core features first (auth + upload/download)

### Contingency Plans:

- **If behind schedule:** Cut permission management to basic level
- **If socket issues:** Simplify to single-threaded server
- **If GUI problems:** Create minimal but functional interface

---

## 14. Questions for Professor's Lecture Next Week

1. Should we use raw TCP sockets or can we use a library for WebSocket?
2. What is the expected level of security (encryption)?
3. Should we implement concurrent connections (threading)?
4. Is compression required for directory downloads?
5. What is the maximum file size we should support?

---

## 15. References and Resources

### Documentation:
- Beej's Guide to Network Programming
- Linux Socket Programming (man pages)
- SQLite C API Documentation
- Next.js Documentation
- React Documentation

### Tools:
- GCC compiler
- Valgrind (memory leak detection)
- Wireshark (protocol debugging)
- Postman/curl (API testing)

---

## Conclusion

This project plan provides a comprehensive roadmap for developing the Secure File Transfer System within the 4-day timeframe. By dividing responsibilities among team members and following the structured timeline, we aim to deliver a fully functional system that meets all requirements.

**Next Steps:**
1. Review and approve this plan with team
2. Set up development environments (today)
3. Begin Day 1 tasks tomorrow
4. Daily standup meetings to track progress
5. Prepare presentation materials alongside development

**Success Criteria:**
- All core features working (auth, upload, download, search)
- Clean, documented code
- Successful live demo
- Comprehensive project documentation
- Passing score on all grading criteria

---

**Document Version:** 1.0  
**Last Updated:** October 28, 2025  
**Status:** Ready for implementation
