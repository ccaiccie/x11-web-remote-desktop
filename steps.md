Here's a **detailed write-up in Markdown format**, including **step-by-step configuration**, **challenges faced**, and **why each decision was made** — ideal for a technical blog or documentation.

# 🔐 Browser-Based Remote Access to Ubuntu Desktop using x11vnc and noVNC (with Auto Login)

This guide details how to set up secure, browser-accessible remote desktop access to Ubuntu using `x11vnc` and `noVNC`, including auto-login at boot. It includes real-world issues faced during configuration and how they were resolved.

---

## 🎯 Objectives

- Serve Ubuntu desktop over the web securely via `noVNC`.
- Start VNC server automatically **before user manually logs in**.
- Solve common errors like *authentication required*, *black screen*, or *password rejected*.
- Ensure service resilience with `systemd`.

---

## 🧩 Problem Statement

Typical `x11vnc` setups only work **after a user logs in** because it mirrors the currently active X session. However, we wanted:

> ✅ A system that boots → logs in automatically → runs x11vnc + noVNC → exposes a web-accessible VNC session.

---

## 🛠️ Step-by-Step Setup

### 1. **Install Required Packages**

```bash
sudo apt update
sudo apt install x11vnc novnc websockify
````

---

### 2. **Create a VNC Password**

```bash
mkdir -p /home/superuser/.vnc
x11vnc -storepasswd YOUR_STRONG_PASSWORD /home/superuser/.vnc/passwd
chmod 600 /home/superuser/.vnc/passwd
chown superuser:superuser /home/superuser/.vnc/passwd
```

> 🔒 *Avoid storing the password in `/root/.vnc/passwd` unless the service runs as root. Mismatched ownership can cause "password rejected" errors.*

---

### 3. **Enable GUI Auto-Login (GDM)**

Edit:

```bash
sudo nano /etc/gdm3/custom.conf
```

Uncomment or add:

```ini
[daemon]
AutomaticLoginEnable = true
AutomaticLogin = superuser
```

> ✅ *This ensures the desktop session starts without needing manual login — a requirement for x11vnc to mirror the session.*

---

### 4. **Determine the Active X Authority File**

Check with:

```bash
ps aux | grep '[X]org'
```

Example output:

```bash
/usr/lib/xorg/Xorg vt2 -displayfd 3 -auth /run/user/1000/gdm/Xauthority ...
```

Use the path after `-auth` in the next steps.

---

### 5. **Create a Systemd Service for x11vnc**

```bash
sudo nano /etc/systemd/system/x11vnc.service
```

```ini
[Unit]
Description=Start x11vnc after auto-login
After=graphical.target

[Service]
Type=simple
User=superuser
ExecStart=/usr/bin/x11vnc \
  -auth /run/user/1000/gdm/Xauthority \
  -display :0 \
  -rfbauth /home/superuser/.vnc/passwd \
  -noshm -forever -shared -rfbport 5901
Restart=always

[Install]
WantedBy=graphical.target
```

> ⚠️ *Using `multi-user.target` didn’t work because the graphical session hadn’t launched yet. Switched to `graphical.target` to wait for desktop.*

---

### 6. **Reload Systemd and Enable the Service**

```bash
sudo systemctl daemon-reexec
sudo systemctl enable x11vnc
sudo systemctl start x11vnc
```

---

### 7. **Create and Configure noVNC Web Service**

```bash
sudo nano /etc/systemd/system/novnc.service
```

```ini
[Unit]
Description=Serve noVNC WebSocket Proxy
After=network.target

[Service]
Type=simple
User=superuser
ExecStart=/usr/bin/websockify --web=/usr/share/novnc/ --wrap-mode=ignore 6080 localhost:5901
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable:

```bash
sudo systemctl enable novnc
sudo systemctl start novnc
```

---

## ✅ Final Result

* VNC accessible on `localhost:5901`
* Web viewer available at `http://<host-ip>:6080`
* Desktop auto-login enabled
* Services auto-recover on crash or reboot

---

## 🧱 Challenges and Solutions

| Problem                                                    | Root Cause                                  | Solution                                                                            |                                                    |
| ---------------------------------------------------------- | ------------------------------------------- | ----------------------------------------------------------------------------------- | -------------------------------------------------- |
| ❌ x11vnc: "Authentication required, no protocol specified" | Wrong or missing `-auth` file               | Used \`ps aux                                                                       | grep Xorg\` to extract the correct Xauthority file |
| ❌ Password rejected                                        | Ownership mismatch or wrong `-rfbauth` path | Ensured `.vnc/passwd` owned by the target user, not root                            |                                                    |
| ❌ Black screen on connect                                  | No session running yet                      | Enabled GUI auto-login via `gdm3`                                                   |                                                    |
| ❌ Port already in use                                      | Multiple x11vnc instances or stuck services | Used `sudo pkill x11vnc` and changed to port `5901`                                 |                                                    |
| ❌ VNC starts only after login                              | x11vnc mirrors active session               | Solved by enabling **auto-login** and changing systemd target to `graphical.target` |                                                    |

---

## 🧪 Testing

1. Reboot system
2. Wait 20–30 seconds
3. Visit: [http://your-ip:6080](http://your-ip:6080)
4. Enter password from step 2

You should see the full desktop session running automatically.

---

## 🚀 Next Steps

* Add a firewall rule for port `6080` (web) and `5901` (VNC)
* Set up HTTPS reverse proxy for production security
* Add logging and monitoring for both services

---

## 🧰 Bonus: One-Liner Startup Test (Manual)

```bash
x11vnc -auth /run/user/1000/gdm/Xauthority -display :0 -rfbauth /home/superuser/.vnc/passwd -forever -shared -rfbport 5901
```

---

Let me know if you'd like the **full Bash script to automate this entire setup**.

```

---

Would you like this turned into a GitHub-ready `.md` file or posted as an article draft for Medium/Dev.to?
```
