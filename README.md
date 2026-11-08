## The wizard script

This is a single file that works on both Termux and Windows. It detects the platform and adjusts its hints and commands. The Tailscale step checks that Tailscale is running and reads your tailnet name. The publish step runs `tailscale serve` on port 443 pointing at the local Phylum server. The final step confirms that the tailnet URL responds.

---

## Termux

Run this in Termux. It writes the wizard with a heredoc and launches it:

```bash
mkdir -p ~/phylum-setup
cat > ~/phylum-setup/phylum_setup_gui.py <<'PYEOF'
#!/usr/bin/env python3
"""Phylum setup wizard: GUI with step-by-step guidance. Standard library only."""
import json
import os
import platform
import queue
import shutil
import socket
import ssl
import subprocess
import sys
import threading
import urllib.error
import urllib.request

try:
    import tkinter as tk
    from tkinter import ttk
    from tkinter.scrolledtext import ScrolledText
except ImportError:
    sys.exit("tkinter is missing. Termux: pkg install python-tkinter | Windows: reinstall Python with the tcl/tk option.")

PORT = 2448
REPO_URL = "https://codeberg.org/shroff/phylum.git"
IS_WINDOWS = platform.system() == "Windows"
IS_TERMUX = "TERMUX_VERSION" in os.environ or "com.termux" in sys.prefix
CREATE_NO_WINDOW = 0x08000000 if IS_WINDOWS else 0
BASE_DIR = os.path.join(os.path.expanduser("~"), "phylum-setup")
REPO_DIR = os.path.join(BASE_DIR, "phylum")
LOG_PATH = os.path.join(BASE_DIR, "setup.log")


def run(cmd, cwd=None, timeout=60):
    try:
        p = subprocess.run(cmd, cwd=cwd, capture_output=True, text=True, errors="replace",
                           timeout=timeout, creationflags=CREATE_NO_WINDOW)
        return p.returncode, (p.stdout + p.stderr).strip()
    except FileNotFoundError:
        return 127, f"Command not found: {cmd[0]}"
    except subprocess.TimeoutExpired:
        return 124, f"Timed out after {timeout}s"


def pick(termux_text, windows_text):
    return termux_text if IS_TERMUX else windows_text


def find_tailscale():
    found = shutil.which("tailscale")
    if found:
        return found
    if IS_WINDOWS:
        for p in (r"C:\Program Files\Tailscale\tailscale.exe",
                  r"C:\Program Files (x86)\Tailscale\tailscale.exe"):
            if os.path.exists(p):
                return p
    return None


def port_open(host="127.0.0.1", port=PORT):
    try:
        with socket.create_connection((host, port), timeout=2):
            return True
    except OSError:
        return False


def probe(url, context=None):
    try:
        urllib.request.urlopen(url, timeout=4, context=context)
        return True
    except urllib.error.HTTPError:
        return True  # server answered, just with an error status
    except Exception:
        return False


# ---- Steps: each returns (ok, message, next_action_hint) ----

def step_git(ctx):
    if shutil.which("git"):
        return True, "Git is installed.", ""
    return False, "Git is not installed.", pick(
        "Run: pkg install git  -- then press Run setup again.",
        "Run: winget install --id Git.Git -e  -- then press Run setup again.")


def step_clone(ctx):
    os.makedirs(BASE_DIR, exist_ok=True)
    if os.path.isdir(os.path.join(REPO_DIR, ".git")):
        code, out = run(["git", "-C", REPO_DIR, "pull", "--ff-only"], timeout=120)
        if code == 0:
            return True, "Repository is up to date.", ""
        return False, "Could not update the repository.", (
            "Local changes may conflict. Check the phylum folder, then press Run setup again. "
            + out[-200:])
    code, out = run(["git", "clone", REPO_URL, REPO_DIR], timeout=300)
    if code == 0:
        return True, "Repository cloned.", ""
    return False, "Clone failed.", "Check your internet connection, then press Run setup again. " + out[-200:]


def step_env(ctx):
    example = os.path.join(REPO_DIR, "env.example")
    target = os.path.join(REPO_DIR, ".env")
    if os.path.exists(target):
        return True, ".env already exists and was left untouched.", ""
    if not os.path.exists(example):
        return False, "env.example was not found in the repository.", "Check the repository layout in docs/."
    shutil.copy(example, target)
    return True, ".env was created from env.example.", (
        "Open the .env file in the phylum folder and fill in any required values.")


def step_server(ctx):
    if port_open():
        return True, f"Phylum is listening on port {PORT}.", ""
    docker_hint = ("Press 'Start server (Docker)' below."
                   if shutil.which("docker") else
                   "Start Phylum manually (Docker is not installed or not on PATH).")
    return False, f"Nothing is listening on port {PORT}.", (
        docker_hint + " Then press Run setup again.")


def step_protocol(ctx):
    insecure = ssl.create_default_context()
    insecure.check_hostname = False
    insecure.verify_mode = ssl.CERT_NONE
    if probe(f"https://127.0.0.1:{PORT}/", context=insecure):
        scheme = "https+insecure"
    elif probe(f"http://127.0.0.1:{PORT}/"):
        scheme = "http"
    else:
        return False, "The port is open but neither HTTP nor HTTPS answered.", (
            "Wait 30 seconds and press Run setup again. Check the server log if it persists.")
    ctx["target"] = f"{scheme}://127.0.0.1:{PORT}"
    label = "HTTPS (self-signed)" if scheme.startswith("https") else "HTTP"
    return True, f"Server answers over {label}.", ""


def tailscale_status(ts):
    code, out = run([ts, "status", "--json"], timeout=15)
    if code != 0:
        return None, out
    try:
        data = json.loads(out)
    except json.JSONDecodeError:
        return None, "Unexpected output from tailscale status."
    return {"state": data.get("BackendState", ""),
            "dns": (data.get("Self") or {}).get("DNSName", "").rstrip(".")}, ""


def step_tailscale(ctx):
    ts = find_tailscale()
    if not ts:
        return False, "The Tailscale command was not found.", pick(
            "Run: pkg install tailscale, start tailscaled, then press Run setup again.",
            "Install Tailscale from tailscale.com/download, sign in, then press Run setup again.")
    info, err = tailscale_status(ts)
    if info is None:
        return False, "Tailscale is installed but not responding.", (
            "Open Tailscale and make sure it is running. " + err[-200:])
    if info["state"] != "Running" or not info["dns"]:
        return False, f"Tailscale state is '{info['state']}'.", (
            "Turn Tailscale on and sign in if asked, then press Run setup again.")
    ctx["ts"] = ts
    ctx["dns"] = info["dns"]
    return True, f"Tailscale is running as {info['dns']}.", ""


def step_serve(ctx):
    code, out = run([ctx["ts"], "serve", "--bg", "--https=443", ctx["target"]], timeout=30)
    if code != 0:
        hint = out[-300:] or "Unknown error."
        if IS_WINDOWS:
            hint += " Try running PowerShell as Administrator."
        return False, "tailscale serve failed.", hint
    return True, f"Serving {ctx['target']} on tailnet port 443.", (
        "If Mullvad is connected here, turn off 'Block connections without VPN' "
        "or exclude Tailscale from the VPN so the tailnet keeps working.")


def step_verify(ctx):
    url = f"https://{ctx['dns']}"
    if probe(url):
        ctx["url"] = url
        return True, f"Tailnet URL is reachable: {url}", ""
    return False, "The tailnet URL is not reachable from this machine yet.", (
        "Wait a minute for the certificate to be issued, then press Run setup again.")


class SetupWizard:
    def __init__(self, root):
        self.root = root
        self.ctx = {}
        self.events = queue.Queue()
        self.busy = False
        self.next_index = 0
        self.steps = [
            ("Check Git", step_git),
            ("Clone or update repo", step_clone),
            ("Prepare .env file", step_env),
            ("Check Phylum server", step_server),
            ("Detect server protocol", step_protocol),
            ("Check Tailscale", step_tailscale),
            ("Publish with tailscale serve", step_serve),
            ("Verify tailnet URL", step_verify),
        ]
        root.title("Phylum Setup Wizard")
        root.geometry("900x640")
        root.minsize(760, 560)

        top = ttk.Frame(root, padding=10)
        top.pack(fill="both", expand=True)

        self.tree = ttk.Treeview(top, columns=("status", "step"), show="headings", height=9)
        self.tree.heading("status", text="Status")
        self.tree.heading("step", text="Step")
        self.tree.column("status", width=150, anchor="center")
        self.tree.column("step", width=600)
        self.tree.tag_configure("ok", foreground="#1a7f37")
        self.tree.tag_configure("fail", foreground="#c62828")
        self.tree.tag_configure("run", foreground="#1565c0")
        for name, _ in self.steps:
            self.tree.insert("", "end", values=("Pending", name))
        self.rows = self.tree.get_children()
        self.tree.pack(fill="x")

        self.action = tk.Label(top, text="Press 'Run setup' to begin.", justify="left", anchor="w",
                               wraplength=840, font=("Helvetica", 11, "bold"), pady=10)
        self.action.pack(fill="x")

        self.log = ScrolledText(top, height=14, state="disabled", wrap="word")
        self.log.pack(fill="both", expand=True)

        bar = ttk.Frame(top)
        bar.pack(fill="x", pady=(8, 0))
        self.run_btn = ttk.Button(bar, text="Run setup", command=self.start)
        self.server_btn = ttk.Button(bar, text="Start server (Docker)", command=self.start_server)
        self.copy_btn = ttk.Button(bar, text="Copy tailnet URL", command=self.copy_url, state="disabled")
        quit_btn = ttk.Button(bar, text="Quit", command=root.destroy)
        for b in (self.run_btn, self.server_btn, self.copy_btn, quit_btn):
            b.pack(side="left", padx=4)

        self.root.after(100, self.poll)

    def set_row(self, i, status, tag=""):
        self.tree.item(self.rows[i], values=(status, self.steps[i][0]), tags=(tag,) if tag else ())

    def set_buttons(self, enabled):
        state = "normal" if enabled else "disabled"
        self.run_btn.config(state=state)
        self.server_btn.config(state=state)

    def write_log(self, text):
        self.log.config(state="normal")
        self.log.insert("end", text + "\n")
        self.log.see("end")
        self.log.config(state="disabled")
        try:
            os.makedirs(BASE_DIR, exist_ok=True)
            with open(LOG_PATH, "a", encoding="utf-8") as f:
                f.write(text + "\n")
        except OSError:
            pass

    def start(self):
        if self.busy:
            return
        self.busy = True
        self.set_buttons(False)
        threading.Thread(target=self.worker, args=(self.next_index,), daemon=True).start()

    def worker(self, start):
        for i in range(start, len(self.steps)):
            name, fn = self.steps[i]
            self.events.put(("running", i))
            try:
                ok, msg, hint = fn(self.ctx)
            except Exception as exc:
                ok, msg, hint = False, f"Unexpected error: {exc}", "Press Run setup again. Details are in the log."
            self.events.put(("result", i, ok, msg, hint))
            if not ok:
                self.events.put(("stopped", i))
                return
        self.events.put(("finished",))

    def start_server(self):
        if self.busy:
            return
        if not shutil.which("docker"):
            self.action.config(text="Docker is not installed or not on PATH. Start Phylum manually, "
                                    "then press Run setup.")
            return
        self.busy = True
        self.set_buttons(False)

        def job():
            compose = os.path.join(REPO_DIR, "compose.yml")
            example = os.path.join(REPO_DIR, "compose.example.yml")
            if not os.path.exists(compose) and os.path.exists(example):
                shutil.copy(example, compose)
                self.events.put(("log", "Created compose.yml from compose.example.yml."))
            code, out = run(["docker", "compose", "up", "-d"], cwd=REPO_DIR, timeout=600)
            self.events.put(("log", f"docker compose up -d finished with exit code {code}.\n{out[-800:]}"))
            self.events.put(("idle",))

        threading.Thread(target=job, daemon=True).start()

    def copy_url(self):
        url = self.ctx.get("url")
        if url:
            self.root.clipboard_clear()
            self.root.clipboard_append(url)
            self.write_log(f"Copied {url} to the clipboard.")

    def finish_busy(self):
        self.busy = False
        self.set_buttons(True)

    def poll(self):
        try:
            while True:
                ev = self.events.get_nowait()
                kind = ev[0]
                if kind == "running":
                    i = ev[1]
                    self.set_row(i, "Running...", "run")
                    self.action.config(text=f"Working on: {self.steps[i][0]}...")
                elif kind == "result":
                    _, i, ok, msg, hint = ev
                    self.set_row(i, "OK" if ok else "Needs attention", "ok" if ok else "fail")
                    self.write_log(f"[{'OK' if ok else 'FAIL'}] {self.steps[i][0]}: {msg}")
                    if hint:
                        self.write_log(f"    Next: {hint}")
                    self.action.config(text=f"{msg}\n\nNext: {hint}" if hint else msg)
                elif kind == "stopped":
                    self.next_index = ev[1]
                    self.finish_busy()
                elif kind == "finished":
                    self.next_index = len(self.steps)
                    self.finish_busy()
                    url = self.ctx.get("url", "")
                    self.action.config(text=(
                        "Setup complete.\n\nOn your phone: install Tailscale, sign in to the same account, "
                        f"open the Phylum app, and enter {url} (no port number). Then tap Connect."))
                    self.copy_btn.config(state="normal")
                elif kind == "log":
                    self.write_log(ev[1])
                elif kind == "idle":
                    self.finish_busy()
                    self.action.config(text="Server start finished. Press Run setup to re-check.")
        except queue.Empty:
            pass
        self.root.after(100, self.poll)


def main():
    os.makedirs(BASE_DIR, exist_ok=True)
    try:
        root = tk.Tk()
    except tk.TclError as exc:
        sys.exit(f"No display available ({exc}). On Termux, start termux-x11 and run: export DISPLAY=:0")
    SetupWizard(root)
    root.mainloop()


if __name__ == "__main__":
    main()
PYEOF

python ~/phylum-setup/phylum_setup_gui.py
```

---

## Windows

Run this in PowerShell. It creates the folder, writes the same wizard with a here-string, and launches it with the Python launcher. Python from python.org must be installed with the tcl/tk option, which is on by default.

```powershell
$dir = Join-Path $env:USERPROFILE "phylum-setup"
New-Item -ItemType Directory -Force -Path $dir | Out-Null
$script = Join-Path $dir "phylum_setup_gui.py"
@'
#!/usr/bin/env python3
"""Phylum setup wizard: GUI with step-by-step guidance. Standard library only."""
import json
import os
import platform
import queue
import shutil
import socket
import ssl
import subprocess
import sys
import threading
import urllib.error
import urllib.request

try:
    import tkinter as tk
    from tkinter import ttk
    from tkinter.scrolledtext import ScrolledText
except ImportError:
    sys.exit("tkinter is missing. Termux: pkg install python-tkinter | Windows: reinstall Python with the tcl/tk option.")

PORT = 2448
REPO_URL = "https://codeberg.org/shroff/phylum.git"
IS_WINDOWS = platform.system() == "Windows"
IS_TERMUX = "TERMUX_VERSION" in os.environ or "com.termux" in sys.prefix
CREATE_NO_WINDOW = 0x08000000 if IS_WINDOWS else 0
BASE_DIR = os.path.join(os.path.expanduser("~"), "phylum-setup")
REPO_DIR = os.path.join(BASE_DIR, "phylum")
LOG_PATH = os.path.join(BASE_DIR, "setup.log")


def run(cmd, cwd=None, timeout=60):
    try:
        p = subprocess.run(cmd, cwd=cwd, capture_output=True, text=True, errors="replace",
                           timeout=timeout, creationflags=CREATE_NO_WINDOW)
        return p.returncode, (p.stdout + p.stderr).strip()
    except FileNotFoundError:
        return 127, f"Command not found: {cmd[0]}"
    except subprocess.TimeoutExpired:
        return 124, f"Timed out after {timeout}s"


def pick(termux_text, windows_text):
    return termux_text if IS_TERMUX else windows_text


def find_tailscale():
    found = shutil.which("tailscale")
    if found:
        return found
    if IS_WINDOWS:
        for p in (r"C:\Program Files\Tailscale\tailscale.exe",
                  r"C:\Program Files (x86)\Tailscale\tailscale.exe"):
            if os.path.exists(p):
                return p
    return None


def port_open(host="127.0.0.1", port=PORT):
    try:
        with socket.create_connection((host, port), timeout=2):
            return True
    except OSError:
        return False


def probe(url, context=None):
    try:
        urllib.request.urlopen(url, timeout=4, context=context)
        return True
    except urllib.error.HTTPError:
        return True  # server answered, just with an error status
    except Exception:
        return False


# ---- Steps: each returns (ok, message, next_action_hint) ----

def step_git(ctx):
    if shutil.which("git"):
        return True, "Git is installed.", ""
    return False, "Git is not installed.", pick(
        "Run: pkg install git  -- then press Run setup again.",
        "Run: winget install --id Git.Git -e  -- then press Run setup again.")


def step_clone(ctx):
    os.makedirs(BASE_DIR, exist_ok=True)
    if os.path.isdir(os.path.join(REPO_DIR, ".git")):
        code, out = run(["git", "-C", REPO_DIR, "pull", "--ff-only"], timeout=120)
        if code == 0:
            return True, "Repository is up to date.", ""
        return False, "Could not update the repository.", (
            "Local changes may conflict. Check the phylum folder, then press Run setup again. "
            + out[-200:])
    code, out = run(["git", "clone", REPO_URL, REPO_DIR], timeout=300)
    if code == 0:
        return True, "Repository cloned.", ""
    return False, "Clone failed.", "Check your internet connection, then press Run setup again. " + out[-200:]


def step_env(ctx):
    example = os.path.join(REPO_DIR, "env.example")
    target = os.path.join(REPO_DIR, ".env")
    if os.path.exists(target):
        return True, ".env already exists and was left untouched.", ""
    if not os.path.exists(example):
        return False, "env.example was not found in the repository.", "Check the repository layout in docs/."
    shutil.copy(example, target)
    return True, ".env was created from env.example.", (
        "Open the .env file in the phylum folder and fill in any required values.")


def step_server(ctx):
    if port_open():
        return True, f"Phylum is listening on port {PORT}.", ""
    docker_hint = ("Press 'Start server (Docker)' below."
                   if shutil.which("docker") else
                   "Start Phylum manually (Docker is not installed or not on PATH).")
    return False, f"Nothing is listening on port {PORT}.", (
        docker_hint + " Then press Run setup again.")


def step_protocol(ctx):
    insecure = ssl.create_default_context()
    insecure.check_hostname = False
    insecure.verify_mode = ssl.CERT_NONE
    if probe(f"https://127.0.0.1:{PORT}/", context=insecure):
        scheme = "https+insecure"
    elif probe(f"http://127.0.0.1:{PORT}/"):
        scheme = "http"
    else:
        return False, "The port is open but neither HTTP nor HTTPS answered.", (
            "Wait 30 seconds and press Run setup again. Check the server log if it persists.")
    ctx["target"] = f"{scheme}://127.0.0.1:{PORT}"
    label = "HTTPS (self-signed)" if scheme.startswith("https") else "HTTP"
    return True, f"Server answers over {label}.", ""


def tailscale_status(ts):
    code, out = run([ts, "status", "--json"], timeout=15)
    if code != 0:
        return None, out
    try:
        data = json.loads(out)
    except json.JSONDecodeError:
        return None, "Unexpected output from tailscale status."
    return {"state": data.get("BackendState", ""),
            "dns": (data.get("Self") or {}).get("DNSName", "").rstrip(".")}, ""


def step_tailscale(ctx):
    ts = find_tailscale()
    if not ts:
        return False, "The Tailscale command was not found.", pick(
            "Run: pkg install tailscale, start tailscaled, then press Run setup again.",
            "Install Tailscale from tailscale.com/download, sign in, then press Run setup again.")
    info, err = tailscale_status(ts)
    if info is None:
        return False, "Tailscale is installed but not responding.", (
            "Open Tailscale and make sure it is running. " + err[-200:])
    if info["state"] != "Running" or not info["dns"]:
        return False, f"Tailscale state is '{info['state']}'.", (
            "Turn Tailscale on and sign in if asked, then press Run setup again.")
    ctx["ts"] = ts
    ctx["dns"] = info["dns"]
    return True, f"Tailscale is running as {info['dns']}.", ""


def step_serve(ctx):
    code, out = run([ctx["ts"], "serve", "--bg", "--https=443", ctx["target"]], timeout=30)
    if code != 0:
        hint = out[-300:] or "Unknown error."
        if IS_WINDOWS:
            hint += " Try running PowerShell as Administrator."
        return False, "tailscale serve failed.", hint
    return True, f"Serving {ctx['target']} on tailnet port 443.", (
        "If Mullvad is connected here, turn off 'Block connections without VPN' "
        "or exclude Tailscale from the VPN so the tailnet keeps working.")


def step_verify(ctx):
    url = f"https://{ctx['dns']}"
    if probe(url):
        ctx["url"] = url
        return True, f"Tailnet URL is reachable: {url}", ""
    return False, "The tailnet URL is not reachable from this machine yet.", (
        "Wait a minute for the certificate to be issued, then press Run setup again.")


class SetupWizard:
    def __init__(self, root):
        self.root = root
        self.ctx = {}
        self.events = queue.Queue()
        self.busy = False
        self.next_index = 0
        self.steps = [
            ("Check Git", step_git),
            ("Clone or update repo", step_clone),
            ("Prepare .env file", step_env),
            ("Check Phylum server", step_server),
            ("Detect server protocol", step_protocol),
            ("Check Tailscale", step_tailscale),
            ("Publish with tailscale serve", step_serve),
            ("Verify tailnet URL", step_verify),
        ]
        root.title("Phylum Setup Wizard")
        root.geometry("900x640")
        root.minsize(760, 560)

        top = ttk.Frame(root, padding=10)
        top.pack(fill="both", expand=True)

        self.tree = ttk.Treeview(top, columns=("status", "step"), show="headings", height=9)
        self.tree.heading("status", text="Status")
        self.tree.heading("step", text="Step")
        self.tree.column("status", width=150, anchor="center")
        self.tree.column("step", width=600)
        self.tree.tag_configure("ok", foreground="#1a7f37")
        self.tree.tag_configure("fail", foreground="#c62828")
        self.tree.tag_configure("run", foreground="#1565c0")
        for name, _ in self.steps:
            self.tree.insert("", "end", values=("Pending", name))
        self.rows = self.tree.get_children()
        self.tree.pack(fill="x")

        self.action = tk.Label(top, text="Press 'Run setup' to begin.", justify="left", anchor="w",
                               wraplength=840, font=("Helvetica", 11, "bold"), pady=10)
        self.action.pack(fill="x")

        self.log = ScrolledText(top, height=14, state="disabled", wrap="word")
        self.log.pack(fill="both", expand=True)

        bar = ttk.Frame(top)
        bar.pack(fill="x", pady=(8, 0))
        self.run_btn = ttk.Button(bar, text="Run setup", command=self.start)
        self.server_btn = ttk.Button(bar, text="Start server (Docker)", command=self.start_server)
        self.copy_btn = ttk.Button(bar, text="Copy tailnet URL", command=self.copy_url, state="disabled")
        quit_btn = ttk.Button(bar, text="Quit", command=root.destroy)
        for b in (self.run_btn, self.server_btn, self.copy_btn, quit_btn):
            b.pack(side="left", padx=4)

        self.root.after(100, self.poll)

    def set_row(self, i, status, tag=""):
        self.tree.item(self.rows[i], values=(status, self.steps[i][0]), tags=(tag,) if tag else ())

    def set_buttons(self, enabled):
        state = "normal" if enabled else "disabled"
        self.run_btn.config(state=state)
        self.server_btn.config(state=state)

    def write_log(self, text):
        self.log.config(state="normal")
        self.log.insert("end", text + "\n")
        self.log.see("end")
        self.log.config(state="disabled")
        try:
            os.makedirs(BASE_DIR, exist_ok=True)
            with open(LOG_PATH, "a", encoding="utf-8") as f:
                f.write(text + "\n")
        except OSError:
            pass

    def start(self):
        if self.busy:
            return
        self.busy = True
        self.set_buttons(False)
        threading.Thread(target=self.worker, args=(self.next_index,), daemon=True).start()

    def worker(self, start):
        for i in range(start, len(self.steps)):
            name, fn = self.steps[i]
            self.events.put(("running", i))
            try:
                ok, msg, hint = fn(self.ctx)
            except Exception as exc:
                ok, msg, hint = False, f"Unexpected error: {exc}", "Press Run setup again. Details are in the log."
            self.events.put(("result", i, ok, msg, hint))
            if not ok:
                self.events.put(("stopped", i))
                return
        self.events.put(("finished",))

    def start_server(self):
        if self.busy:
            return
        if not shutil.which("docker"):
            self.action.config(text="Docker is not installed or not on PATH. Start Phylum manually, "
                                    "then press Run setup.")
            return
        self.busy = True
        self.set_buttons(False)

        def job():
            compose = os.path.join(REPO_DIR, "compose.yml")
            example = os.path.join(REPO_DIR, "compose.example.yml")
            if not os.path.exists(compose) and os.path.exists(example):
                shutil.copy(example, compose)
                self.events.put(("log", "Created compose.yml from compose.example.yml."))
            code, out = run(["docker", "compose", "up", "-d"], cwd=REPO_DIR, timeout=600)
            self.events.put(("log", f"docker compose up -d finished with exit code {code}.\n{out[-800:]}"))
            self.events.put(("idle",))

        threading.Thread(target=job, daemon=True).start()

    def copy_url(self):
        url = self.ctx.get("url")
        if url:
            self.root.clipboard_clear()
            self.root.clipboard_append(url)
            self.write_log(f"Copied {url} to the clipboard.")

    def finish_busy(self):
        self.busy = False
        self.set_buttons(True)

    def poll(self):
        try:
            while True:
                ev = self.events.get_nowait()
                kind = ev[0]
                if kind == "running":
                    i = ev[1]
                    self.set_row(i, "Running...", "run")
                    self.action.config(text=f"Working on: {self.steps[i][0]}...")
                elif kind == "result":
                    _, i, ok, msg, hint = ev
                    self.set_row(i, "OK" if ok else "Needs attention", "ok" if ok else "fail")
                    self.write_log(f"[{'OK' if ok else 'FAIL'}] {self.steps[i][0]}: {msg}")
                    if hint:
                        self.write_log(f"    Next: {hint}")
                    self.action.config(text=f"{msg}\n\nNext: {hint}" if hint else msg)
                elif kind == "stopped":
                    self.next_index = ev[1]
                    self.finish_busy()
                elif kind == "finished":
                    self.next_index = len(self.steps)
                    self.finish_busy()
                    url = self.ctx.get("url", "")
                    self.action.config(text=(
                        "Setup complete.\n\nOn your phone: install Tailscale, sign in to the same account, "
                        f"open the Phylum app, and enter {url} (no port number). Then tap Connect."))
                    self.copy_btn.config(state="normal")
                elif kind == "log":
                    self.write_log(ev[1])
                elif kind == "idle":
                    self.finish_busy()
                    self.action.config(text="Server start finished. Press Run setup to re-check.")
        except queue.Empty:
            pass
        self.root.after(100, self.poll)


def main():
    os.makedirs(BASE_DIR, exist_ok=True)
    try:
        root = tk.Tk()
    except tk.TclError as exc:
        sys.exit(f"No display available ({exc}). On Termux, start termux-x11 and run: export DISPLAY=:0")
    SetupWizard(root)
    root.mainloop()


if __name__ == "__main__":
    main()
'@ | Set-Content -Path $script -Encoding UTF8

py -3 $script
```

---

## Notes on using the wizard

The wizard stops at the first failed step and tells you what to fix, then you press **Run setup** again and it resumes from that step. Every result is also written to `setup.log` in the `phylum-setup` folder, so you can share the exact error if something fails.

On the phone, use the URL the wizard shows, which is `https://<your-machine>.<tailnet>.ts.net` with no `:2448`. Tailscale serves it on port 443, so the port number is not needed. Remember that the Phylum app was pointed at `:2448` in the earlier setup, so update the address in the app.

```
mkdir -p ~/phylum-setup
cat > ~/phylum-setup/phylum_setup_gui.py <<'PYEOF'
#!/usr/bin/env python3
"""Phylum setup wizard: GUI with step-by-step guidance. Standard library only."""
import json
import os
import platform
import queue
import shutil
import socket
import ssl
import subprocess
import sys
import threading
import urllib.error
import urllib.request

try:
    import tkinter as tk
    from tkinter import ttk
    from tkinter.scrolledtext import ScrolledText
except ImportError:
    sys.exit("tkinter is missing. Termux: pkg install python-tkinter | Windows: reinstall Python with the tcl/tk option.")

PORT = 2448
REPO_URL = "https://codeberg.org/shroff/phylum.git"
IS_WINDOWS = platform.system() == "Windows"
IS_TERMUX = "TERMUX_VERSION" in os.environ or "com.termux" in sys.prefix
CREATE_NO_WINDOW = 0x08000000 if IS_WINDOWS else 0
BASE_DIR = os.path.join(os.path.expanduser("~"), "phylum-setup")
REPO_DIR = os.path.join(BASE_DIR, "phylum")
LOG_PATH = os.path.join(BASE_DIR, "setup.log")


def run(cmd, cwd=None, timeout=60):
    try:
        p = subprocess.run(cmd, cwd=cwd, capture_output=True, text=True, errors="replace",
                           timeout=timeout, creationflags=CREATE_NO_WINDOW)
        return p.returncode, (p.stdout + p.stderr).strip()
    except FileNotFoundError:
        return 127, f"Command not found: {cmd[0]}"
    except subprocess.TimeoutExpired:
        return 124, f"Timed out after {timeout}s"


def pick(termux_text, windows_text):
    return termux_text if IS_TERMUX else windows_text


def find_tailscale():
    found = shutil.which("tailscale")
    if found:
        return found
    if IS_WINDOWS:
        for p in (r"C:\Program Files\Tailscale\tailscale.exe",
                  r"C:\Program Files (x86)\Tailscale\tailscale.exe"):
            if os.path.exists(p):
                return p
    return None


def port_open(host="127.0.0.1", port=PORT):
    try:
        with socket.create_connection((host, port), timeout=2):
            return True
    except OSError:
        return False


def probe(url, context=None):
    try:
        urllib.request.urlopen(url, timeout=4, context=context)
        return True
    except urllib.error.HTTPError:
        return True  # server answered, just with an error status
    except Exception:
        return False


# ---- Steps: each returns (ok, message, next_action_hint) ----

def step_git(ctx):
    if shutil.which("git"):
        return True, "Git is installed.", ""
    return False, "Git is not installed.", pick(
        "Run: pkg install git  -- then press Run setup again.",
        "Run: winget install --id Git.Git -e  -- then press Run setup again.")


def step_clone(ctx):
    os.makedirs(BASE_DIR, exist_ok=True)
    if os.path.isdir(os.path.join(REPO_DIR, ".git")):
        code, out = run(["git", "-C", REPO_DIR, "pull", "--ff-only"], timeout=120)
        if code == 0:
            return True, "Repository is up to date.", ""
        return False, "Could not update the repository.", (
            "Local changes may conflict. Check the phylum folder, then press Run setup again. "
            + out[-200:])
    code, out = run(["git", "clone", REPO_URL, REPO_DIR], timeout=300)
    if code == 0:
        return True, "Repository cloned.", ""
    return False, "Clone failed.", "Check your internet connection, then press Run setup again. " + out[-200:]


def step_env(ctx):
    example = os.path.join(REPO_DIR, "env.example")
    target = os.path.join(REPO_DIR, ".env")
    if os.path.exists(target):
        return True, ".env already exists and was left untouched.", ""
    if not os.path.exists(example):
        return False, "env.example was not found in the repository.", "Check the repository layout in docs/."
    shutil.copy(example, target)
    return True, ".env was created from env.example.", "Open the .env file in the phylum folder and fill in any required values."


def step_server(ctx):
    if port_open():
        return True, f"Phylum is listening on port {PORT}.", ""
    docker_hint = ("Press 'Start server (Docker)' below."
                   if shutil.which("docker") else
                   "Start Phylum manually (Docker is not installed or not on PATH).")
    return False, f"Nothing is listening on port {PORT}.", docker_hint + " Then press Run setup again."


def step_protocol(ctx):
    insecure = ssl.create_default_context()
    insecure.check_hostname = False
    insecure.verify_mode = ssl.CERT_NONE
    if probe(f"https://127.0.0.1:{PORT}/", context=insecure):
        scheme = "https+insecure"
        label = "
