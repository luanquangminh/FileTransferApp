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

## 2. Operational Flow

### 2.1 System Overview
The system operates as a client-server architecture where:
- **Server Side:** C-based server handles all file operations, socket communications, and business logic
- **Client Side:** Next.js/React frontend provides user-friendly interface
- **Communication:** WebSocket/TCP socket connections for real-time bidirectional communication

### 2.2 User Authentication Flow

```
User → Web Interface (Login/Register)
  ↓
Frontend → Server: AUTH_REQUEST {username, password_hash}
  ↓
Server → Database: Validate credentials
  ↓
Server → Server: Generate JWT token
  ↓
Server → Client: AUTH_RESPONSE {token, user_id, status}
  ↓
Client: Store token, establish authenticated session
```

### 2.3 File Upload Flow

```
1. User selects file(s) via drag-drop or file picker
2. Frontend chunks large files (64KB blocks)
3. Client → Server: UPLOAD_INIT {token, filename, filesize, destination}
4. Server validates permissions and responds with session_id
5. For each chunk:
   - Client → Server: UPLOAD_CHUNK {session_id, chunk_number, data}
   - Server → Client: CHUNK_ACK {status}
6. Client → Server: UPLOAD_COMPLETE {session_id}
7. Server reassembles file, saves metadata, updates database
8. Server → Client: SUCCESS {file_id, path}
```

### 2.4 File Download Flow

```
1. User browses file tree and selects file
2. Client → Server: DOWNLOAD_REQUEST {token, file_id}
3. Server validates user permissions
4. Server chunks file and streams to client
5. Server → Client: DOWNLOAD_CHUNK {chunk_number, data} (multiple)
6. Client reassembles chunks with progress updates
7. Client initiates browser download
```

### 2.5 Directory Operations

**Upload Directory:**
- Recursively traverse local directory structure
- Create remote directories maintaining hierarchy
- Upload files with proper path relationships

**Download Directory:**
- Server creates temporary ZIP archive
- Stream archive to client
- Client extracts locally

### 2.6 Permission Management

- Each file/folder has: owner, group, permission bits (rwx)
- Owner can modify permissions through GUI
- Server validates all operations against permissions
- Permission changes are logged for audit

---

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

### 4.1 Protocol Specifications

**Protocol Name:** FTP-Custom  
**Transport:** TCP Sockets  
**Port:** 8080 (configurable)  
**Encoding:** Binary header + JSON/Binary payload

### 4.2 Message Format

```
┌─────────────────┬──────┬─────────────────────────┐
│ Field           │ Size │ Description             │
├─────────────────┼──────┼─────────────────────────┤
│ Magic Number    │ 4B   │ 0x46545053 ("FTPS")     │
│ Version         │ 1B   │ Protocol version (0x01) │
│ Command Code    │ 2B   │ Operation type          │
│ Sequence Number │ 4B   │ Message sequence ID     │
│ Payload Length  │ 4B   │ Length of payload       │
│ Checksum        │ 4B   │ CRC32 for integrity     │
│ Payload         │ Var  │ JSON metadata or binary │
└─────────────────┴──────┴─────────────────────────┘
Total Header: 19 bytes
```

### 4.3 Command Codes

| Code   | Command           | Description                    |
|--------|-------------------|--------------------------------|
| 0x0001 | AUTH_REQUEST      | User authentication            |
| 0x0002 | AUTH_RESPONSE     | Authentication result + token  |
| 0x0003 | REGISTER_REQUEST  | New user registration          |
| 0x0010 | LIST_DIR          | List directory contents        |
| 0x0011 | CHANGE_DIR        | Change current directory       |
| 0x0012 | MAKE_DIR          | Create new directory           |
| 0x0020 | UPLOAD_INIT       | Initialize file upload         |
| 0x0021 | UPLOAD_CHUNK      | Upload file chunk              |
| 0x0022 | UPLOAD_COMPLETE   | Finalize upload                |
| 0x0030 | DOWNLOAD_REQUEST  | Request file download          |
| 0x0031 | DOWNLOAD_CHUNK    | Stream file chunk              |
| 0x0040 | SEARCH_REQUEST    | Search files by name           |
| 0x0041 | SEARCH_RESPONSE   | Search results                 |
| 0x0050 | FILE_DELETE       | Delete file                    |
| 0x0051 | FILE_RENAME       | Rename file                    |
| 0x0052 | FILE_MOVE         | Move file                      |
| 0x0053 | FILE_COPY         | Copy file                      |
| 0x0060 | SET_PERMISSION    | Modify permissions             |
| 0x0061 | GET_PERMISSION    | Query permissions              |
| 0x0070 | GET_LOGS          | Retrieve activity logs         |
| 0x00FF | ERROR             | Error response                 |

### 4.4 Response Codes

| Code | Status               | Description                 |
|------|----------------------|-----------------------------|
| 200  | OK                   | Success                     |
| 201  | CREATED              | Resource created            |
| 400  | BAD_REQUEST          | Invalid request format      |
| 401  | UNAUTHORIZED         | Authentication failed       |
| 403  | FORBIDDEN            | Insufficient permissions    |
| 404  | NOT_FOUND            | File/directory not found    |
| 413  | PAYLOAD_TOO_LARGE    | File exceeds limit          |
| 500  | INTERNAL_ERROR       | Server error                |
| 507  | INSUFFICIENT_STORAGE | Server storage full         |

### 4.5 Example Protocol Messages

**Authentication Request:**
```json
{
  "command": "AUTH_REQUEST",
  "payload": {
    "username": "user123",
    "password_hash": "bcrypt_hash_here"
  }
}
```

**Authentication Response:**
```json
{
  "command": "AUTH_RESPONSE",
  "status": 200,
  "payload": {
    "token": "jwt_token_here",
    "user_id": 12345,
    "expires_at": 1698765432
  }
}
```

**Upload Init:**
```json
{
  "command": "UPLOAD_INIT",
  "token": "jwt_token",
  "payload": {
    "filename": "document.pdf",
    "filesize": 2048576,
    "destination": "/home/user/documents/"
  }
}
```

**Upload Chunk:**
```
Header (19 bytes) + Binary data (64KB)
```

### 4.6 Security Features

1. **Password Security:** bcrypt hashing with salt
2. **Session Management:** JWT tokens (24-hour expiration)
3. **Data Integrity:** CRC32 checksums on all transfers
4. **Rate Limiting:** Max 100 requests/minute per client
5. **Connection Timeout:** 5 minutes idle timeout

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
