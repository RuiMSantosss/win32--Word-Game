# Win32 Distributed Word Puzzle System

## Overview
This project is a **multi-process distributed game engine** developed in **C** utilizing the **WIN32 API**. It serves as a practical implementation of advanced Operating System concepts, specifically focusing on **Inter-Process Communication (IPC)**, **concurrency**, and **memory synchronization**.

The system simulates a real-time multiplayer environment where human users and automated bots compete to identify hidden words as letters are periodically revealed by a central server.

## System Architecture
The solution is composed of five distinct executables/modules that operate asynchronously, communicating via **Named Pipes** and **Shared Memory**.

### 1. The Referee (*Arbitro*)
Acting as the **central server**, the Referee orchestrates the entire game session. Its responsibilities include:
* **Session Management:** Handles player connections and enforces lobby limits using **semaphores**.
* **IPC Handling:** Manages bidirectional **Named Pipe** connections with all active clients.
* **Multithreading:** Spawns dedicated threads for:
    * Client request handling (`playerAttend`).
    * Game logic and timing (letter revelation via `wordsTimer`).
    * Admin command processing.
* **State Synchronization:** Updates the **Shared Memory** block with the current game state (visible letters, scores) and signals clients via **Events**.
* **Thread Safety:** Protects the global state (`appContext`) using **Critical Sections** and **Mutexes**.

### 2. The Player Client (*Jogoui*)
A console-based client interface for human participants. Upon execution:
* It performs a handshake with the Referee via Named Pipes.
* It launches two primary threads:
    1.  **Listener Thread:** Waits for incoming server messages.
    2.  **Update Thread:** Reads from Shared Memory whenever a synchronization **Event** is triggered.
* **User Interaction:** The main thread captures standard input commands for guessing words, checking the leaderboard (`:pont`), or exiting (`:sair`).

### 3. The Bot (*Bot*)
An autonomous agent designed to simulate player activity.
* It shares the core architectural logic of the human client.
* Instead of processing user input, it utilizes a randomized algorithm to generate word guesses at variable intervals.
* The server treats Bots and Humans identically within the protocol.

### 4. The Dashboard (*Painel*)
A **Win32 GUI** application designed for passive monitoring.
* **Read-Only Access:** It accesses Shared Memory to visualize the game state but does not write to it.
* **Real-Time Visualization:** Displays the currently revealed letters, the last correct guess, and the top players.
* **Event-Driven Rendering:** A background thread waits for server signals to trigger a window repaint, ensuring the UI is always in sync with the backend without polling.

### 5. Core Library (*Funcoes*)
A static library (`.lib`) containing reusable logic, data structures, and protocol definitions shared between the Clients and the Bots to ensure consistency.

---

## Technical Implementation

### Technologies Used
* **Language:** C (Native WIN32 API).
* **Concurrency:** Extensive use of Threads for non-blocking operations.
* **Communication:** Named Pipes (Client-Server messaging) and Shared Memory (Broadcasting game state).
* **Synchronization:**
    * **Mutexes:** Preventing race conditions on resources.
    * **Semaphores:** Managing player capacity.
    * **Events:** Signaling state changes to wake up dormant threads.
    * **Critical Sections:** Lightweight protection for in-process global structures.

### Communication Protocol
To ensure robust and extensible communication, the system employs a **Two-Step Protocol** over Named Pipes:

1.  **Header:** The process sends a `COMMAND_TYPE` enum (integer).
2.  **Payload:** The process sends the specific `struct` associated with that command.

Lifecycle & Safety
The system implements Graceful Shutdown procedures:

Player Exit: Sending :sair triggers an EXITGAME request. The server acknowledges this, allowing the client to unblock any pending ReadFile operations and close handles cleanly.

Server Shutdown: If the administrator terminates the session, an EXPELLED broadcast is sent to all clients, forcing them to detach from resources and close instantly.

Academic Context
This project was developed for the "Operating Systems II" course (2024/2025) by a group of 2 students.

Final Grade: 9.2 / 10

**Command Types:**
```c
typedef enum {
  REGISTER,   // Handshake
  PONT,       // Score request
  JOGS,       // Player list request
  EXITGAME,   // Graceful disconnect
  WORDGUESS,  // Submission
  JOIN/LEFT,  // Notifications
  GUESSED,    // Success notification
  LEADER,     // Leaderboard update
  EXPELLED    // Force disconnect
} COMMAND_TYPE;

