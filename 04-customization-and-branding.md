# 04. Customization, Native Next.js UI & Whiteboard Integration

Unity Meet operates on a **Pure Next.js Frontend + Jitsi WebRTC Backend Architecture**.

---

## 🏗️ Architecture: Pure Next.js + Headless Jitsi

1. **No CSS Hacking / No `head.html`:** We eliminated `head.html` and iframe stylesheet injections completely. Jitsi runs strictly as a clean, high-performance WebRTC media engine.
2. **100% Native Next.js Control:**
   * Floating Capsule Toolbar
   * Pre-Join Green Room (Camera test & Mic visualizer)
   * Collaborative Whiteboard Canvas
   * Leave / End Meeting Confirmation Modal
   * Room Passcode / Security Modal
   * Post-Call "You left the meeting" Screen
3. **Seamless API Communication:** Next.js interacts with Jitsi through the official Jitsi IFrame API (`externalApi.executeCommand(...)`) and real-time state listeners (`audioMuteStatusChanged`, `videoMuteStatusChanged`, `participantJoined`, etc.).

---

## 🎛️ Native Toolbar Buttons (Next.js)

* 🎙️ **Microphone:** Live synced state with 1-click mute/unmute.
* 📹 **Camera:** Live synced state with 1-click video toggle.
* 🖥️ **Share Screen:** Native screen, window, or tab sharing.
* 💬 **Chat:** In-call text messaging sidebar toggle.
* ✋ **Raise Hand:** Speaker priority toggle with purple active glow.
* 👥 **Participants:** Live participant counter badge.
* ✏️ **Interactive Whiteboard:** Built-in drawing canvas with shapes, colors, eraser, clear, and PNG export.
* ⊞ **Tile View:** Grid layout switcher.
* 🔒 **Security:** Real-time meeting PIN passcode lock modal.
* 🔴 **End Call / Leave:** 1-click confirmation dialog modal before exiting to the post-call screen.

---

## ✏️ Built-In Collaborative Whiteboard

* **Drawing Tools:** Pen, Rectangle, Circle, Line, Eraser.
* **Palette:** Purple (`#a855f7`), Blue (`#3b82f6`), Green (`#10b981`), Red (`#ef4444`), Orange (`#f59e0b`), White (`#ffffff`).
* **Export:** 1-click PNG image download of the board canvas.
* **Zero Dependency:** Operates directly in Next.js without requiring external backend relay services or Prosody plugins.
