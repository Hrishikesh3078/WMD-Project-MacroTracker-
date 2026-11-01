# SRAMS - Smart Room Allotment and Management System

A complete but deliberately **simple** project for allotting rooms (classrooms, labs,
seminar halls) in a college. It has three parts that talk to each other over a REST API:

| Part | Technology | Folder | What it does |
|------|------------|--------|--------------|
| REST API + database | **Kotlin** (no external libraries) | `backend/` | Keeps rooms, users and requests. Approves or rejects bookings. |
| Website | **React** + Vite | `web/` | Dashboard, room management, approve/reject requests. |
| Mobile app | **Flutter** | `mobile/` | Browse rooms, request a room, follow your requests. |
| Login | **Firebase** Authentication (or built-in demo mode) | `backend/`, `web/`, `mobile/` | E-mail and password sign in. |

Everything runs **without Firebase first** (demo mode), so you can see the whole
project working before you create a Firebase account.

---

## 1. What you need to install

| Tool | Why | Check it with |
|------|-----|---------------|
| Kotlin compiler (`kotlinc`) + a JDK (17 or newer) | to build the backend | `kotlinc -version` |
| Node.js (18 or newer) | to run the React website | `node -v` |
| Flutter SDK | to run the mobile app | `flutter --version` |

You do **not** need Gradle, Maven, a database server, Spring or any npm package
for the backend - it uses only what is built into the JDK.

---

## 2. Start the backend (always start this first)

```bat
cd backend
run.bat
```

`run.bat` compiles `src\` into `srams.jar` the first time and then starts the API.

Check it in a browser: <http://localhost:8090/api/health>

You should see:

```json
{"status":"ok","service":"SRAMS REST API","demoMode":true,"time":"..."}
```

Useful settings (set them before starting, only if you need to):

| Setting | Default | Meaning |
|---------|---------|---------|
| `SRAMS_PORT` | `8090` | The API port. 8080 is often already used by other programs, so SRAMS uses 8090. If you change it here, also change `API_BASE_URL` in `web/src/config.js` and `apiBaseUrl` in `mobile/lib/config.dart`. |
| `SRAMS_DEMO` | `true` | `false` = really check Firebase ID tokens. |
| `SRAMS_FIREBASE_PROJECT_ID` | *(empty)* | Your Firebase project id, needed when `SRAMS_DEMO=false`. |
| `SRAMS_ADMIN_EMAILS` | `admin@srams.edu` | Comma separated list of administrator e-mails. |

Data is saved in `backend\data\` as three readable text files
(`rooms.txt`, `requests.txt`, `users.txt`). Delete that folder to start fresh.

---

## 3. Start the website

```bat
cd web
npm install
npm run dev
```

Open <http://localhost:5173>. The website reads the API address from
`web/src/config.js` (default `http://localhost:8090/api`).

---

## 4. Start the mobile app

```bat
cd mobile
flutter create .      :: only the first time - adds the android/ios folders
flutter pub get
flutter run
```

`flutter create .` keeps your `lib\` folder and `pubspec.yaml` and only adds the
platform folders that are missing, so run it once after downloading the project.

The app reads the API address from `mobile/lib/config.dart`:

* **Android emulator**: leave `http://10.0.2.2:8090/api` (an emulator cannot see `localhost`)
* **Real phone**: use your computer's address, for example `http://192.168.1.5:8090/api`
* **Windows / Chrome**: `http://localhost:8090/api`

---

## 5. Log in (demo mode)

Any e-mail address works while Firebase is not configured.

| E-mail | Role | What you can do |
|--------|------|-----------------|
| `admin@srams.edu` | Administrator | add/edit/delete rooms, approve or reject requests, see everything |
| `student@srams.edu` | Normal user | browse rooms, request a room, cancel a waiting request |

---

## 6. Turning on real Firebase login

Follow **[docs/FIREBASE_SETUP.md](docs/FIREBASE_SETUP.md)**. In short:

1. Create a Firebase project and enable **Authentication -> Email/Password**.
2. Paste the web values into `web/src/config.js`.
3. Paste the Android values into `mobile/lib/config.dart`.
4. Restart the backend with `set SRAMS_DEMO=false` and
   `set SRAMS_FIREBASE_PROJECT_ID=your-project-id`.

The apps then ask Firebase to check the e-mail and password, send the received
token to the backend, and the backend checks the token signature against
Google's public certificates (`backend/src/Auth.kt`).

---

## 7. How the three parts fit together

```
   Flutter app  ─┐
                 ├──►  REST API (Kotlin, port 8090)  ──►  data\*.txt
   React website ┘            ▲
                              │ verifies the login token
                        Firebase Authentication
```

* Every screen calls the same REST endpoints, so both apps behave the same way.
* The role (administrator or normal user) is decided by the backend, never by the app.

---

## 8. Project objectives and where they live in the code

| Objective | How SRAMS does it | Main files |
|-----------|-------------------|------------|
| Automated and streamlined allocation | One request form; an administrator approves with one button | `web/src/pages/RequestsPage.jsx`, `mobile/lib/screens/requests_tab.dart` |
| Real-time occupancy and availability tracking | Room status is recalculated after every approval or cancellation; dashboard shows live numbers | `backend/src/Store.kt` (`refreshRoomStatus`), `GET /api/stats` |
| Optimized resource and space management | Search and filter by building, type and status; capacity is shown everywhere | `GET /api/rooms?q=&status=`, `web/src/pages/RoomsPage.jsx` |
| Transparent and fair allotment | Every request stores who asked, why, for when, who decided and the reason | `AllotmentRequest` in `backend/src/Models.kt` |
| Enhanced user and administrator experience | Separate admin permissions, clear status colours, friendly error messages | `web/src/components/Navbar.jsx`, `mobile/lib/widgets/status_chip.dart` |
| Secure data and records management | Firebase login, token verification, role checks on every write, records kept on disk | `backend/src/Auth.kt`, `backend/src/Server.kt` |

---

## 9. Business rules already implemented

* A room cannot be booked twice for overlapping times (409 Conflict).
* A room under maintenance cannot be requested.
* End time must be after start time; dates and times are validated.
* Only an administrator can add, change or delete rooms and approve or reject requests.
* Only the owner (or an administrator) can cancel a request, and only while it is waiting.
* A room that still has waiting or approved requests cannot be deleted.
* Approving a request marks the room occupied; cancelling or rejecting frees it again.

---

## 10. Troubleshooting

| Problem | Fix |
|---------|-----|
| `Address already in use: bind` | Another program uses the port. Start with `set SRAMS_PORT=8091` before `run.bat` and change the same number in `web/src/config.js` and `mobile/lib/config.dart`. |
| Website says "Cannot reach the server" | The backend is not running. Start `backend\run.bat` first. |
| `'kotlinc' is not recognized` | Install the Kotlin compiler and add its `bin` folder to `PATH`. |
| Flutter: `Connection refused` | On the emulator use `10.0.2.2`, not `localhost`. On a phone use your PC's LAN address. |
| Flutter: `SocketException` on a phone | The phone and the PC must be on the same Wi-Fi, and Windows Firewall must allow port 8090. |
| Website login does nothing | Open the browser console (F12) - the message comes from the API. |

---

## 11. Where to add your own features

* **A new field on a room** - add it to `Room` in `backend/src/Models.kt`, to the
  `saveRooms`/`loadRooms` lists in `Store.kt`, to `roomJson` in `Server.kt`, then show it
  in `RoomsPage.jsx` and `rooms_tab.dart`.
* **A new endpoint** - add one `if` block in `route()` inside `backend/src/Server.kt`.
* **A database instead of text files** - only `backend/src/Store.kt` needs to change.
* **A mobile screen for administrators** - copy `rooms_tab.dart` and add buttons that
  call the same endpoints the website uses.

---

## 12. Folder overview

```
SRAMS/
├─ backend/                 Kotlin REST API (no libraries)
│  ├─ src/
│  │  ├─ Server.kt          all HTTP routes
│  │  ├─ Store.kt           data + text-file storage
│  │  ├─ Models.kt          Room, User, AllotmentRequest
│  │  ├─ Auth.kt            Firebase token check + demo mode
│  │  └─ MiniJson.kt        tiny JSON helper
│  ├─ build.bat / run.bat
│  └─ data/                 created on the first run
├─ web/                     React website
│  └─ src/
│     ├─ config.js          *** the file you edit ***
│     ├─ api.js             all REST calls
│     ├─ auth.js            Firebase login
│     ├─ App.jsx            screens and navigation
│     └─ pages/             Dashboard, Rooms, Requests, Login
├─ mobile/                  Flutter app
│  └─ lib/
│     ├─ config.dart        *** the file you edit ***
│     ├─ api.dart           all REST calls
│     ├─ auth_service.dart  Firebase login
│     └─ screens/           login, dashboard, rooms, requests
└─ docs/
   ├─ API.md                every endpoint with examples
   └─ FIREBASE_SETUP.md     step by step Firebase guide
```
