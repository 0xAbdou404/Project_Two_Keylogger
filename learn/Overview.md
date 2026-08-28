# Keylogger
## What This Is

A cross-platform keylogger that captures keyboard events, tracks active windows, and delivers logs remotely via webhooks, Build with Python using pynput for event capture, this demonstrates how malware monitors user activity and exxfiltrates data without detection

## Why This Matter

Keyloggers are one of the oldest and most effective attack vectors. They've been used in major breaches including the 2013 Target breach where attackers used keylogging malware on point-of-sale systems to steal 40 million credit cards.

## What You'll Learn

This project teaches you how keyboard capture malware works under the hood. By building it yourself, you'll understand:
#### Security Concepts:
- Keyboard event interception: HOw operating systems expose keyboard events to applications and why this creates a security boundary that's difficult to protect
- Data exfiltration patterns: The techniques malware uses to send stolen data to command-and-control server(C2 server), including batching to avoid detection
- Cross-platform malware development: Platform-specific APIs for Windows(win32gui), macOS(AppKit), and Linux(xdotool) that malware exploit
#### Technical Skills:
- Event-driven programming with callbacks that process keyboard input in real time
- Thread-safe logging using locks to prevent race conditions when multiple threads access shared resources
- Platform detection and conditional imports to create malware that adapts to different operating systems
- File rotation strategies to manage log sizes and avoid filling up disk space

#### Tools and Techniques:
- pynput library for low-level keyboard and mouse event capture
- Webhook delivery for remote data exfiltration over HTTPS
- Process and window tracking to correlate keystrokes with the applications they were typed into

---
