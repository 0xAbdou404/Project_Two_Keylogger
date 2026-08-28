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

