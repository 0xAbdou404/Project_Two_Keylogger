# System Architecture
this document breaks down how the system is designed and why certain architectural decisions were made.
## High Level Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    Operating System                      │
│                  (Keyboard Event Stream)                 │
└────────────────────────┬─────────────────────────────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │  pynput Listener     │
              │  (Event Callbacks)   │
              └──────────┬───────────┘
                         │
                         ▼
              ┌──────────────────────┐
              │     Keylogger        │
              │   (Main Controller)  │
              └─────┬────────┬───────┘
                    │        │
        ┌───────────┘        └──────────┐
        ▼                               ▼
┌───────────────┐              ┌────────────────┐
│ WindowTracker │              │  LogManager    │
│  (Platform-   │              │ (File Writing) │
│   Specific)   │              └────────┬───────┘
└───────────────┘                       │
                                        ▼
                              ┌──────────────────┐
                              │ WebhookDelivery  │
                              │  (Exfiltration)  │
                              └──────────────────┘
```
### Component Breakdown
#### Keylogger(Main Controller)
- *Purpose*: Orchestrates all componets and handles event processing pipeline
- *Responsibilities*: Receive keyboard events from pynput, processes keys, coordinates window tracking, delegates to logging and webhook delivery
- *Interfaces*: Exposes `start()` and `stop()` methods for lifecycle management, registers `_on_press()` callback with pynput listener
- *Location*: `keylogger.py:373-533`
#### LogManager
- *Purpose*: Manages persistent storage of keystroke events with automatic file rotation using direct file I/O
- *Responsibilites*: Creates timestamped log files, writes events to disk via raw file handles, monitors file size and rotates when limit reached, provides thread-safe access via locks, handles explicit file handle cleanup via `close()`
- *Interface*: `write_event(event)` for logging, `get_current_log_content()` for reading back logs, `close()` for releasing file handles
- *Location*: `keylogger.py:222-296`
#### WebhookDelivery
- *Purpose*: Handles remote exfiltration of captured keystrokes via HTTP webhooks
- *Responsibilities*: Buffer events to reduce network traffic, batches events before sending, uses a buffer swap pattern to deliver outside the lock, delivers JSON payload to configured endpoint, handles delivery failures gracefully
- *Interfaces*: `add_event(event)` for queuing, `flush()`for forcing immediate delivery
- *Location*: `keylogger.py:298-371`
#### WindowTracker
- *Purpose*: Determines which application has focus when keystrokes occur
- *Responsibilities*: Platform detection(Windows/macOS/Linux), calls platform-specific APIs to get active window title, provides unified interface across platform
*Interfaces*: Static method `get_active_window()` returns current window title or None
- *Location*: `keylogger.py:155-219`
#### KeyEvent(Data Model)
- *Purpose*: Immutable representation of a single keystroke with metadata
- *Responsibilities*: Stores timestamp, key value, window context, and key type classification
- *Interfaces*: `to_dict()` for JSON serialization, `to_log_string()` for human-readable formatting
- *Location*: `Keylogger.py:123-152`

## Data Flow:
### Primary Use Case: Keystroke Capture and Logging
Step by step walkthrough of what happens when a user presses a key:
```
1. OS Keyboard Event → pynput Listener
   User presses 'a' key, OS delivers envent to all registered pynput captures event and triggers our callback

2. Listener → Keylogger._on_press() (keylogger.py:428)
   Callback receives Key or KeyCode object
   Check if it's the toggle key (dynamically reads config toggle key) → pause/resume if so 
   Checks if logging is active → early return if paused

3. Keylogger → WindowTracker.get_active_window() (Keylogger.py:390)
   Calls platform-specific code to get active window
   Caches result for config.window_check_interval seconds (default 0.5) to avoid excessive API calls
   Returns window title like "Chrome - Gmail" or None

4. Keylogger → _process_key() (keylogger.py:406)
   Coverts Key/KeyCode to string representation
   Looks up special keys via module-level SPECIAL_KEYS constant
   Maps specail keys (Enter→"[Enter]", Space→"[Space]")
   Classsifies key type (CHAR, SPECIAL, UNKOWN)

5. Keylogger → Creates KeyEvent (keylogger.py:450)
   Bundles timestamp, key string, window title, and key type Creates dataclass instance

6. Keylogger → LogManager.write_event() (Keylogger.py: 248)
   Acquires lock for thread safety
   Formats event to log string: "[2025-01-31 14:30:22][Chrome] a"
   Writes to current log file via direct file I/O (self._file.write + flush)
   Checks file size and rotates if needed

7. Keylogger → WebhookDelivery.add_event() (keylogger.py:308)
   Add event to buffer array under lock
   Checks if buffer reached reached batch size (default 50)
   If full, swaps the buffer (replaces with empty list) under lock
   Delivers the batch OUTSIDE the lock via HTTP POST
   
```
Example with code references:
```
1. User types "p" → OS delivers KeyCode(char='p')

2. _on_press receives event (keylogger.py:428-458)
   Validates logging is active, not the toggle key

3. _update_active_window() called (keylogger.py:390-404)
   Returns "Visual Studio Code - keylogger.py"

4. _process_key(KeyCode(char='p')) → ("p", KeyType.CHAR)
   Not a special key, has .char attribute

5. KeyEvent created:
   timestamp=datetime.now()
   key="p"
   window_title="Visual Studio Code - keylogger.py"
   key_type=KeyType.CHAR

6. LogManager.write_event() (keylogger.py:248-255)
   Writes: "[2025-01-31 14:30:45][Visual Studio Code - keylogger.py] p"
   Checks: Current file is 4.2 MB, under 5 MB limit, no rotation

7. WebhookDelivery.add_event() (keylogger.py:308-324)
   Acquires lock, appends event, buffer now has 47 events
   Not yet at batch size 50, releases lock, no delivery
```
### Secondary Use Case: Log File Rotation
Step by step for when log file grows too large:
```
1. LogManager.write_event() → _check_rotation() (keylogger.py:257)
   After writing and flushing event, checks current log file size

2. _check_rotation() (keylogger.py:257-268)
   Tries to stat the file for its size
   If FileNotFoundError (file deleted externally), rotates immediately
   Otherwise reads file size: 5.1 MB (over 5 MB limit)
   Calls _rotate()

3. _rotate() (keylogger.py:270-280)
   Closes current file handle (self._file.close())
   Generates new path via _get_new_log_path()

4. _get_new_log_path() (keylogger.py:240-246)
   Generates new filename with current timestamp including microseconds
   Format: "keylog_20250131_143500_123456.txt"
   Returns Path object in log_dir

5. Opens new file handle
   self._file = open(new_path, 'a', encoding='utf-8')
   Ready for next write

6. Next write_event() call → Goes to new file
   Old file preserved with all historical keystrokes
```
