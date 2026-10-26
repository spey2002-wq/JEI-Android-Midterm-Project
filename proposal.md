# Android Midterm Project — Planning & Design Proposal

| Item | Details |
|------|---------|
| **App Name** | JEI Smart Campus Study Room Booking App |
| **Student ID** | 2025621055 |
| **Student Name** | RAI SPANDAN |
| **GitHub Repository URL** | https://github.com/[YourGitHubUsername]/JEI-Android-Midterm-Project |
| **Date** | 2026-10-26 |

---

## 1. Project Overview

### 1.1 One-Sentence Description

JEI Smart Campus Study Room Booking App is an Android application that lets JEI University students browse campus study rooms, check real-time availability, and reserve a time slot, while administrators and TAs can monitor room status.

### 1.2 Problem to Solve

Today, study rooms at JEI University are usually reserved by hand, for example by writing on a paper sign-up sheet, asking a librarian, or sending messages in group chats. This causes several problems:

- **Manual reservation is slow and error-prone.** Paper sheets get lost, handwriting is unclear, and staff must spend time managing them.
- **Scheduling conflicts.** Two groups can end up booking the same room at the same time because nobody can see the whole schedule in one place.
- **No real-time availability view.** Students must walk to a building to find out whether a room is free, which wastes time, especially during midterm and final exam periods.
- **No booking history.** Students cannot easily check, change, or cancel what they booked, so rooms stay "reserved" even when nobody uses them.

This app solves these problems by storing all rooms and reservations in a local Room (SQLite) database, automatically checking for time conflicts before saving a booking, and showing availability in a simple list.

### 1.3 Target Users & Use Cases

| User Type | Description | Main Use Cases |
|-----------|-------------|----------------|
| **JEI University Students** (primary) | Undergraduate and graduate students who need quiet or group study space | • Browse the list of study rooms<br>• Filter rooms by capacity and building<br>• View a room's equipment and today's booked time slots<br>• Book a room for a date and time<br>• View, cancel, or check out of own bookings |
| **TAs / Administrators** (secondary) | Teaching assistants and library/department staff who manage rooms | • Check which rooms are currently available or occupied<br>• View all reservations for a room on a given day<br>• Open or close a room for maintenance (`is_available` flag)<br>• Review booking history to resolve disputes |

**Example scenario:** Kim Minjun (Computer Software, 3rd year) opens the app the night before a midterm, filters for rooms that hold 4+ people, sees that "Study Room A-202" is free tomorrow from 14:00 to 16:00, and books it in under a minute.

---

## 2. Core Features

| # | Feature | User Action & Expected Result | Android Technology |
|---|---------|-------------------------------|--------------------|
| 1 | **Browse Study Room List** | The user opens the app and sees all study rooms as cards (name, building, capacity, availability badge). Tapping a card opens the room details. | `RecyclerView` + `ListAdapter` + `DiffUtil`, `RoomListFragment`, Room `StudyRoomDao` (`Flow`) |
| 2 | **View Room Details & Availability** | The user sees the room's building, floor, capacity, equipment, and a list of already-booked time slots for the selected date. A "Book This Room" button leads to the booking form. | `RoomDetailFragment`, Navigation Component with Safe Args, `RecyclerView` for time slots, Room query by `room_id` and date |
| 3 | **Book a Room (with Conflict Check)** | The user picks a date, start time, end time, and purpose, then taps "Submit". If the slot overlaps an existing booking, an error message is shown; otherwise the booking is saved and the user moves to "My Bookings". | `BookingApplyFragment`, `DatePickerDialog`, `TimePickerDialog`, input validation, Room `@Insert` + conflict `@Query`, `ViewModel` + Coroutines |
| 4 | **Manage My Bookings (View / Cancel / Check-out)** | The user sees a history of their bookings (status: CONFIRMED / CANCELLED / COMPLETED). Tapping "Cancel" or "Check-out" changes the status and updates the list immediately. | `MyBookingsFragment`, `RecyclerView` with item buttons, Room `@Update` / `@Query`, `AlertDialog` confirmation, `Flow` + `LiveData`/`StateFlow` |
| 5 | **Filter & Search Rooms** *(extra)* | The user types a keyword or selects a minimum capacity. The room list updates instantly. | `SearchView`, `ChipGroup`, Room `@Query` with `LIKE` |
| 6 | **Admin Room Open/Close** *(extra)* | An admin toggles a room's availability (e.g., for cleaning). Closed rooms show a "Closed" badge and cannot be booked. | Room `@Update`, `Switch`, role check on `users.role` |

### 2.1 Most Important Feature and Why

**Feature 3 — Book a Room (with Conflict Check)** is the most important feature.

- It is the **core purpose of the app**: without booking, the app is only a room catalog.
- It directly fixes the biggest problem, **scheduling conflicts**. The app runs an overlap query before every insert:
  `newStart < existingEnd AND newEnd > existingStart` (only against `CONFIRMED` bookings of the same room and date).
- It uses the most Android concepts together: Fragment, Navigation arguments, date/time pickers, form validation, Room `@Insert` and `@Query`, and Coroutines on a background thread.
- All other features depend on it. Room details show the bookings it creates, and My Bookings displays and manages them.

### 2.2 RecyclerView Usage Plan

The project uses `RecyclerView` in **three places**.

| RecyclerView | Screen | Data Source |
|--------------|--------|-------------|
| `RoomAdapter` | `RoomListFragment` | `study_rooms` table |
| `TimeSlotAdapter` | `RoomDetailFragment` | `reservations` for one room and one date |
| `BookingAdapter` | `MyBookingsFragment` | `reservations` joined with `study_rooms` for the current user |

**A. Data list details**

- **RoomListFragment:** list of `StudyRoom` objects ordered by building and name.
- **RoomDetailFragment:** list of booked time slots (`start_time – end_time`, purpose) for the selected day.
- **MyBookingsFragment:** list of `ReservationDetail` objects (reservation + room name + building), newest first.

**B. Layout cards (`item_room.xml`, `item_booking.xml`)**

Each item is a `MaterialCardView` with rounded corners and light elevation.

```
item_room.xml                                 item_booking.xml
+------------------------------------+        +------------------------------------+
| [icon]  Study Room A-202           |        | Study Room A-202  (Main Library)   |
|         Main Library, 2F           |        | 2026-10-28  14:00 - 16:00          |
|         Capacity: 8   [Available]  |        | Purpose: Group Study               |
+------------------------------------+        | Status: [CONFIRMED]                |
                                              |        [Cancel]    [Check-out]     |
                                              +------------------------------------+
```

- Room card: `ImageView` (room icon), `TextView` (name), `TextView` (building and floor), `TextView` (capacity), status badge (green "Available" / grey "Closed").
- Booking card: `TextView`s (room, date/time, purpose), a status chip with a colour per status (CONFIRMED = blue, COMPLETED = green, CANCELLED = grey), and two `Button`s (Cancel, Check-out) shown only for `CONFIRMED` bookings.

**C. Adapter logic**

- Use `ListAdapter<T, ViewHolder>` with a `DiffUtil.ItemCallback` (compare items by `roomId` / `reservationId`), so list updates are efficient and animated.
- Use **ViewBinding** in each `ViewHolder` (no `findViewById`).
- Pass **lambda callbacks** from the Fragment into the adapter, for example `RoomAdapter(onRoomClick: (StudyRoom) -> Unit)` and `BookingAdapter(onCancel: (ReservationDetail) -> Unit, onCheckOut: (ReservationDetail) -> Unit)`. The Fragment handles navigation and database calls, so the adapter stays simple.
- In `onBindViewHolder`, set text and colours from the data and hide the buttons when the status is not `CONFIRMED`.
- The Fragment collects the `Flow` from the Room DAO via the `ViewModel`, then calls `adapter.submitList(list)`. This updates the screen automatically when the database changes.

---

## 3. Screen Wireframes

### 3.1 RoomListFragment

**Purpose:** Home screen. Lets the user browse all study rooms, see availability at a glance, and search/filter.

**Text UI Wireframe:**

```
+--------------------------------------------+
|  JEI Study Rooms                    [ ⚙ ]  |  <- Toolbar
+--------------------------------------------+
|  [ 🔍 Search room or building...        ]  |  <- SearchView
|  (All) (2+ people) (4+ people) (8+ people) |  <- ChipGroup filter
+--------------------------------------------+
|  +--------------------------------------+  |
|  | [🏫] Study Room A-101                |  |
|  |      Main Library, 1F                |  |
|  |      Capacity: 4        [Available]  |  |
|  +--------------------------------------+  |
|  +--------------------------------------+  |
|  | [🏫] Study Room A-202                |  |
|  |      Main Library, 2F                |  |
|  |      Capacity: 8        [Available]  |  |  <- RecyclerView
|  +--------------------------------------+  |
|  +--------------------------------------+  |
|  | [🏫] Quiet Room C-001                |  |
|  |      Student Center, 0F              |  |
|  |      Capacity: 2           [Closed]  |  |
|  +--------------------------------------+  |
+--------------------------------------------+
|   [ Rooms ]               [ My Bookings ]  |  <- BottomNavigationView
+--------------------------------------------+
```

**Main UI Elements:**

| Element | Type | Role |
|---------|------|------|
| Toolbar | `MaterialToolbar` | App title |
| Search box | `SearchView` | Search by room name or building |
| Filter chips | `ChipGroup` | Filter by minimum capacity |
| Room list | `RecyclerView` + `RoomAdapter` | Shows room cards |
| Room card | `MaterialCardView` | Name, building/floor, capacity, status badge |
| Bottom navigation | `BottomNavigationView` | Switch between Rooms and My Bookings |

---

### 3.2 RoomDetailFragment

**Purpose:** Shows full information about one room and the time slots already booked on a selected date, so the user knows when it is free.

**Text UI Wireframe:**

```
+--------------------------------------------+
|  [←]  Study Room A-202                     |  <- Toolbar (back arrow)
+--------------------------------------------+
|  Building : Main Library                   |
|  Floor    : 2F                             |
|  Capacity : 8 people                       |
|  Equipment: Projector, Whiteboard, Wi-Fi   |
|  Status   : [Available]                    |
+--------------------------------------------+
|  Date:  [ < ]  2026-10-28 (Wed)  [ > ]     |  <- Date selector
+--------------------------------------------+
|  Booked time slots                         |
|  +--------------------------------------+  |
|  | 10:00 - 12:00   Midterm Prep         |  |
|  +--------------------------------------+  |  <- RecyclerView
|  | 14:00 - 16:00   Group Study          |  |
|  +--------------------------------------+  |
|  (No other bookings - rest of day free)    |
+--------------------------------------------+
|        [      Book This Room      ]        |  <- Button
+--------------------------------------------+
```

**Main UI Elements:**

| Element | Type | Role |
|---------|------|------|
| Toolbar with back arrow | `MaterialToolbar` | Return to list |
| Room info labels | `TextView` ×5 | Building, floor, capacity, equipment, status |
| Date selector | `ImageButton` ×2 + `TextView` | Change the date being viewed |
| Time-slot list | `RecyclerView` + `TimeSlotAdapter` | Booked slots for that date |
| Empty message | `TextView` | Shown when no bookings exist |
| Book button | `MaterialButton` | Go to booking form (disabled if room is closed) |

---

### 3.3 BookingApplyFragment

**Purpose:** Lets the user choose date, start/end time, and purpose, then submit the reservation. Validates input and checks for conflicts.

**Text UI Wireframe:**

```
+--------------------------------------------+
|  [←]  Book a Room                          |
+--------------------------------------------+
|  Room: Study Room A-202 (Main Library)     |
|                                            |
|  Date                                      |
|  [ 2026-10-28                    📅 ]      |  <- DatePickerDialog
|                                            |
|  Start time          End time              |
|  [ 14:00   🕑 ]      [ 16:00   🕑 ]        |  <- TimePickerDialog
|                                            |
|  Purpose                                   |
|  ( ) Group Study  ( ) Presentation         |  <- RadioGroup / dropdown
|  ( ) Self Study   ( ) Other                |
|                                            |
|  ⚠ This time slot is already booked!       |  <- Error text (hidden by default)
|                                            |
|  [ Cancel ]            [   Submit   ]      |
+--------------------------------------------+
```

**Main UI Elements:**

| Element | Type | Role |
|---------|------|------|
| Room summary | `TextView` | Shows the selected room (from Safe Args) |
| Date field | `TextInputEditText` + `DatePickerDialog` | Choose reservation date |
| Start / End time | `TextInputEditText` ×2 + `TimePickerDialog` | Choose time range |
| Purpose | `RadioGroup` or `AutoCompleteTextView` | Reason for booking |
| Error message | `TextView` | Shows validation or conflict errors |
| Cancel button | `MaterialButton` (outlined) | Go back without saving |
| Submit button | `MaterialButton` | Validate, check conflicts, save |

**Validation rules:** the date must not be in the past; the end time must be after the start time; the maximum booking length is 3 hours; no overlap with another `CONFIRMED` booking.

---

### 3.4 MyBookingsFragment

**Purpose:** Shows the logged-in user's booking history and allows cancelling an upcoming booking or checking out after use.

**Text UI Wireframe:**

```
+--------------------------------------------+
|  My Bookings                               |
+--------------------------------------------+
|  (All) (Confirmed) (Completed) (Cancelled) |  <- ChipGroup filter
+--------------------------------------------+
|  +--------------------------------------+  |
|  | Study Room A-202  (Main Library)     |  |
|  | 2026-10-28   14:00 - 16:00           |  |
|  | Purpose: Group Study                 |  |
|  | Status: [CONFIRMED]                  |  |
|  |         [ Cancel ]   [ Check-out ]   |  |
|  +--------------------------------------+  |  <- RecyclerView
|  +--------------------------------------+  |
|  | Study Room A-101  (Main Library)     |  |
|  | 2026-10-20   09:00 - 11:00           |  |
|  | Purpose: Self Study                  |  |
|  | Status: [COMPLETED]                  |  |
|  +--------------------------------------+  |
+--------------------------------------------+
|   [ Rooms ]               [ My Bookings ]  |
+--------------------------------------------+
```

**Main UI Elements:**

| Element | Type | Role |
|---------|------|------|
| Filter chips | `ChipGroup` | Filter by booking status |
| Booking list | `RecyclerView` + `BookingAdapter` | Shows booking cards |
| Status chip | `Chip` | Colour-coded status |
| Cancel button | `MaterialButton` | Sets status to `CANCELLED` (after confirm dialog) |
| Check-out button | `MaterialButton` | Sets status to `COMPLETED` |
| Empty state | `TextView` | "No bookings yet" message |
| Bottom navigation | `BottomNavigationView` | Switch screens |

---

## 4. Navigation Flow

The app uses a **single Activity** (`MainActivity`) with a `NavHostFragment` and one navigation graph (`nav_graph.xml`). `RoomListFragment` is the start destination. Data is passed between screens with **Safe Args**.

**Navigation graph (text diagram):**

```
                       +-------------------+
        (start)  ----> |  RoomListFragment |  <------------------------+
                       +-------------------+                           |
                         |             ^                               |
        tap room card    |             | Back / Up arrow               |
        (roomId)         v             |                               |
                       +--------------------+                          |
                       | RoomDetailFragment |                          |
                       +--------------------+                          |
                         |             ^                               |
   tap "Book This Room"  |             | Back / Cancel                 |
   (roomId, date)        v             |                               |
                       +----------------------+                        |
                       | BookingApplyFragment |                        |
                       +----------------------+                        |
                         |                                             |
   tap "Submit" (valid,  |                                             |
   no conflict)          v                                             |
                       +--------------------+   bottom nav "Rooms"     |
                       |  MyBookingsFragment| -------------------------+
                       +--------------------+
                         ^
                         | bottom nav "My Bookings" (from RoomListFragment)
```

**Transition triggers:**

| # | From | To | Trigger | Data Passed (Safe Args) | Back Stack Behaviour |
|---|------|----|---------|-------------------------|----------------------|
| 1 | `RoomListFragment` | `RoomDetailFragment` | User taps a room card in the RecyclerView | `roomId: Int` | Added; Back returns to list |
| 2 | `RoomDetailFragment` | `BookingApplyFragment` | User taps "Book This Room" (enabled only if room is available) | `roomId: Int`, `selectedDate: String` | Added; Back returns to room details |
| 3 | `BookingApplyFragment` | `MyBookingsFragment` | User taps "Submit" and validation + conflict check pass; a success Snackbar is shown | none (the new booking is read from the database) | Booking form and room detail are popped (`popUpTo = RoomListFragment`), so Back from My Bookings returns to the room list |
| 4 | `BookingApplyFragment` | `RoomDetailFragment` | User taps "Cancel" or the Back arrow | none | Pops the form |
| 5 | `RoomListFragment` ⇄ `MyBookingsFragment` | each other | User taps a bottom-navigation item | none | Top-level destinations; the stack is not stacked repeatedly |
| 6 | `MyBookingsFragment` | `RoomListFragment` | User taps the "Rooms" tab, or the Back button | none | Returns to start destination |

**Navigation notes**

- Safe Args plugin: `androidx.navigation.safeargs.kotlin`.
- The current user's `userId` is kept in a `SharedPreferences` session (simple login or a default demo user), so it does not need to be passed on every screen.
- Error cases (conflict, invalid time) keep the user on `BookingApplyFragment` and show an error message instead of navigating.

---

## 5. Room SQLite Database Design

**Database name:** `campus_booking_database` (Room / SQLite, version 1)

**Relationships:** `users` 1 — N `reservations` N — 1 `study_rooms`

### 5.1 Table: `users`

| Column | Type | Key / Constraint | Description |
|--------|------|------------------|-------------|
| `user_id` | INTEGER | PK, AUTOINCREMENT, NOT NULL | Unique user ID |
| `student_number` | TEXT | NOT NULL, UNIQUE | University student/staff number |
| `full_name` | TEXT | NOT NULL | Full name |
| `email` | TEXT | NOT NULL, UNIQUE | University email (login ID) |
| `password_hash` | TEXT | NOT NULL | Hashed password |
| `department` | TEXT | NOT NULL | Department / major |
| `role` | TEXT | NOT NULL, DEFAULT 'STUDENT' | `STUDENT` or `ADMIN` |
| `created_at` | TEXT | NOT NULL | Account creation time (`yyyy-MM-dd HH:mm:ss`) |

**Example record:**

| user_id | student_number | full_name | email | password_hash | department | role | created_at |
|---------|----------------|-----------|-------|---------------|------------|------|------------|
| 1 | 20231001 | Kim Minjun | minjun@jei.ac.kr | a1b2c3... | Computer Software | STUDENT | 2026-03-02 09:15:00 |

### 5.2 Table: `study_rooms`

| Column | Type | Key / Constraint | Description |
|--------|------|------------------|-------------|
| `room_id` | INTEGER | PK, AUTOINCREMENT, NOT NULL | Unique room ID |
| `room_name` | TEXT | NOT NULL, UNIQUE | Display name |
| `building` | TEXT | NOT NULL | Building name |
| `floor` | INTEGER | NOT NULL | Floor number |
| `capacity` | INTEGER | NOT NULL | Maximum number of people |
| `equipment` | TEXT | NOT NULL, DEFAULT '' | Comma-separated equipment list |
| `is_available` | INTEGER | NOT NULL, DEFAULT 1 | `1` = open for booking, `0` = closed |

**Example record:**

| room_id | room_name | building | floor | capacity | equipment | is_available |
|---------|-----------|----------|-------|----------|-----------|--------------|
| 2 | Study Room A-202 | Main Library | 2 | 8 | Projector,Whiteboard,Wi-Fi | 1 |

### 5.3 Table: `reservations`

| Column | Type | Key / Constraint | Description |
|--------|------|------------------|-------------|
| `reservation_id` | INTEGER | PK, AUTOINCREMENT, NOT NULL | Unique reservation ID |
| `user_id` | INTEGER | FK → `users.user_id` (ON DELETE CASCADE), NOT NULL | User who booked |
| `room_id` | INTEGER | FK → `study_rooms.room_id` (ON DELETE CASCADE), NOT NULL | Room being booked |
| `reservation_date` | TEXT | NOT NULL | Date of use (`yyyy-MM-dd`) |
| `start_time` | TEXT | NOT NULL | Start time (`HH:mm`) |
| `end_time` | TEXT | NOT NULL | End time (`HH:mm`) |
| `purpose` | TEXT | NOT NULL | Reason for booking |
| `status` | TEXT | NOT NULL, DEFAULT 'CONFIRMED' | `CONFIRMED`, `CANCELLED`, or `COMPLETED` |
| `created_at` | TEXT | NOT NULL | When the booking was made |

**Example record:**

| reservation_id | user_id | room_id | reservation_date | start_time | end_time | purpose | status | created_at |
|----------------|---------|---------|------------------|------------|----------|---------|--------|------------|
| 1 | 1 | 2 | 2026-10-28 | 14:00 | 16:00 | Group Study | CONFIRMED | 2026-10-26 09:30:00 |

> The full Room Entity and DAO draft code is in `schema_notes.txt`.

---

## 6. Room & Data Operations

### 6.1 Component Responsibilities

| Component | Class Name(s) | Responsibility |
|-----------|---------------|----------------|
| **Entity** | `User`, `StudyRoom`, `Reservation` | Kotlin data classes annotated with `@Entity`; each one maps to a table. `Reservation` declares two `@ForeignKey` links. |
| **DAO** | `UserDao`, `StudyRoomDao`, `ReservationDao` | Interfaces with `@Insert`, `@Update`, `@Delete`, and `@Query` methods. They define every database operation, including the conflict-check query. |
| **AppDatabase** | `AppDatabase` | Abstract `RoomDatabase` annotated with `@Database(entities = [...], version = 1)`. It is a singleton and exposes the DAOs. Can pre-load sample rooms. |
| **Repository** | `RoomRepository`, `BookingRepository` | A single data access layer between the UI and the DAOs. Contains business logic (e.g., "check conflict, then insert") and hides Room from the UI. |
| **ViewModel** | `RoomListViewModel`, `BookingViewModel` | Holds UI state, calls the Repository in `viewModelScope`, and exposes `Flow`/`LiveData`. Survives screen rotation. |
| **Fragment** | `RoomListFragment`, `RoomDetailFragment`, `BookingApplyFragment`, `MyBookingsFragment` | Displays data, handles clicks, observes the ViewModel, and navigates. Contains no SQL or database code. |

**Layered structure:**

```
Fragment  ->  ViewModel  ->  Repository  ->  DAO  ->  AppDatabase (SQLite)
 (UI)        (UI state)     (logic)        (queries)   (campus_booking_database)
```

### 6.2 Data Processing Flow

**Flow A — Display the room list (read)**

1. `RoomListFragment` starts and asks `RoomListViewModel` for the rooms.
2. The ViewModel calls `RoomRepository.getAllRooms()`.
3. The Repository calls `StudyRoomDao.getAllRooms()`, which runs `SELECT * FROM study_rooms ...` and returns a `Flow<List<StudyRoom>>`.
4. The Fragment collects the `Flow` (`repeatOnLifecycle(STARTED)`) and calls `roomAdapter.submitList(rooms)`.
5. `DiffUtil` compares the lists, and `RecyclerView` redraws only the changed cards.

**Flow B — Create a booking (write with conflict check)**

1. The user fills in the form in `BookingApplyFragment` and taps **Submit**.
2. The Fragment validates the input (date not in the past, end after start, max 3 hours, purpose chosen). On failure, an error message is shown.
3. The Fragment calls `BookingViewModel.submitBooking(...)`.
4. The ViewModel launches a coroutine in `viewModelScope` (background thread) and calls `BookingRepository.createBooking(...)`.
5. The Repository calls `ReservationDao.countConflicts(roomId, date, start, end)`.
6. **If the result is greater than 0:** return `Result.Conflict`, and the Fragment shows "This time slot is already booked!".
7. **If the result is 0:** the Repository builds a `Reservation` (status `CONFIRMED`, `created_at` = now) and calls `ReservationDao.insertReservation(...)`.
8. Room writes the row to SQLite and returns the new ID. The Repository returns `Result.Success`.
9. The Fragment shows a Snackbar and navigates to `MyBookingsFragment`.

**Flow C — Cancel or check out (update)**

1. In `MyBookingsFragment`, the user taps **Cancel** (an `AlertDialog` asks for confirmation) or **Check-out**.
2. The adapter's lambda tells the Fragment, which calls `BookingViewModel.cancel(id)` or `checkOut(id)`.
3. The Repository calls `ReservationDao.updateStatus(id, "CANCELLED" / "COMPLETED")`.
4. Because the list is a `Flow`, Room emits the new list automatically, `submitList()` is called again, and the card updates (the buttons disappear).

**Flow D — View a room's booked slots (read)**

1. `RoomDetailFragment` receives `roomId` from Safe Args.
2. The ViewModel loads the room with `getRoomById(roomId)` and observes `getReservationsByRoomAndDate(roomId, date)`.
3. When the user changes the date, the query is run again, and `TimeSlotAdapter` shows the new list.

---

## 7. Development Schedule

| Weeks | Phase | Planned Tasks | Deliverable |
|-------|-------|---------------|-------------|
| **Weeks 9–10** | UI / XML Layouts | • Create the Android Studio project (Kotlin, ViewBinding) and GitHub repository<br>• Design the colour theme and styles (Material Components)<br>• Build XML layouts: `fragment_room_list`, `fragment_room_detail`, `fragment_booking_apply`, `fragment_my_bookings`<br>• Build item layouts: `item_room`, `item_time_slot`, `item_booking`<br>• Use hard-coded dummy data to check the look | All 4 screens and 3 item cards display correctly with dummy data |
| **Weeks 11–12** | Navigation & RecyclerView | • Create `nav_graph.xml` with 4 destinations and the Safe Args plugin<br>• Add `BottomNavigationView`<br>• Write `RoomAdapter`, `TimeSlotAdapter`, `BookingAdapter` (`ListAdapter` + `DiffUtil`)<br>• Connect click events (card → detail → booking form)<br>• Add date and time pickers and form validation | Full screen-to-screen navigation works with in-memory sample data |
| **Weeks 13–14** | Room DB Integration | • Add Room dependencies (KSP)<br>• Implement Entities, DAOs, `AppDatabase` (from `schema_notes.txt`)<br>• Implement Repository and ViewModel classes<br>• Pre-load sample rooms and a demo user<br>• Implement conflict check, cancel, and check-out<br>• Replace dummy data with database data | Bookings are saved and stay after the app restarts; conflicts are blocked |
| **Weeks 15–16** | Testing, GitHub Upload, Demo | • Test the main scenarios and edge cases (overlap, past date, closed room, rotation)<br>• Fix bugs and clean up the UI<br>• Write the README and update the proposal<br>• Commit and push the final code to GitHub (with clear commit messages)<br>• Prepare the demo (APK build, screen recording, presentation) | Final GitHub repository, working APK, and demo presentation |

**Risk notes:** Room and the coroutine data flow are new to me, so I will start the database work early (Week 13) and keep a few days of buffer in Weeks 15–16 for bug fixing.

---

## 8. Final Check & AI Disclosure

### 8.1 Final Checklist

- [x] App name, student ID, student name, GitHub URL, and date are filled in the header
- [x] Project overview completed (one-sentence description, problem, target users and use cases)
- [x] At least 4 core features defined in a table with Android technologies
- [x] Most important feature identified and explained
- [x] RecyclerView usage plan written (data, card layout, adapter logic)
- [x] 4 screen wireframes designed (`RoomListFragment`, `RoomDetailFragment`, `BookingApplyFragment`, `MyBookingsFragment`)
- [x] Navigation flow described with transition triggers
- [x] Room database designed with 3 tables, including keys, constraints, and example records
- [x] Room components and data processing flow explained
- [x] Development schedule for Weeks 9–16 prepared
- [x] AI usage statement included
- [x] Student ID and student name filled in with real information
- [ ] Replace `[YourGitHubUsername]` in the GitHub URL with the real username before submitting
- [ ] Push the proposal and the project code to the GitHub repository

### 8.2 AI Usage Statement

> Cursor AI was used to assist in structuring the project proposal and drafting the initial Room DB schema. All content was thoroughly reviewed, edited, and validated by the student.
