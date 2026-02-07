# OpenClaw Notifications - Implementation Plan

## Executive Summary

This document outlines the implementation plan for adding **push notifications** and **proactive AI outreach** to OpenClaw. The system will allow the DGX Spark backend to proactively reach out to users, creating a more engaging AI assistant experience.

---

## Table of Contents

1. [Goals & Non-Goals](#goals--non-goals)
2. [Architecture Overview](#architecture-overview)
3. [Phase 1: iOS APNs Foundation](#phase-1-ios-apns-foundation)
4. [Phase 2: DGX Spark Notification Service](#phase-2-dgx-spark-notification-service)
5. [Phase 3: Device Registration Flow](#phase-3-device-registration-flow)
6. [Phase 4: Proactive Outreach Engine](#phase-4-proactive-outreach-engine)
7. [Phase 5: Deep Linking & Actions](#phase-5-deep-linking--actions)
8. [Phase 6: AI-Initiated Outreach](#phase-6-ai-initiated-outreach)
9. [Security Considerations](#security-considerations)
10. [Testing Strategy](#testing-strategy)
11. [File Changes Summary](#file-changes-summary)

---

## Goals & Non-Goals

### Goals

- **Push Notifications**: Alert users when app is backgrounded/closed
- **Proactive Outreach**: OpenClaw initiates contact based on schedules, events, or AI decisions
- **Deep Linking**: Tapping a notification opens the relevant conversation/context
- **Actionable Notifications**: Reply directly from notification banner
- **User Control**: Preferences for notification types, quiet hours, frequency

### Non-Goals (v1.0)

- Rich media notifications (images, video) — future enhancement
- Notification grouping/threading — future enhancement
- Critical alerts (bypassing Do Not Disturb) — not needed
- Location-based triggers — not in scope

---

## Architecture Overview

### System Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           DGX Spark                                     │
│                                                                         │
│  ┌─────────────────┐    ┌─────────────────────┐    ┌─────────────────┐  │
│  │    OpenClaw     │───►│  Notification       │───►│   APNs Gateway  │  │
│  │     (LLM)       │    │  Service (FastAPI)  │    │   (HTTP/2+JWT)  │  │
│  └─────────────────┘    └─────────────────────┘    └────────┬────────┘  │
│          │                        ▲                         │           │
│          │                        │                         │           │
│  ┌───────▼─────────┐    ┌────────┴────────┐                │           │
│  │  Tool: send_    │    │ Device Registry │                │           │
│  │  notification   │    │   (SQLite)      │                │           │
│  └─────────────────┘    └─────────────────┘                │           │
│                                                             │           │
│  ┌─────────────────────────────────────────┐               │           │
│  │         Outreach Scheduler              │               │           │
│  │  • Morning briefing (cron)              │───────────────┘           │
│  │  • Task completion (event)              │                           │
│  │  • AI-initiated (LLM decision)          │                           │
│  └─────────────────────────────────────────┘                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ HTTPS (Tailscale Funnel)
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Apple Push Notification Service                      │
│                     api.push.apple.com (HTTP/2)                         │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ APNs Protocol
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         iOS Device                                      │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      OpenClaw App                                │   │
│  │                                                                  │   │
│  │  ┌──────────────────┐  ┌──────────────────┐  ┌───────────────┐  │   │
│  │  │ PushNotification │  │ DeviceRegistration│  │ Notification  │  │   │
│  │  │    Manager       │  │    Service       │  │   Delegate    │  │   │
│  │  └──────────────────┘  └──────────────────┘  └───────────────┘  │   │
│  │                                                                  │   │
│  │  ┌──────────────────────────────────────────────────────────┐   │   │
│  │  │                    AppState                               │   │   │
│  │  │  + notificationPermission: PermissionStatus              │   │   │
│  │  │  + pendingAction: DeepLinkAction?                        │   │   │
│  │  └──────────────────────────────────────────────────────────┘   │   │
│  │                                                                  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Phase 1: iOS APNs Foundation

### Prerequisites

1. **Apple Developer Account** with Push Notifications capability
2. **APNs Authentication Key** (.p8 file) from Apple Developer Portal
3. **App ID** with Push Notifications enabled

### Step 1.1: Generate APNs Key

1. Go to [Apple Developer Portal](https://developer.apple.com/account/resources/authkeys/list)
2. Click **Keys** → **Create a key**
3. Name: `OpenClaw APNs Key`
4. Check **Apple Push Notifications service (APNs)**
5. Click **Continue** → **Register**
6. **Download the .p8 file** (you can only download once!)
7. Note the **Key ID** (e.g., `ABC123DEFG`)
8. Note your **Team ID** (visible in Membership details)

### Step 1.2: Enable Push Notifications in Xcode

1. Select **OpenClaw** target
2. Go to **Signing & Capabilities**
3. Click **+ Capability**
4. Add **Push Notifications**
5. Add **Background Modes** → Check **Remote notifications**

### Step 1.3: Create New iOS Files

#### File: `OpenClaw/Services/PushNotificationManager.swift`

```swift
//
//  PushNotificationManager.swift
//  OpenClaw
//
//  Manages APNs registration and notification permissions
//

import Foundation
import UserNotifications
import UIKit

enum NotificationPermissionStatus: Equatable {
    case notDetermined
    case authorized
    case denied
    case provisional
}

@MainActor
final class PushNotificationManager: NSObject, ObservableObject {
    static let shared = PushNotificationManager()

    @Published private(set) var permissionStatus: NotificationPermissionStatus = .notDetermined
    @Published private(set) var deviceToken: String?
    @Published private(set) var registrationError: String?

    private override init() {
        super.init()
    }

    // MARK: - Permission Handling

    func checkPermissionStatus() async {
        let settings = await UNUserNotificationCenter.current().notificationSettings()
        await MainActor.run {
            switch settings.authorizationStatus {
            case .notDetermined:
                permissionStatus = .notDetermined
            case .authorized:
                permissionStatus = .authorized
            case .denied:
                permissionStatus = .denied
            case .provisional:
                permissionStatus = .provisional
            case .ephemeral:
                permissionStatus = .authorized
            @unknown default:
                permissionStatus = .notDetermined
            }
        }
    }

    func requestPermission() async -> Bool {
        let center = UNUserNotificationCenter.current()

        do {
            let granted = try await center.requestAuthorization(
                options: [.alert, .sound, .badge, .provisional]
            )

            await MainActor.run {
                permissionStatus = granted ? .authorized : .denied
            }

            if granted {
                await registerForRemoteNotifications()
                registerNotificationCategories()
            }

            return granted
        } catch {
            print("[PushNotification] Permission request error: \(error)")
            return false
        }
    }

    // MARK: - APNs Registration

    private func registerForRemoteNotifications() async {
        await MainActor.run {
            UIApplication.shared.registerForRemoteNotifications()
        }
    }

    func handleDeviceToken(_ tokenData: Data) {
        let token = tokenData.map { String(format: "%02x", $0) }.joined()
        self.deviceToken = token
        self.registrationError = nil
        print("[PushNotification] Device token: \(token.prefix(16))...")

        // Register with backend
        Task {
            await DeviceRegistrationService.shared.register(token: token)
        }
    }

    func handleRegistrationError(_ error: Error) {
        self.registrationError = error.localizedDescription
        print("[PushNotification] Registration error: \(error)")
    }

    // MARK: - Notification Categories

    private func registerNotificationCategories() {
        // Reply action - allows text input directly from notification
        let replyAction = UNTextInputNotificationAction(
            identifier: "REPLY_ACTION",
            title: "Reply",
            options: [],
            textInputButtonTitle: "Send",
            textInputPlaceholder: "Type a message..."
        )

        // Start conversation action
        let startChatAction = UNNotificationAction(
            identifier: "START_CHAT_ACTION",
            title: "Start Chat",
            options: [.foreground]
        )

        // Snooze action
        let snoozeAction = UNNotificationAction(
            identifier: "SNOOZE_ACTION",
            title: "Snooze 1 hour",
            options: []
        )

        // Dismiss action
        let dismissAction = UNNotificationAction(
            identifier: "DISMISS_ACTION",
            title: "Dismiss",
            options: [.destructive]
        )

        // Categories
        let messageCategory = UNNotificationCategory(
            identifier: "OPENCLAW_MESSAGE",
            actions: [replyAction, startChatAction, dismissAction],
            intentIdentifiers: [],
            options: [.customDismissAction]
        )

        let briefingCategory = UNNotificationCategory(
            identifier: "OPENCLAW_BRIEFING",
            actions: [startChatAction, snoozeAction],
            intentIdentifiers: [],
            options: []
        )

        let reminderCategory = UNNotificationCategory(
            identifier: "OPENCLAW_REMINDER",
            actions: [startChatAction, snoozeAction, dismissAction],
            intentIdentifiers: [],
            options: []
        )

        let alertCategory = UNNotificationCategory(
            identifier: "OPENCLAW_ALERT",
            actions: [startChatAction],
            intentIdentifiers: [],
            options: []
        )

        UNUserNotificationCenter.current().setNotificationCategories([
            messageCategory,
            briefingCategory,
            reminderCategory,
            alertCategory
        ])
    }

    // MARK: - Badge Management

    func clearBadge() async {
        await MainActor.run {
            UIApplication.shared.applicationIconBadgeNumber = 0
        }
        try? await UNUserNotificationCenter.current().setBadgeCount(0)
    }
}
```

#### File: `OpenClaw/Services/DeviceRegistrationService.swift`

```swift
//
//  DeviceRegistrationService.swift
//  OpenClaw
//
//  Registers device tokens with the DGX Spark notification service
//

import Foundation
import UIKit

actor DeviceRegistrationService {
    static let shared = DeviceRegistrationService()

    private var isRegistered = false
    private var lastRegisteredToken: String?

    private var notificationServiceURL: String? {
        try? KeychainManager.shared.get(.notificationServiceURL)
    }

    private var registrationSecret: String? {
        try? KeychainManager.shared.get(.notificationSecret)
    }

    // MARK: - Registration

    func register(token: String) async {
        // Skip if already registered with same token
        guard token != lastRegisteredToken else {
            print("[DeviceRegistration] Already registered with this token")
            return
        }

        guard let baseURL = notificationServiceURL else {
            print("[DeviceRegistration] Notification service URL not configured")
            return
        }

        guard let url = URL(string: "\(baseURL)/api/v1/devices/register") else {
            print("[DeviceRegistration] Invalid URL")
            return
        }

        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        // Add authentication header if configured
        if let secret = registrationSecret {
            request.setValue(secret, forHTTPHeaderField: "X-OpenClaw-Auth")
        }

        let body: [String: Any] = [
            "device_token": token,
            "device_name": await UIDevice.current.name,
            "device_model": await UIDevice.current.model,
            "os_version": await UIDevice.current.systemVersion,
            "app_version": Bundle.main.infoDictionary?["CFBundleShortVersionString"] as? String ?? "unknown"
        ]

        do {
            request.httpBody = try JSONSerialization.data(withJSONObject: body)
            let (_, response) = try await URLSession.shared.data(for: request)

            if let httpResponse = response as? HTTPURLResponse {
                if httpResponse.statusCode == 200 {
                    print("[DeviceRegistration] Successfully registered device")
                    lastRegisteredToken = token
                    isRegistered = true
                } else {
                    print("[DeviceRegistration] Registration failed: HTTP \(httpResponse.statusCode)")
                }
            }
        } catch {
            print("[DeviceRegistration] Registration error: \(error)")
        }
    }

    func unregister() async {
        guard let token = lastRegisteredToken,
              let baseURL = notificationServiceURL,
              let url = URL(string: "\(baseURL)/api/v1/devices/\(token)") else {
            return
        }

        var request = URLRequest(url: url)
        request.httpMethod = "DELETE"

        if let secret = registrationSecret {
            request.setValue(secret, forHTTPHeaderField: "X-OpenClaw-Auth")
        }

        do {
            let (_, response) = try await URLSession.shared.data(for: request)
            if let httpResponse = response as? HTTPURLResponse, httpResponse.statusCode == 200 {
                print("[DeviceRegistration] Successfully unregistered device")
                lastRegisteredToken = nil
                isRegistered = false
            }
        } catch {
            print("[DeviceRegistration] Unregister error: \(error)")
        }
    }
}
```

#### File: `OpenClaw/App/AppDelegate.swift`

```swift
//
//  AppDelegate.swift
//  OpenClaw
//
//  Handles APNs callbacks and notification center delegation
//

import UIKit
import UserNotifications

class AppDelegate: NSObject, UIApplicationDelegate {

    func application(
        _ application: UIApplication,
        didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
    ) -> Bool {
        // Set notification delegate
        UNUserNotificationCenter.current().delegate = NotificationDelegate.shared
        return true
    }

    // MARK: - APNs Token Callbacks

    func application(
        _ application: UIApplication,
        didRegisterForRemoteNotificationsWithDeviceToken deviceToken: Data
    ) {
        Task { @MainActor in
            PushNotificationManager.shared.handleDeviceToken(deviceToken)
        }
    }

    func application(
        _ application: UIApplication,
        didFailToRegisterForRemoteNotificationsWithError error: Error
    ) {
        Task { @MainActor in
            PushNotificationManager.shared.handleRegistrationError(error)
        }
    }
}
```

#### File: `OpenClaw/App/NotificationDelegate.swift`

```swift
//
//  NotificationDelegate.swift
//  OpenClaw
//
//  Handles notification presentation and user interactions
//

import UserNotifications

class NotificationDelegate: NSObject, UNUserNotificationCenterDelegate {
    static let shared = NotificationDelegate()

    private override init() {
        super.init()
    }

    // MARK: - Foreground Presentation

    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification
    ) async -> UNNotificationPresentationOptions {
        // Show notification even when app is in foreground
        return [.banner, .sound, .badge]
    }

    // MARK: - Notification Response

    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse
    ) async {
        let userInfo = response.notification.request.content.userInfo
        let actionIdentifier = response.actionIdentifier

        print("[NotificationDelegate] Action: \(actionIdentifier)")

        // Extract OpenClaw-specific data
        let openclawData = userInfo["openclaw"] as? [String: Any]
        let notificationType = openclawData?["type"] as? String
        let context = openclawData?["context"] as? String
        let message = openclawData?["message"] as? String

        await MainActor.run {
            switch actionIdentifier {
            case UNNotificationDefaultActionIdentifier:
                // User tapped the notification
                handleNotificationTap(type: notificationType, context: context, message: message)

            case "START_CHAT_ACTION":
                // Start a new conversation
                AppState.shared.pendingAction = .startConversation(context: context)

            case "REPLY_ACTION":
                // Handle inline reply
                if let textResponse = response as? UNTextInputNotificationResponse {
                    let replyText = textResponse.userText
                    handleQuickReply(text: replyText, context: context)
                }

            case "SNOOZE_ACTION":
                // Schedule a reminder for 1 hour later
                scheduleSnooze(originalNotification: response.notification)

            case "DISMISS_ACTION", UNNotificationDismissActionIdentifier:
                // User dismissed - no action needed
                break

            default:
                break
            }
        }
    }

    // MARK: - Action Handlers

    private func handleNotificationTap(type: String?, context: String?, message: String?) {
        switch type {
        case "start_conversation":
            AppState.shared.pendingAction = .startConversation(context: context)
        case "show_message":
            if let message = message {
                AppState.shared.pendingAction = .showMessage(message)
            }
        case "open_settings":
            AppState.shared.pendingAction = .openSettings
        default:
            // Default: just open the app
            AppState.shared.pendingAction = .startConversation(context: nil)
        }
    }

    private func handleQuickReply(text: String, context: String?) {
        // Send the reply through the conversation manager
        Task { @MainActor in
            AppState.shared.pendingAction = .sendMessage(text, context: context)
        }
    }

    private func scheduleSnooze(originalNotification: UNNotification) {
        let content = originalNotification.request.content.mutableCopy() as! UNMutableNotificationContent
        content.title = "Reminder: \(content.title)"

        // Schedule for 1 hour later
        let trigger = UNTimeIntervalNotificationTrigger(timeInterval: 3600, repeats: false)
        let request = UNNotificationRequest(
            identifier: UUID().uuidString,
            content: content,
            trigger: trigger
        )

        UNUserNotificationCenter.current().add(request)
    }
}
```

### Step 1.4: Update Existing Files

#### Update: `OpenClaw/App/AppState.swift`

```swift
//
//  AppState.swift
//  OpenClaw
//
//  Global application state management
//

import Foundation
import Combine

/// Deep link actions triggered by notifications
enum DeepLinkAction: Equatable {
    case startConversation(context: String?)
    case showMessage(String)
    case sendMessage(String, context: String?)
    case openSettings

    static func == (lhs: DeepLinkAction, rhs: DeepLinkAction) -> Bool {
        switch (lhs, rhs) {
        case (.startConversation(let a), .startConversation(let b)):
            return a == b
        case (.showMessage(let a), .showMessage(let b)):
            return a == b
        case (.sendMessage(let m1, let c1), .sendMessage(let m2, let c2)):
            return m1 == m2 && c1 == c2
        case (.openSettings, .openSettings):
            return true
        default:
            return false
        }
    }
}

@MainActor
final class AppState: ObservableObject {
    static let shared = AppState()

    @Published private(set) var isConfigured: Bool = false
    @Published var showOnboarding: Bool = false

    // Notification state
    @Published var notificationPermission: NotificationPermissionStatus = .notDetermined
    @Published var pendingAction: DeepLinkAction?

    let keychainManager = KeychainManager.shared
    let networkMonitor = NetworkMonitor.shared

    private init() {
        checkConfiguration()
    }

    func checkConfiguration() {
        isConfigured = keychainManager.hasAgentId()
        showOnboarding = !isConfigured
    }

    func markConfigured() {
        isConfigured = true
        showOnboarding = false
    }

    func clearPendingAction() {
        pendingAction = nil
    }
}
```

#### Update: `OpenClaw/OpenClawApp.swift`

Add the AppDelegate adaptor and notification permission request:

```swift
@main
struct OpenClawApp: App {
    @UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
    @StateObject private var appState = AppState.shared
    @StateObject private var pushManager = PushNotificationManager.shared

    var body: some Scene {
        WindowGroup {
            Group {
                if appState.isConfigured {
                    ConversationView()
                } else {
                    OnboardingView()
                }
            }
            .environmentObject(appState)
            .environmentObject(pushManager)
            .task {
                // Check and request notification permissions
                await pushManager.checkPermissionStatus()
                if pushManager.permissionStatus == .notDetermined {
                    _ = await pushManager.requestPermission()
                }
            }
            .onReceive(NotificationCenter.default.publisher(for: UIApplication.didBecomeActiveNotification)) { _ in
                // Clear badge when app becomes active
                Task {
                    await pushManager.clearBadge()
                }
            }
        }
    }
}
```

#### Update: `OpenClaw/Services/KeychainManager.swift`

Add new keys for notification service:

```swift
enum Key: String {
    case elevenLabsApiKey = "elevenlabs_api_key"
    case agentId = "elevenlabs_agent_id"
    case openClawEndpoint = "openclaw_endpoint"
    // New keys for notifications
    case notificationServiceURL = "notification_service_url"
    case notificationSecret = "notification_secret"
    case notificationsEnabled = "notifications_enabled"
}

// Add convenience methods
func hasNotificationServiceURL() -> Bool {
    (try? get(.notificationServiceURL)) != nil
}
```

---

## Phase 2: DGX Spark Notification Service

### Directory Structure

```
dgx-spark/
└── openclaw-notifications/
    ├── config.py              # Configuration and secrets
    ├── main.py                # FastAPI application
    ├── apns_client.py         # Apple Push Notification client
    ├── device_registry.py     # Device token storage
    ├── notification_types.py  # Payload schemas
    ├── outreach_scheduler.py  # Proactive notification triggers
    ├── requirements.txt       # Python dependencies
    └── .env                   # Environment variables (gitignored)
```

### Step 2.1: Install Dependencies

```bash
# requirements.txt
fastapi>=0.109.0
uvicorn[standard]>=0.27.0
pyjwt>=2.8.0
cryptography>=42.0.0
httpx[http2]>=0.26.0
apscheduler>=3.10.0
python-dotenv>=1.0.0
aiosqlite>=0.19.0
```

### Step 2.2: Core Files

#### File: `config.py`

```python
import os
from pathlib import Path
from dotenv import load_dotenv

load_dotenv()

class Config:
    # APNs Configuration
    APNS_KEY_PATH = os.getenv("APNS_KEY_PATH", "AuthKey.p8")
    APNS_KEY_ID = os.getenv("APNS_KEY_ID")
    APNS_TEAM_ID = os.getenv("APNS_TEAM_ID")
    APNS_BUNDLE_ID = os.getenv("APNS_BUNDLE_ID", "com.openclaw.app")
    APNS_SANDBOX = os.getenv("APNS_SANDBOX", "true").lower() == "true"

    # Server Configuration
    HOST = os.getenv("HOST", "0.0.0.0")
    PORT = int(os.getenv("PORT", "8100"))

    # Security
    REGISTRATION_SECRET = os.getenv("REGISTRATION_SECRET")

    # OpenClaw LLM endpoint (for AI-generated content)
    OPENCLAW_ENDPOINT = os.getenv("OPENCLAW_ENDPOINT", "http://localhost:8080")

    # Database
    DATABASE_PATH = os.getenv("DATABASE_PATH", "devices.db")
```

#### File: `apns_client.py`

```python
"""
Apple Push Notification Service client using HTTP/2 and JWT authentication.
"""

import jwt
import time
import json
from pathlib import Path
import httpx
from dataclasses import dataclass
from typing import Optional
from config import Config


@dataclass
class APNsResponse:
    success: bool
    status_code: int
    reason: Optional[str] = None
    apns_id: Optional[str] = None


class APNsClient:
    """Sends push notifications via Apple Push Notification Service."""

    APNS_PRODUCTION = "https://api.push.apple.com"
    APNS_SANDBOX = "https://api.sandbox.push.apple.com"

    def __init__(self):
        self.key_id = Config.APNS_KEY_ID
        self.team_id = Config.APNS_TEAM_ID
        self.bundle_id = Config.APNS_BUNDLE_ID
        self.base_url = self.APNS_SANDBOX if Config.APNS_SANDBOX else self.APNS_PRODUCTION

        key_path = Path(Config.APNS_KEY_PATH)
        if not key_path.exists():
            raise FileNotFoundError(f"APNs key not found: {key_path}")
        self._private_key = key_path.read_text()

        self._token: Optional[str] = None
        self._token_time: float = 0

    def _get_bearer_token(self) -> str:
        """Generate or reuse a JWT bearer token (valid for ~1 hour)."""
        now = time.time()
        # Refresh token if older than 55 minutes
        if self._token and (now - self._token_time) < 3300:
            return self._token

        self._token = jwt.encode(
            {"iss": self.team_id, "iat": int(now)},
            self._private_key,
            algorithm="ES256",
            headers={"kid": self.key_id}
        )
        self._token_time = now
        return self._token

    async def send(
        self,
        device_token: str,
        title: str,
        body: str,
        subtitle: Optional[str] = None,
        category: str = "OPENCLAW_MESSAGE",
        badge: Optional[int] = None,
        sound: str = "default",
        data: Optional[dict] = None,
        thread_id: str = "openclaw"
    ) -> APNsResponse:
        """Send a push notification to a single device."""

        # Build the alert payload
        alert = {"title": title, "body": body}
        if subtitle:
            alert["subtitle"] = subtitle

        # Build the aps payload
        aps = {
            "alert": alert,
            "sound": sound,
            "category": category,
            "mutable-content": 1,
            "thread-id": thread_id
        }
        if badge is not None:
            aps["badge"] = badge

        # Build full payload
        payload = {"aps": aps}
        if data:
            payload["openclaw"] = data

        url = f"{self.base_url}/3/device/{device_token}"
        headers = {
            "authorization": f"bearer {self._get_bearer_token()}",
            "apns-topic": self.bundle_id,
            "apns-push-type": "alert",
            "apns-priority": "10",
            "apns-expiration": "0"
        }

        async with httpx.AsyncClient(http2=True) as client:
            try:
                resp = await client.post(
                    url,
                    json=payload,
                    headers=headers,
                    timeout=30.0
                )

                if resp.status_code == 200:
                    return APNsResponse(
                        success=True,
                        status_code=200,
                        apns_id=resp.headers.get("apns-id")
                    )
                else:
                    error_data = resp.json() if resp.content else {}
                    return APNsResponse(
                        success=False,
                        status_code=resp.status_code,
                        reason=error_data.get("reason", "Unknown error")
                    )
            except Exception as e:
                return APNsResponse(
                    success=False,
                    status_code=0,
                    reason=str(e)
                )

    async def send_to_all(
        self,
        tokens: list[str],
        title: str,
        body: str,
        **kwargs
    ) -> list[tuple[str, APNsResponse]]:
        """Send notification to multiple devices."""
        results = []
        for token in tokens:
            response = await self.send(token, title, body, **kwargs)
            results.append((token, response))
        return results
```

#### File: `device_registry.py`

```python
"""
SQLite-backed device token registry.
"""

import aiosqlite
from datetime import datetime
from dataclasses import dataclass
from typing import Optional
from config import Config


@dataclass
class Device:
    token: str
    device_name: str
    device_model: str
    os_version: str
    app_version: str
    registered_at: str
    last_seen_at: str


class DeviceRegistry:
    def __init__(self):
        self.db_path = Config.DATABASE_PATH

    async def init_db(self):
        """Initialize the database schema."""
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("""
                CREATE TABLE IF NOT EXISTS devices (
                    token TEXT PRIMARY KEY,
                    device_name TEXT,
                    device_model TEXT,
                    os_version TEXT,
                    app_version TEXT,
                    registered_at TEXT,
                    last_seen_at TEXT
                )
            """)
            await db.commit()

    async def register(
        self,
        token: str,
        device_name: str = "",
        device_model: str = "",
        os_version: str = "",
        app_version: str = ""
    ) -> Device:
        """Register or update a device token."""
        now = datetime.utcnow().isoformat()

        async with aiosqlite.connect(self.db_path) as db:
            # Check if exists
            cursor = await db.execute(
                "SELECT registered_at FROM devices WHERE token = ?",
                (token,)
            )
            existing = await cursor.fetchone()

            if existing:
                # Update existing
                await db.execute("""
                    UPDATE devices SET
                        device_name = ?,
                        device_model = ?,
                        os_version = ?,
                        app_version = ?,
                        last_seen_at = ?
                    WHERE token = ?
                """, (device_name, device_model, os_version, app_version, now, token))
                registered_at = existing[0]
            else:
                # Insert new
                await db.execute("""
                    INSERT INTO devices VALUES (?, ?, ?, ?, ?, ?, ?)
                """, (token, device_name, device_model, os_version, app_version, now, now))
                registered_at = now

            await db.commit()

        return Device(
            token=token,
            device_name=device_name,
            device_model=device_model,
            os_version=os_version,
            app_version=app_version,
            registered_at=registered_at,
            last_seen_at=now
        )

    async def unregister(self, token: str) -> bool:
        """Remove a device token."""
        async with aiosqlite.connect(self.db_path) as db:
            cursor = await db.execute(
                "DELETE FROM devices WHERE token = ?",
                (token,)
            )
            await db.commit()
            return cursor.rowcount > 0

    async def get_all_tokens(self) -> list[str]:
        """Get all registered device tokens."""
        async with aiosqlite.connect(self.db_path) as db:
            cursor = await db.execute("SELECT token FROM devices")
            rows = await cursor.fetchall()
            return [row[0] for row in rows]

    async def get_all_devices(self) -> list[Device]:
        """Get all registered devices with metadata."""
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            cursor = await db.execute("SELECT * FROM devices")
            rows = await cursor.fetchall()
            return [Device(**dict(row)) for row in rows]

    async def get_device(self, token: str) -> Optional[Device]:
        """Get a specific device by token."""
        async with aiosqlite.connect(self.db_path) as db:
            db.row_factory = aiosqlite.Row
            cursor = await db.execute(
                "SELECT * FROM devices WHERE token = ?",
                (token,)
            )
            row = await cursor.fetchone()
            return Device(**dict(row)) if row else None
```

#### File: `main.py`

```python
"""
OpenClaw Notification Service - FastAPI Application
"""

from fastapi import FastAPI, HTTPException, Header, BackgroundTasks
from pydantic import BaseModel
from typing import Optional
from contextlib import asynccontextmanager

from config import Config
from apns_client import APNsClient
from device_registry import DeviceRegistry
from outreach_scheduler import OutreachScheduler


# Initialize components
registry = DeviceRegistry()
apns = APNsClient()
scheduler: Optional[OutreachScheduler] = None


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan handler."""
    global scheduler

    # Startup
    await registry.init_db()
    scheduler = OutreachScheduler(apns, registry)
    scheduler.start()
    print("✓ Notification service started")

    yield

    # Shutdown
    if scheduler:
        scheduler.stop()
    print("✓ Notification service stopped")


app = FastAPI(
    title="OpenClaw Notification Service",
    description="Push notification backend for OpenClaw iOS app",
    version="1.0.0",
    lifespan=lifespan
)


# Request/Response Models
class DeviceRegistration(BaseModel):
    device_token: str
    device_name: str = ""
    device_model: str = ""
    os_version: str = ""
    app_version: str = ""


class NotificationRequest(BaseModel):
    title: str
    body: str
    subtitle: Optional[str] = None
    category: str = "OPENCLAW_MESSAGE"
    badge: Optional[int] = None
    data: Optional[dict] = None


class NotificationResponse(BaseModel):
    sent: int
    results: list[dict]


# Auth dependency
def verify_auth(x_openclaw_auth: Optional[str] = Header(None)):
    if Config.REGISTRATION_SECRET:
        if x_openclaw_auth != Config.REGISTRATION_SECRET:
            raise HTTPException(status_code=401, detail="Unauthorized")


# Endpoints
@app.get("/health")
async def health_check():
    return {"status": "healthy", "service": "openclaw-notifications"}


@app.post("/api/v1/devices/register")
async def register_device(
    req: DeviceRegistration,
    x_openclaw_auth: Optional[str] = Header(None)
):
    verify_auth(x_openclaw_auth)

    device = await registry.register(
        token=req.device_token,
        device_name=req.device_name,
        device_model=req.device_model,
        os_version=req.os_version,
        app_version=req.app_version
    )

    return {
        "status": "registered",
        "device": {
            "token": device.token[:16] + "...",
            "device_name": device.device_name,
            "registered_at": device.registered_at
        }
    }


@app.delete("/api/v1/devices/{device_token}")
async def unregister_device(
    device_token: str,
    x_openclaw_auth: Optional[str] = Header(None)
):
    verify_auth(x_openclaw_auth)

    removed = await registry.unregister(device_token)
    if not removed:
        raise HTTPException(status_code=404, detail="Device not found")

    return {"status": "unregistered"}


@app.get("/api/v1/devices")
async def list_devices(x_openclaw_auth: Optional[str] = Header(None)):
    verify_auth(x_openclaw_auth)

    devices = await registry.get_all_devices()
    return {
        "count": len(devices),
        "devices": [
            {
                "token": d.token[:16] + "...",
                "device_name": d.device_name,
                "device_model": d.device_model,
                "last_seen_at": d.last_seen_at
            }
            for d in devices
        ]
    }


@app.post("/api/v1/notifications/send", response_model=NotificationResponse)
async def send_notification(
    req: NotificationRequest,
    x_openclaw_auth: Optional[str] = Header(None)
):
    verify_auth(x_openclaw_auth)

    tokens = await registry.get_all_tokens()
    if not tokens:
        return NotificationResponse(sent=0, results=[])

    results = await apns.send_to_all(
        tokens=tokens,
        title=req.title,
        body=req.body,
        subtitle=req.subtitle,
        category=req.category,
        badge=req.badge,
        data=req.data
    )

    return NotificationResponse(
        sent=len(results),
        results=[
            {
                "token": token[:16] + "...",
                "success": resp.success,
                "reason": resp.reason
            }
            for token, resp in results
        ]
    )


@app.post("/api/v1/notifications/send/{device_token}")
async def send_to_device(
    device_token: str,
    req: NotificationRequest,
    x_openclaw_auth: Optional[str] = Header(None)
):
    verify_auth(x_openclaw_auth)

    device = await registry.get_device(device_token)
    if not device:
        raise HTTPException(status_code=404, detail="Device not found")

    response = await apns.send(
        device_token=device_token,
        title=req.title,
        body=req.body,
        subtitle=req.subtitle,
        category=req.category,
        badge=req.badge,
        data=req.data
    )

    return {
        "success": response.success,
        "apns_id": response.apns_id,
        "reason": response.reason
    }


if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host=Config.HOST, port=Config.PORT)
```

---

## Phase 3: Device Registration Flow

This phase connects the iOS app to the DGX Spark notification service.

### Step 3.1: Add Settings UI for Notification Service

Update `SettingsView.swift` to include notification service configuration:

```swift
// Add to SettingsView.swift - new section

// MARK: - Notifications Section
SettingsSection(title: "Notifications") {
    SettingsTextField(
        label: "Notification Service URL",
        placeholder: "https://your-server.ts.net:8100",
        text: $viewModel.notificationServiceURL
    )

    if viewModel.isNotificationServiceConfigured {
        SettingsSecureField(
            label: "Registration Secret",
            placeholder: "Optional authentication secret",
            text: $viewModel.notificationSecret
        )
    }

    SettingsRow {
        Text("Push Notifications")
        Spacer()
        Toggle("", isOn: $viewModel.notificationsEnabled)
            .labelsHidden()
            .tint(.anthropicCoral)
    }

    // Permission status
    HStack {
        Text("Permission Status")
            .foregroundStyle(Color.textSecondary)
        Spacer()
        Text(notificationStatusText)
            .foregroundStyle(notificationStatusColor)
    }
    .padding(.horizontal, 16)
    .padding(.vertical, 14)
    .background(Color.surfacePrimary)
}

// Add computed properties
private var notificationStatusText: String {
    switch pushManager.permissionStatus {
    case .authorized: return "Enabled"
    case .denied: return "Denied"
    case .provisional: return "Provisional"
    case .notDetermined: return "Not Set"
    }
}

private var notificationStatusColor: Color {
    switch pushManager.permissionStatus {
    case .authorized, .provisional: return .statusConnected
    case .denied: return .statusDisconnected
    case .notDetermined: return .textSecondary
    }
}
```

### Step 3.2: Update SettingsViewModel

Add notification-related properties and methods to `SettingsViewModel.swift`:

```swift
// Add properties
@Published var notificationServiceURL: String = ""
@Published var notificationSecret: String = ""
@Published var notificationsEnabled: Bool = true

var isNotificationServiceConfigured: Bool {
    !notificationServiceURL.isEmpty
}

// Add to init()
notificationServiceURL = (try? keychainManager.get(.notificationServiceURL)) ?? ""
notificationSecret = (try? keychainManager.get(.notificationSecret)) ?? ""
notificationsEnabled = (try? keychainManager.get(.notificationsEnabled)) == "true"

// Add to save()
if !notificationServiceURL.isEmpty {
    try keychainManager.save(notificationServiceURL, for: .notificationServiceURL)
}
if !notificationSecret.isEmpty {
    try keychainManager.save(notificationSecret, for: .notificationSecret)
}
try keychainManager.save(notificationsEnabled ? "true" : "false", for: .notificationsEnabled)
```

---

## Phase 4: Proactive Outreach Engine

### File: `outreach_scheduler.py`

```python
"""
Proactive notification scheduler - triggers notifications based on schedules and events.
"""

from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger
import httpx
from typing import Optional

from config import Config
from apns_client import APNsClient
from device_registry import DeviceRegistry


class OutreachScheduler:
    """Manages scheduled and event-driven notifications."""

    def __init__(self, apns: APNsClient, registry: DeviceRegistry):
        self.apns = apns
        self.registry = registry
        self.scheduler = AsyncIOScheduler()
        self._setup_jobs()

    def _setup_jobs(self):
        """Configure scheduled jobs."""

        # Morning briefing at 8 AM local time
        self.scheduler.add_job(
            self.morning_briefing,
            CronTrigger(hour=8, minute=0),
            id="morning_briefing",
            name="Morning Briefing"
        )

        # Evening summary at 6 PM
        self.scheduler.add_job(
            self.evening_summary,
            CronTrigger(hour=18, minute=0),
            id="evening_summary",
            name="Evening Summary"
        )

    def start(self):
        """Start the scheduler."""
        self.scheduler.start()
        print("✓ Outreach scheduler started")

    def stop(self):
        """Stop the scheduler."""
        self.scheduler.shutdown()

    # MARK: - Scheduled Jobs

    async def morning_briefing(self):
        """Send a personalized morning briefing."""
        # Ask OpenClaw to generate the briefing
        briefing = await self._ask_openclaw(
            "Generate a brief, friendly morning greeting for the user. "
            "Keep it under 50 words. Be warm and motivating. "
            "Don't use bullet points or formatting."
        )

        if not briefing:
            briefing = "Good morning! Ready to start your day?"

        await self._send_to_all(
            title="Good Morning ☀️",
            body=briefing,
            category="OPENCLAW_BRIEFING",
            data={"type": "start_conversation", "context": "morning_briefing"}
        )

    async def evening_summary(self):
        """Send an evening summary notification."""
        summary = await self._ask_openclaw(
            "Generate a brief evening check-in message. "
            "Keep it under 50 words. Be warm and supportive."
        )

        if not summary:
            summary = "How was your day? I'm here if you'd like to chat."

        await self._send_to_all(
            title="Evening Check-in",
            body=summary,
            category="OPENCLAW_MESSAGE",
            data={"type": "start_conversation", "context": "evening_summary"}
        )

    # MARK: - Event-Driven Notifications

    async def notify_task_complete(self, task_name: str, result: str):
        """Notify user that a background task completed."""
        await self._send_to_all(
            title="Task Complete",
            body=f"{task_name}: {result[:100]}",
            category="OPENCLAW_ALERT",
            data={"type": "show_message", "message": result}
        )

    async def send_reminder(self, message: str, delay_minutes: int = 0):
        """Send a reminder notification."""
        if delay_minutes > 0:
            # Schedule for later
            from datetime import datetime, timedelta
            run_time = datetime.now() + timedelta(minutes=delay_minutes)
            self.scheduler.add_job(
                self._send_reminder_now,
                "date",
                run_date=run_time,
                args=[message]
            )
        else:
            await self._send_reminder_now(message)

    async def _send_reminder_now(self, message: str):
        await self._send_to_all(
            title="Reminder",
            body=message,
            category="OPENCLAW_REMINDER",
            data={"type": "start_conversation", "context": "reminder"}
        )

    # MARK: - AI-Initiated Outreach

    async def ai_proactive_check(self):
        """Let OpenClaw decide if it should reach out."""
        decision = await self._ask_openclaw(
            "Based on your knowledge, should you proactively reach out "
            "to the user right now? If yes, what would you say? "
            "Respond with JSON: {\"should_notify\": true/false, \"message\": \"...\"}"
        )

        if decision:
            import json
            try:
                data = json.loads(decision)
                if data.get("should_notify") and data.get("message"):
                    await self._send_to_all(
                        title="OpenClaw",
                        body=data["message"],
                        category="OPENCLAW_MESSAGE",
                        data={"type": "start_conversation", "context": "ai_initiated"}
                    )
            except json.JSONDecodeError:
                pass

    # MARK: - Helpers

    async def _ask_openclaw(self, prompt: str) -> Optional[str]:
        """Call the local OpenClaw endpoint to generate content."""
        try:
            async with httpx.AsyncClient() as client:
                resp = await client.post(
                    f"{Config.OPENCLAW_ENDPOINT}/v1/chat/completions",
                    json={
                        "model": "openclaw",
                        "messages": [{"role": "user", "content": prompt}],
                        "max_tokens": 150,
                        "temperature": 0.7
                    },
                    timeout=30.0
                )
                if resp.status_code == 200:
                    data = resp.json()
                    return data["choices"][0]["message"]["content"]
        except Exception as e:
            print(f"OpenClaw request failed: {e}")
        return None

    async def _send_to_all(self, **kwargs):
        """Send notification to all registered devices."""
        tokens = await self.registry.get_all_tokens()
        if tokens:
            await self.apns.send_to_all(tokens, **kwargs)
```

---

## Phase 5: Deep Linking & Actions

### Update ConversationView to Handle Pending Actions

```swift
// In ConversationView.swift

.onAppear {
    handlePendingAction()
}
.onChange(of: appState.pendingAction) { _, newAction in
    if newAction != nil {
        handlePendingAction()
    }
}

private func handlePendingAction() {
    guard let action = appState.pendingAction else { return }

    switch action {
    case .startConversation(let context):
        // Start a conversation, optionally with context
        Task {
            if let context = context {
                // Could pre-populate a message or set agent context
                print("Starting conversation with context: \(context)")
            }
            await viewModel.startConversation()
        }

    case .showMessage(let message):
        // Display a message from the AI
        viewModel.addReceivedMessage(message)

    case .sendMessage(let text, _):
        // Send a message (from notification reply)
        Task {
            await viewModel.startConversation()
            await viewModel.sendMessage(text)
        }

    case .openSettings:
        viewModel.showSettings = true
    }

    appState.clearPendingAction()
}
```

---

## Phase 6: AI-Initiated Outreach

### OpenClaw Tool Integration

Add `send_notification` as a tool in your OpenClaw agent configuration:

```json
{
  "name": "send_notification",
  "description": "Send a push notification to the user's phone. Use this when you have important information, a reminder, or want to proactively reach out.",
  "parameters": {
    "type": "object",
    "properties": {
      "title": {
        "type": "string",
        "description": "Notification title (keep short)"
      },
      "message": {
        "type": "string",
        "description": "Notification body text"
      },
      "urgency": {
        "type": "string",
        "enum": ["low", "normal", "high"],
        "description": "How urgent is this notification"
      }
    },
    "required": ["title", "message"]
  }
}
```

### Tool Handler on DGX Spark

```python
# Add to your OpenClaw tool handlers

async def handle_send_notification(title: str, message: str, urgency: str = "normal"):
    """Tool handler for AI-initiated notifications."""

    category = {
        "low": "OPENCLAW_MESSAGE",
        "normal": "OPENCLAW_MESSAGE",
        "high": "OPENCLAW_ALERT"
    }.get(urgency, "OPENCLAW_MESSAGE")

    tokens = await registry.get_all_tokens()
    if tokens:
        await apns.send_to_all(
            tokens=tokens,
            title=title,
            body=message,
            category=category,
            data={"type": "start_conversation", "context": "ai_initiated"}
        )

    return {"success": True, "devices_notified": len(tokens)}
```

---

## Security Considerations

### Summary

| Component | Security Measure |
|-----------|------------------|
| APNs .p8 Key | Stored only on DGX Spark, chmod 600, never in git |
| Registration Endpoint | Protected by shared secret header |
| Device Tokens | Stored in SQLite, tokens are opaque (no PII) |
| iOS Keychain | Notification secret stored encrypted |
| Transport | HTTPS via Tailscale Funnel |

### Best Practices

1. **Never log full device tokens** - only first 16 chars
2. **Rotate registration secret** periodically
3. **Validate token format** before storing
4. **Rate limit** registration endpoint
5. **Monitor for invalid tokens** and remove them

---

## Testing Strategy

### iOS App Testing

1. **Simulator Limitations**: APNs doesn't work on simulator
2. **Use TestFlight** for push notification testing
3. **Console.app** to view device logs

### Backend Testing

```bash
# Test notification endpoint
curl -X POST https://your-server.ts.net:8100/api/v1/notifications/send \
  -H "Content-Type: application/json" \
  -H "X-OpenClaw-Auth: your-secret" \
  -d '{"title": "Test", "body": "Hello from OpenClaw!"}'
```

### Checklist

- [ ] App requests notification permission
- [ ] Device token is generated and logged
- [ ] Token is sent to backend successfully
- [ ] Backend can send test notification
- [ ] Notification appears on device
- [ ] Tapping notification opens app
- [ ] Notification actions work (Reply, Start Chat)
- [ ] Scheduled notifications fire on time
- [ ] Badge clears when app opens

---

## File Changes Summary

### New iOS Files

| File | Purpose |
|------|---------|
| `Services/PushNotificationManager.swift` | APNs registration, permissions, categories |
| `Services/DeviceRegistrationService.swift` | Backend registration |
| `App/AppDelegate.swift` | APNs callbacks |
| `App/NotificationDelegate.swift` | Notification handling |

### Modified iOS Files

| File | Changes |
|------|---------|
| `App/AppState.swift` | Add `DeepLinkAction`, `pendingAction`, notification state |
| `OpenClawApp.swift` | Add AppDelegate adaptor, permission request |
| `Services/KeychainManager.swift` | Add notification-related keys |
| `Features/Settings/SettingsView.swift` | Notification preferences UI |
| `Features/Settings/SettingsViewModel.swift` | Notification settings logic |
| `Features/Conversation/ConversationView.swift` | Handle pending actions |

### New DGX Spark Files

| File | Purpose |
|------|---------|
| `config.py` | Configuration and secrets |
| `main.py` | FastAPI application |
| `apns_client.py` | APNs HTTP/2 client |
| `device_registry.py` | SQLite device storage |
| `outreach_scheduler.py` | Scheduled notifications |

### Xcode Project Changes

- Add **Push Notifications** capability
- Add **Background Modes** → Remote notifications

---

## Timeline Estimate

| Phase | Effort | Dependencies |
|-------|--------|--------------|
| Phase 1: iOS APNs Foundation | 1-2 days | APNs key from Apple |
| Phase 2: DGX Spark Service | 1-2 days | Phase 1 complete |
| Phase 3: Device Registration | 0.5 days | Phase 1 & 2 complete |
| Phase 4: Outreach Engine | 1 day | Phase 3 complete |
| Phase 5: Deep Linking | 0.5 days | Phase 3 complete |
| Phase 6: AI-Initiated | 1 day | Phase 4 complete |
| **Total** | **5-7 days** | |

---

<p align="center">
  <em>OpenClaw Notifications Implementation Plan v1.0</em>
</p>
