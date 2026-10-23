"""
WiFi Hotspot Pro v2.0 — Công cụ phát WiFi Hotspot cho Windows
==============================================================
Hỗ trợ cả Mobile Hotspot (Windows 10 1607+) và Hosted Network (legacy).
Tự động phát hiện phương thức phù hợp nhất cho máy của bạn.

Tính năng:
  • Dual engine: Mobile Hotspot + Hosted Network (tự động fallback)
  • Tự phục hồi khi mất kết nối (watchdog)
  • Ẩn xuống khay hệ thống khi đóng cửa sổ
  • Tự chạy cùng Windows (Task Scheduler, quyền Admin)
  • Ghi log đầy đủ vào %APPDATA% để debug
  • Hiển thị số thiết bị đang kết nối
  • Giao diện hiện đại (dark theme)

Cài đặt:  pip install pystray pillow pywin32
Chạy:     pythonw wifi_hotspot.py
Đóng gói: pyinstaller --noconsole --onefile --uac-admin --name WifiHotspot wifi_hotspot.py
"""

import ctypes
import json
import logging
import logging.handlers
import os
import queue
import random
import re
import string
import subprocess
import sys
import threading
import time
import tkinter as tk
from tkinter import ttk, messagebox

# ----------- optional imports with graceful fallback -----------
try:
    import pystray
    from PIL import Image, ImageDraw
    HAS_TRAY = True
except ImportError:
    HAS_TRAY = False
    print("[WARN] pystray/Pillow chưa cài. Chạy: pip install pystray pillow")

try:
    import pythoncom
    import win32com.client
    HAS_PYWIN32 = True
except ImportError:
    HAS_PYWIN32 = False
    print("[WARN] pywin32 chưa cài. Chạy: pip install pywin32")


# ================================================================ CONSTANTS
APP_NAME = "WifiHotspot"
TASK_NAME = "WifiHotspotAutoStart"
VERSION = "2.0.0"
NO_WINDOW = 0x08000000

CFG_DIR = os.path.join(os.environ.get("APPDATA", "."), APP_NAME)
CFG_FILE = os.path.join(CFG_DIR, "config.json")
LOG_FILE = os.path.join(CFG_DIR, "hotspot.log")

ENGINE_MOBILE = "mobile"
ENGINE_HOSTED = "hosted"
ENGINE_NONE   = "none"

# Dark-theme colour palette
C = {
    "bg":       "#0f1023",
    "surface":  "#1a1b35",
    "surface2": "#24254a",
    "primary":  "#6c5ce7",
    "primary_h":"#8577f0",
    "success":  "#00cec9",
    "error":    "#ff7675",
    "warning":  "#ffeaa7",
    "text":     "#dfe6e9",
    "text2":    "#a0a4b8",
    "border":   "#34355a",
    "entry":    "#151630",
}


# ================================================================ LOGGING
os.makedirs(CFG_DIR, exist_ok=True)

log = logging.getLogger(APP_NAME)
log.setLevel(logging.DEBUG)

_fh = logging.handlers.RotatingFileHandler(
    LOG_FILE, maxBytes=2 * 1024 * 1024, backupCount=3, encoding="utf-8")
_fh.setFormatter(logging.Formatter(
    "%(asctime)s [%(levelname)-7s] %(message)s", datefmt="%Y-%m-%d %H:%M:%S"))
log.addHandler(_fh)

_sh = logging.StreamHandler()
_sh.setLevel(logging.WARNING)
log.addHandler(_sh)

log.info("=" * 60)
log.info("WiFi Hotspot Pro v%s  —  Python %s  —  frozen=%s",
         VERSION, sys.version.split()[0], getattr(sys, "frozen", False))


# ================================================================ UTILITIES

def is_admin():
    try:
        return bool(ctypes.windll.shell32.IsUserAnAdmin())
    except Exception:
        return False


def relaunch_as_admin():
    try:
        if getattr(sys, "frozen", False):
            exe, params = sys.executable, " ".join(f'"{a}"' for a in sys.argv[1:])
        else:
            exe = sys.executable
            pyw = os.path.join(os.path.dirname(exe), "pythonw.exe")
            if os.path.exists(pyw):
                exe = pyw
            params = " ".join(f'"{a}"' for a in sys.argv)
        ctypes.windll.shell32.ShellExecuteW(None, "runas", exe, params, None, 1)
    except Exception as e:
        log.error("Không thể yêu cầu quyền Admin: %s", e)


def single_instance():
    ctypes.windll.kernel32.CreateMutexW(None, False, "Global\\WifiHotspotProMtx")
    if ctypes.windll.kernel32.GetLastError() == 183:
        log.warning("Đã có instance khác đang chạy")
        return False
    return True


def run_cmd(cmd, timeout=30):
    """Chạy lệnh hệ thống, trả về (code, output_str)."""
    try:
        r = subprocess.run(cmd, capture_output=True, timeout=timeout,
                           creationflags=NO_WINDOW)
        enc = "cp%d" % ctypes.windll.kernel32.GetOEMCP()
        return r.returncode, (r.stdout + r.stderr).decode(enc, errors="ignore")
    except subprocess.TimeoutExpired:
        log.warning("Lệnh timeout: %s", cmd)
        return -1, "Timeout"
    except Exception as e:
        log.error("Lệnh lỗi: %s — %s", cmd, e)
        return -1, str(e)


def run_ps(script, timeout=45):
    """Chạy PowerShell script, trả về (code, stdout, stderr)."""
    try:
        r = subprocess.run(
            ["powershell", "-NoProfile", "-NonInteractive",
             "-ExecutionPolicy", "Bypass", "-Command", script],
            capture_output=True, text=True, timeout=timeout,
            creationflags=NO_WINDOW)
        return r.returncode, r.stdout.strip(), r.stderr.strip()
    except subprocess.TimeoutExpired:
        return -1, "", "PowerShell timeout"
    except FileNotFoundError:
        return -1, "", "PowerShell không tìm thấy"
    except Exception as e:
        return -1, "", str(e)


def gen_password(length=10):
    cs = string.ascii_letters + string.digits
    return "".join(random.choice(cs) for _ in range(length))


def load_cfg():
    d = {"ssid": "MyHotspot", "password": "",
         "source": "", "engine": "",
         "win_autostart": False, "auto_hotspot": True}
    try:
        with open(CFG_FILE, "r", encoding="utf-8") as f:
            d.update(json.load(f))
    except FileNotFoundError:
        pass
    except Exception as e:
        log.warning("Đọc config lỗi: %s", e)
    return d


def save_cfg(d):
    os.makedirs(CFG_DIR, exist_ok=True)
    try:
        with open(CFG_FILE, "w", encoding="utf-8") as f:
            json.dump(d, f, ensure_ascii=False, indent=2)
    except Exception as e:
        log.error("Ghi config lỗi: %s", e)


# ================================================================ SYSTEM CHECKS

def check_wifi_adapter():
    """Kiểm tra WiFi adapter. Trả về (ok, msg)."""
    code, out = run_cmd(["netsh", "wlan", "show", "interfaces"])
    if code != 0:
        return False, "Không thể kiểm tra WiFi adapter"
    low = out.lower()
    if "no wireless" in low or "không có" in low or "there is no" in low:
        return False, "Không tìm thấy WiFi adapter — kiểm tra driver"
    return True, "OK"


def check_hosted_network_support():
    """Kiểm tra driver có hỗ trợ Hosted Network không."""
    code, out = run_cmd(["netsh", "wlan", "show", "drivers"])
    if code != 0:
        return False, "Không đọc được thông tin driver"
    # Tìm dòng "Hosted network supported"
    for line in out.splitlines():
        low = line.lower()
        if "hosted network" in low or "lưu trữ" in low or "承载网络" in low:
            val = line.split(":")[-1].strip().lower() if ":" in line else ""
            if val in ("yes", "có", "是", "oui", "ja"):
                return True, "Driver hỗ trợ Hosted Network"
            return False, "Driver WiFi không hỗ trợ Hosted Network"
    return False, "Không xác định được hỗ trợ Hosted Network"


def check_mobile_hotspot():
    """Kiểm tra Mobile Hotspot (Windows 10+) khả dụng không."""
    script = r"""
$ErrorActionPreference = 'Stop'
try {
    [Windows.Networking.NetworkOperators.NetworkOperatorTetheringManager,Windows.Networking.NetworkOperators,ContentType=WindowsRuntime] | Out-Null
    [Windows.Networking.Connectivity.NetworkInformation,Windows.Networking.Connectivity,ContentType=WindowsRuntime] | Out-Null
    $prof = [Windows.Networking.Connectivity.NetworkInformation]::GetInternetConnectionProfile()
    if ($null -eq $prof) { Write-Output 'NO_INTERNET'; exit 0 }
    $cap = [Windows.Networking.NetworkOperators.NetworkOperatorTetheringManager]::GetTetheringCapability($prof)
    Write-Output $cap
} catch {
    Write-Output "ERR:$($_.Exception.Message)"
    exit 1
}
"""
    code, stdout, stderr = run_ps(script, timeout=20)
    log.debug("Mobile check: code=%d out='%s' err='%s'", code, stdout, stderr)
    if stdout == "Enabled":
        return True, "Mobile Hotspot sẵn sàng"
    if stdout == "NO_INTERNET":
        return False, "Chưa có kết nối Internet để kiểm tra Mobile Hotspot"
    return False, f"Mobile Hotspot: {stdout or stderr}"


# ================================================================ POWERSHELL WinRT PRELUDE
# Dùng chung cho mọi thao tác Mobile Hotspot
_PS_PRELUDE = r"""
$ErrorActionPreference = 'Stop'
Add-Type -AssemblyName System.Runtime.WindowsRuntime

$ms = [System.WindowsRuntimeSystemExtensions].GetMethods()
$_aop = ($ms | Where-Object {
    $_.Name -eq 'AsTask' -and $_.GetParameters().Count -eq 1 -and
    $_.GetParameters()[0].ParameterType.Name -eq 'IAsyncOperation`1' })[0]
$_aac = ($ms | Where-Object {
    $_.Name -eq 'AsTask' -and $_.GetParameters().Count -eq 1 -and
    $_.GetParameters()[0].ParameterType.Name -eq 'IAsyncAction' })[0]

function AwaitOp($t,$r){ $a=$_aop.MakeGenericMethod($r); $n=$a.Invoke($null,@($t)); $n.Wait(30000)|Out-Null; $n.Result }
function AwaitAc($t){ $n=$_aac.Invoke($null,@($t)); $n.Wait(30000)|Out-Null }

[Windows.Networking.NetworkOperators.NetworkOperatorTetheringManager,Windows.Networking.NetworkOperators,ContentType=WindowsRuntime] | Out-Null
[Windows.Networking.Connectivity.NetworkInformation,Windows.Networking.Connectivity,ContentType=WindowsRuntime] | Out-Null

$_prof = [Windows.Networking.Connectivity.NetworkInformation]::GetInternetConnectionProfile()
if ($null -eq $_prof) { throw 'Không tìm thấy kết nối Internet' }
$_mgr = [Windows.Networking.NetworkOperators.NetworkOperatorTetheringManager]::CreateFromConnectionProfile($_prof)
"""


# ================================================================ MOBILE HOTSPOT ENGINE

class MobileHotspot:
    """Engine dùng Windows 10+ Tethering API qua PowerShell."""

    @staticmethod
    def start(ssid, password):
        safe_ssid = ssid.replace("'", "''")
        safe_pw   = password.replace("'", "''")
        script = _PS_PRELUDE + f"""
try {{
    $cfg = $_mgr.GetCurrentAccessPointConfiguration()
    $cfg.Ssid = '{safe_ssid}'
    $cfg.Passphrase = '{safe_pw}'
    AwaitAc($_mgr.ConfigureAccessPointAsync($cfg))

    if ($_mgr.TetheringOperationalState -ne 'InUse') {{
        $r = AwaitOp ($_mgr.StartTetheringAsync()) ([Windows.Networking.NetworkOperators.NetworkOperatorTetheringOperationResult])
        Write-Output "S:$($r.Status)"
        if ($r.Status -ne 'Success') {{ Write-Output "M:$($r.AdditionalErrorMessage)" }}
    }} else {{
        Write-Output 'S:Success'
    }}
}} catch {{
    Write-Output "S:Error"
    Write-Output "M:$($_.Exception.Message)"
}}
"""
        log.info("Mobile Hotspot: bắt đầu phát SSID=%s", ssid)
        code, stdout, stderr = run_ps(script, timeout=60)
        log.debug("start result: code=%d out='%s'", code, stdout)

        status, msg = "", ""
        for ln in stdout.splitlines():
            if ln.startswith("S:"): status = ln[2:]
            if ln.startswith("M:"): msg = ln[2:]

        if status == "Success":
            log.info("Mobile Hotspot: đang phát thành công")
            return True, "Đang phát  ·  Mobile Hotspot"
        log.error("Mobile Hotspot start lỗi: %s — %s", status, msg or stderr)
        return False, msg or status or stderr or "Lỗi không xác định"

    @staticmethod
    def stop():
        script = _PS_PRELUDE + r"""
try {
    if ($_mgr.TetheringOperationalState -eq 'InUse') {
        $r = AwaitOp ($_mgr.StopTetheringAsync()) ([Windows.Networking.NetworkOperators.NetworkOperatorTetheringOperationResult])
        Write-Output "S:$($r.Status)"
    } else { Write-Output 'S:Success' }
} catch {
    Write-Output "S:Error"
    Write-Output "M:$($_.Exception.Message)"
}
"""
        log.info("Mobile Hotspot: dừng phát")
        code, stdout, _ = run_ps(script, timeout=30)
        for ln in stdout.splitlines():
            if ln.startswith("S:") and ln[2:] == "Success":
                return True, "Đã dừng"
        return False, "Lỗi dừng Mobile Hotspot"

    @staticmethod
    def is_running():
        script = _PS_PRELUDE + "Write-Output $_mgr.TetheringOperationalState"
        _, stdout, _ = run_ps(script, timeout=15)
        return stdout.strip() == "InUse"

    @staticmethod
    def client_count():
        script = _PS_PRELUDE + r"""
try { $c=$_mgr.GetTetheringClients(); Write-Output $c.Count } catch { Write-Output 0 }
"""
        _, stdout, _ = run_ps(script, timeout=15)
        try: return int(stdout.strip())
        except: return 0


# ================================================================ HOSTED NETWORK ENGINE

class HostedNetwork:
    """Engine dùng netsh wlan hostednetwork (legacy)."""

    @staticmethod
    def start(ssid, password, source=""):
        log.info("Hosted Network: bắt đầu phát SSID=%s source=%s", ssid, source)
        code, out = run_cmd(["netsh", "wlan", "set", "hostednetwork",
                             "mode=allow", f"ssid={ssid}", f"key={password}"])
        if code != 0:
            log.error("Cấu hình HN lỗi: %s", out)
            return False, f"Lỗi cấu hình: {out.strip()}"

        if not HostedNetwork.is_running():
            code, out = run_cmd(["netsh", "wlan", "start", "hostednetwork"])
            if code != 0:
                log.error("Khởi động HN lỗi: %s", out)
                return False, f"Lỗi khởi động: {out.strip()}"

        if source:
            time.sleep(2)
            try:
                ok = _setup_ics(source)
                if not ok:
                    return True, "Đang phát  ·  Hosted Network (chưa chia sẻ Internet)"
            except Exception as e:
                log.error("ICS lỗi: %s", e)
                return True, f"Đang phát  ·  Hosted Network (ICS lỗi: {e})"
        log.info("Hosted Network: đang phát")
        return True, "Đang phát  ·  Hosted Network"

    @staticmethod
    def stop():
        log.info("Hosted Network: dừng phát")
        run_cmd(["netsh", "wlan", "stop", "hostednetwork"])
        _disable_ics()
        return True, "Đã dừng"

    @staticmethod
    def is_running():
        _, out = run_cmd(["netsh", "wlan", "show", "hostednetwork"])
        low = out.lower()
        if "started" in low and "not started" not in low:
            return True
        if "已启动" in out:
            return True
        return False

    @staticmethod
    def client_count():
        _, out = run_cmd(["netsh", "wlan", "show", "hostednetwork"])
        for line in out.splitlines():
            if re.search(r"(clients|khách|客户端)", line, re.I):
                m = re.search(r":\s*(\d+)", line)
                if m:
                    return int(m.group(1))
        return 0


# ================================================================ ICS HELPERS

def _is_virtual(dev):
    d = (dev or "").lower()
    return any(k in d for k in (
        "hosted network", "wi-fi direct", "virtual adapter",
        "microsoft hosted", "virtual wifi"))


def list_sources():
    """Liệt kê các kết nối mạng có thể dùng làm nguồn Internet."""
    if not HAS_PYWIN32:
        log.warning("pywin32 chưa cài — không thể liệt kê mạng")
        return []
    pythoncom.CoInitialize()
    try:
        mgr = win32com.client.Dispatch("HNetCfg.HNetShare")
        out = []
        for c in mgr.EnumEveryConnection:
            try:
                p = mgr.NetConnectionProps(c)
                if not _is_virtual(p.DeviceName):
                    out.append((p.Name, p.DeviceName))
            except Exception:
                pass
        return out
    except Exception as e:
        log.error("Liệt kê mạng lỗi: %s", e)
        return []
    finally:
        pythoncom.CoUninitialize()


def _setup_ics(source_name):
    if not HAS_PYWIN32:
        return False
    pythoncom.CoInitialize()
    try:
        mgr = win32com.client.Dispatch("HNetCfg.HNetShare")
        conns = []
        for c in mgr.EnumEveryConnection:
            try:
                conns.append((c, mgr.NetConnectionProps(c)))
            except Exception:
                pass

        # Tắt mọi chia sẻ cũ
        for c, _ in conns:
            try:
                cfg = mgr.INetSharingConfigurationForINetConnection(c)
                if cfg.SharingEnabled:
                    cfg.DisableSharing()
            except Exception:
                pass

        pub = priv = None
        for c, p in conns:
            if p.Name == source_name:
                pub = c
            if _is_virtual(p.DeviceName):
                priv = c

        if pub is None:
            log.error("Không tìm thấy mạng nguồn '%s'", source_name)
            return False
        if priv is None:
            log.error("Không tìm thấy adapter ảo (hosted network)")
            return False

        mgr.INetSharingConfigurationForINetConnection(pub).EnableSharing(0)
        mgr.INetSharingConfigurationForINetConnection(priv).EnableSharing(1)
        log.info("ICS: %s → Hosted Network", source_name)
        return True
    except Exception as e:
        log.error("ICS lỗi: %s", e)
        return False
    finally:
        pythoncom.CoUninitialize()


def _disable_ics():
    if not HAS_PYWIN32:
        return
    pythoncom.CoInitialize()
    try:
        mgr = win32com.client.Dispatch("HNetCfg.HNetShare")
        for c in mgr.EnumEveryConnection:
            try:
                cfg = mgr.INetSharingConfigurationForINetConnection(c)
                if cfg.SharingEnabled:
                    cfg.DisableSharing()
            except Exception:
                pass
    except Exception:
        pass
    finally:
        pythoncom.CoUninitialize()


# ================================================================ HOTSPOT MANAGER

class HotspotManager:
    """Quản lý hotspot — tự chọn engine tốt nhất."""

    def __init__(self):
        self.engine = ENGINE_NONE
        self.engine_label = ""
        self._lock = threading.Lock()

    def detect(self):
        """Phát hiện engine phù hợp. Trả về (engine, msg)."""
        log.info("Đang kiểm tra hệ thống…")

        ok, msg = check_wifi_adapter()
        if not ok:
            self.engine = ENGINE_NONE
            return ENGINE_NONE, msg

        # Ưu tiên Mobile Hotspot (hiện đại hơn, ICS tự động)
        ok, msg_m = check_mobile_hotspot()
        if ok:
            self.engine = ENGINE_MOBILE
            self.engine_label = "Mobile Hotspot"
            log.info("Chọn engine: Mobile Hotspot")
            return ENGINE_MOBILE, "✓  Mobile Hotspot sẵn sàng"

        log.info("Mobile Hotspot không khả dụng: %s", msg_m)

        # Fallback → Hosted Network
        ok, msg_h = check_hosted_network_support()
        if ok:
            self.engine = ENGINE_HOSTED
            self.engine_label = "Hosted Network"
            log.info("Chọn engine: Hosted Network")
            return ENGINE_HOSTED, "✓  Hosted Network (legacy)"

        log.error("Không có engine khả dụng")
        self.engine = ENGINE_NONE
        return ENGINE_NONE, (
            f"Không thể phát WiFi trên máy này.\n"
            f"  • Mobile Hotspot: {msg_m}\n"
            f"  • Hosted Network: {msg_h}")

    def start(self, ssid, pw, source=""):
        with self._lock:
            if self.engine == ENGINE_MOBILE:
                return MobileHotspot.start(ssid, pw)
            if self.engine == ENGINE_HOSTED:
                return HostedNetwork.start(ssid, pw, source)
            return False, "Không có engine"

    def stop(self):
        with self._lock:
            if self.engine == ENGINE_MOBILE:
                return MobileHotspot.stop()
            if self.engine == ENGINE_HOSTED:
                return HostedNetwork.stop()
            return True, ""

    def is_running(self):
        try:
            if self.engine == ENGINE_MOBILE:
                return MobileHotspot.is_running()
            if self.engine == ENGINE_HOSTED:
                return HostedNetwork.is_running()
        except Exception as e:
            log.debug("Kiểm tra trạng thái lỗi: %s", e)
        return False

    def client_count(self):
        try:
            if self.engine == ENGINE_MOBILE:
                return MobileHotspot.client_count()
            if self.engine == ENGINE_HOSTED:
                return HostedNetwork.client_count()
        except Exception:
            pass
        return 0


# ================================================================ AUTOSTART

def set_autostart(enable):
    if not enable:
        run_cmd(["schtasks", "/delete", "/tn", TASK_NAME, "/f"])
        return True
    if getattr(sys, "frozen", False):
        tr = f'"{sys.executable}" --minimized'
    else:
        pyw = os.path.join(os.path.dirname(sys.executable), "pythonw.exe")
        py = pyw if os.path.exists(pyw) else sys.executable
        tr = f'"{py}" "{os.path.abspath(__file__)}" --minimized'
    code, _ = run_cmd(["schtasks", "/create", "/tn", TASK_NAME,
                       "/tr", tr, "/sc", "onlogon",
                       "/rl", "highest", "/delay", "0000:15", "/f"])
    if code == 0:
        log.info("Đã tạo task tự khởi động")
    else:
        log.error("Tạo task tự khởi động thất bại")
    return code == 0


# ================================================================ TRAY ICON

def _make_icon(active=False):
    clr = "#00cec9" if active else "#6c5ce7"
    img = Image.new("RGBA", (64, 64), (0, 0, 0, 0))
    d = ImageDraw.Draw(img)
    d.rounded_rectangle((2, 2, 62, 62), 14, fill=clr)
    for r in (10, 20, 30):
        d.arc((32 - r, 44 - r, 32 + r, 44 + r), 225, 315, fill="white", width=5)
    d.ellipse((28, 40, 36, 48), fill="white")
    return img


# ================================================================ LOG HANDLER → UI

class _LogToUI(logging.Handler):
    """Gửi log entries vào queue để hiển thị trên UI."""
    def __init__(self, q):
        super().__init__()
        self.q = q
        self.setFormatter(logging.Formatter(
            "%(asctime)s  %(message)s", datefmt="%H:%M:%S"))

    def emit(self, record):
        msg = self.format(record)
        self.q.put(("_log", msg))


# ================================================================ APP

class App:
    def __init__(self, minimized=False):
        self.cfg = load_cfg()
        self.q = queue.Queue()
        self.want_run = False
        self.running = False
        self.busy = False
        self.stop_evt = threading.Event()
        self.mgr = HotspotManager()
        self.sources = []

        # ---- window ----
        self.root = tk.Tk()
        self.root.title(f"WiFi Hotspot Pro")
        self.root.configure(bg=C["bg"])
        self.root.resizable(False, False)
        self.root.protocol("WM_DELETE_WINDOW", self._hide)

        try:
            self.root.iconbitmap(default="")
        except Exception:
            pass

        self._build_styles()
        self._build_ui()

        # ---- center on screen ----
        self.root.update_idletasks()
        w, h = self.root.winfo_width(), self.root.winfo_height()
        sw, sh = self.root.winfo_screenwidth(), self.root.winfo_screenheight()
        self.root.geometry(f"+{(sw-w)//2}+{(sh-h)//2}")

        # ---- tray ----
        self.tray = None
        if HAS_TRAY:
            self.tray = pystray.Icon(APP_NAME, _make_icon(False),
                "WiFi Hotspot Pro", pystray.Menu(
                    pystray.MenuItem("Mở", self._tray_show, default=True),
                    pystray.MenuItem(
                        lambda i: "⏹  Dừng" if self.want_run else "▶  Phát",
                        self._tray_toggle),
                    pystray.MenuItem("Thoát", self._tray_quit)))
            self.tray.run_detached()

        # ---- log handler ----
        self._logh = _LogToUI(self.q)
        self._logh.setLevel(logging.INFO)
        log.addHandler(self._logh)

        # ---- background ----
        threading.Thread(target=self._detect_thread, daemon=True).start()
        self.root.after(150, self._poll)

        if minimized:
            self.root.withdraw()

    # -------------------------------------------------------- styles
    def _build_styles(self):
        s = ttk.Style()
        s.theme_use("clam")

        s.configure("BG.TFrame",     background=C["bg"])
        s.configure("Card.TFrame",   background=C["surface"])

        s.configure("T.TLabel",   background=C["bg"],      foreground=C["text"],
                    font=("Segoe UI", 10))
        s.configure("H.TLabel",   background=C["bg"],      foreground=C["text"],
                    font=("Segoe UI", 18, "bold"))
        s.configure("Sub.TLabel", background=C["bg"],      foreground=C["text2"],
                    font=("Segoe UI", 9))
        s.configure("St.TLabel",  background=C["surface"], foreground=C["text2"],
                    font=("Segoe UI", 9), wraplength=340)
        s.configure("Eng.TLabel", background=C["surface"], foreground=C["primary"],
                    font=("Segoe UI", 8, "bold"))

        s.configure("Act.TButton", background=C["primary"], foreground="white",
                    font=("Segoe UI", 12, "bold"), padding=(20, 12), borderwidth=0)
        s.map("Act.TButton",
              background=[("active", C["primary_h"]), ("disabled", C["border"])])

        s.configure("Sm.TButton", background=C["surface2"], foreground=C["text"],
                    font=("Segoe UI", 9), padding=(8, 4))
        s.map("Sm.TButton", background=[("active", C["primary"])])

        s.configure("D.TEntry", fieldbackground=C["entry"], foreground=C["text"],
                    insertcolor=C["text"], borderwidth=1)

        s.configure("D.TCombobox", fieldbackground=C["entry"], foreground=C["text"],
                    background=C["surface2"], borderwidth=1)
        s.map("D.TCombobox", fieldbackground=[("readonly", C["entry"])])

        s.configure("D.TCheckbutton", background=C["bg"], foreground=C["text2"],
                    font=("Segoe UI", 9))
        s.map("D.TCheckbutton", background=[("active", C["bg"])])

    # -------------------------------------------------------- ui
    def _build_ui(self):
        m = ttk.Frame(self.root, style="BG.TFrame", padding=22)
        m.grid(sticky="nsew")

        # header
        ttk.Label(m, text="📡  WiFi Hotspot Pro", style="H.TLabel"
                  ).grid(row=0, column=0, columnspan=3, sticky="w", pady=(0, 2))

        self.lbl_engine = ttk.Label(m, text="⏳  Đang kiểm tra hệ thống…",
                                    style="Sub.TLabel")
        self.lbl_engine.grid(row=1, column=0, columnspan=3, sticky="w", pady=(0, 16))

        # separator
        sep = tk.Frame(m, height=1, bg=C["border"])
        sep.grid(row=2, column=0, columnspan=3, sticky="ew", pady=(0, 14))

        # ssid
        ttk.Label(m, text="Tên WiFi", style="T.TLabel"
                  ).grid(row=3, column=0, sticky="w", pady=4)
        self.v_ssid = tk.StringVar(value=self.cfg["ssid"])
        ttk.Entry(m, textvariable=self.v_ssid, width=30, style="D.TEntry",
                  font=("Segoe UI", 10)).grid(row=3, column=1, columnspan=2,
                                              sticky="ew", pady=4)

        # password
        ttk.Label(m, text="Mật khẩu", style="T.TLabel"
                  ).grid(row=4, column=0, sticky="w", pady=4)
        self.v_pw = tk.StringVar(value=self.cfg["password"])
        self.ent_pw = ttk.Entry(m, textvariable=self.v_pw, width=30,
                                style="D.TEntry", font=("Segoe UI", 10), show="●")
        self.ent_pw.grid(row=4, column=1, columnspan=2, sticky="ew", pady=4)

        # confirm password
        ttk.Label(m, text="Nhập lại MK", style="T.TLabel"
                  ).grid(row=5, column=0, sticky="w", pady=4)
        self.v_pw2 = tk.StringVar(value=self.cfg["password"])
        self.ent_pw2 = ttk.Entry(m, textvariable=self.v_pw2, width=30,
                                 style="D.TEntry", font=("Segoe UI", 10), show="●")
        self.ent_pw2.grid(row=5, column=1, columnspan=2, sticky="ew", pady=4)

        self.v_showpw = tk.BooleanVar()
        ttk.Checkbutton(m, text="Hiện mật khẩu", variable=self.v_showpw,
                        style="D.TCheckbutton",
                        command=self._toggle_pw_visibility
                        ).grid(row=6, column=1, sticky="w")

        # source (chỉ hiện cho Hosted Network)
        self.fr_src = ttk.Frame(m, style="BG.TFrame")
        self.fr_src.grid(row=7, column=0, columnspan=3, sticky="ew", pady=(8, 0))

        ttk.Label(self.fr_src, text="Mạng Internet", style="T.TLabel"
                  ).grid(row=0, column=0, sticky="w")
        self.v_src = tk.StringVar()
        self.cmb_src = ttk.Combobox(self.fr_src, textvariable=self.v_src,
                                    width=26, state="readonly", style="D.TCombobox")
        self.cmb_src.grid(row=0, column=1, sticky="ew", padx=(10, 4))
        ttk.Button(self.fr_src, text="↻", width=3, style="Sm.TButton",
                   command=self._refresh_sources).grid(row=0, column=2)
        self.fr_src.columnconfigure(1, weight=1)

        # options
        of = ttk.Frame(m, style="BG.TFrame")
        of.grid(row=8, column=0, columnspan=3, sticky="ew", pady=(12, 0))
        self.v_autostart = tk.BooleanVar(value=self.cfg["win_autostart"])
        ttk.Checkbutton(of, text="Tự chạy cùng Windows",
                        variable=self.v_autostart, style="D.TCheckbutton"
                        ).grid(row=0, column=0, sticky="w")
        self.v_autohot = tk.BooleanVar(value=self.cfg["auto_hotspot"])
        ttk.Checkbutton(of, text="Tự động phát khi mở ứng dụng",
                        variable=self.v_autohot, style="D.TCheckbutton"
                        ).grid(row=1, column=0, sticky="w")

        # action button
        self.btn = ttk.Button(m, text="▶   Bắt đầu phát", style="Act.TButton",
                              command=self._toggle)
        self.btn.grid(row=9, column=0, columnspan=3, sticky="ew", pady=(18, 10))

        # status card
        sf = ttk.Frame(m, style="Card.TFrame", padding=12)
        sf.grid(row=10, column=0, columnspan=3, sticky="ew")

        self.cv_dot = tk.Canvas(sf, width=12, height=12, bg=C["surface"],
                                highlightthickness=0)
        self.cv_dot.grid(row=0, column=0, padx=(0, 8))
        self._dot("gray")

        self.lbl_status = ttk.Label(sf, text="Chưa phát", style="St.TLabel")
        self.lbl_status.grid(row=0, column=1, sticky="w")

        self.lbl_clients = ttk.Label(sf, text="", style="Eng.TLabel")
        self.lbl_clients.grid(row=0, column=2, sticky="e", padx=(10, 0))
        sf.columnconfigure(1, weight=1)

        # log viewer (collapsed)
        lf = ttk.Frame(m, style="BG.TFrame")
        lf.grid(row=11, column=0, columnspan=3, sticky="ew", pady=(12, 0))
        self.v_showlog = tk.BooleanVar()
        ttk.Checkbutton(lf, text="📋  Nhật ký hoạt động", variable=self.v_showlog,
                        style="D.TCheckbutton",
                        command=self._toggle_log).grid(row=0, column=0, sticky="w")

        self.txt_log = tk.Text(lf, height=7, width=48, bg=C["entry"],
                               fg=C["text2"], font=("Consolas", 8),
                               insertbackground=C["text"],
                               selectbackground=C["primary"],
                               relief="flat", bd=0, state="disabled")
        self.txt_log.grid(row=1, column=0, sticky="ew", pady=(4, 0))
        self.txt_log.grid_remove()
        lf.columnconfigure(0, weight=1)

        # footer — log file path
        ttk.Label(m, text=f"Log: {LOG_FILE}", style="Sub.TLabel"
                  ).grid(row=12, column=0, columnspan=3, sticky="w", pady=(10, 0))

        m.columnconfigure(1, weight=1)

    # -------------------------------------------------------- helpers
    def _dot(self, color):
        self.cv_dot.delete("all")
        self.cv_dot.create_oval(1, 1, 11, 11, fill=color, outline="")

    def _toggle_pw_visibility(self):
        show = "" if self.v_showpw.get() else "●"
        self.ent_pw.config(show=show)
        self.ent_pw2.config(show=show)

    def _toggle_log(self):
        if self.v_showlog.get():
            self.txt_log.grid()
        else:
            self.txt_log.grid_remove()

    def _set_status(self, txt, color=C["text2"]):
        self.lbl_status.config(text=txt, foreground=color)
        cmap = {C["success"]: C["success"], C["error"]: C["error"],
                C["warning"]: C["warning"]}
        self._dot(cmap.get(color, "gray"))

    def _update_state(self, running, txt):
        self.running = running
        self.btn.config(
            text="⏹   Dừng phát" if self.want_run else "▶   Bắt đầu phát",
            state="normal")
        self._set_status(txt, C["success"] if running else C["text2"])
        if self.tray:
            self.tray.icon = _make_icon(running)
        if not running:
            self.lbl_clients.config(text="")

    def _append_log(self, msg):
        self.txt_log.config(state="normal")
        self.txt_log.insert("end", msg + "\n")
        self.txt_log.see("end")
        lc = int(self.txt_log.index("end-1c").split(".")[0])
        if lc > 300:
            self.txt_log.delete("1.0", f"{lc - 300}.0")
        self.txt_log.config(state="disabled")

    # -------------------------------------------------------- sources
    def _refresh_sources(self):
        try:
            self.sources = list_sources()
        except Exception as e:
            self.sources = []
            self._set_status(f"Lỗi: {e}", C["error"])
        self.cmb_src["values"] = [f"{n}  ({d})" for n, d in self.sources]
        names = [n for n, _ in self.sources]
        if self.cfg["source"] in names:
            self.cmb_src.current(names.index(self.cfg["source"]))
        elif names:
            self.cmb_src.current(0)

    def _sel_source(self):
        i = self.cmb_src.current()
        return self.sources[i][0] if 0 <= i < len(self.sources) else ""

    # -------------------------------------------------------- config
    def _save(self):
        self.cfg.update(
            ssid=self.v_ssid.get().strip(), password=self.v_pw.get(),
            source=self._sel_source(), engine=self.mgr.engine,
            win_autostart=self.v_autostart.get(),
            auto_hotspot=self.v_autohot.get())
        save_cfg(self.cfg)
        ok = set_autostart(self.cfg["win_autostart"])
        if self.cfg["win_autostart"] and not ok:
            self._set_status("Không tạo được tác vụ tự chạy Windows", C["error"])

    def _validate(self):
        ssid = self.v_ssid.get().strip()
        pw = self.v_pw.get()
        pw2 = self.v_pw2.get()
        if not ssid:
            messagebox.showwarning("Thiếu", "Nhập tên WiFi"); return False
        if len(ssid) > 32:
            messagebox.showwarning("Tên WiFi", "Tối đa 32 ký tự"); return False
        if not pw:
            messagebox.showwarning("Thiếu", "Nhập mật khẩu"); return False
        if not 8 <= len(pw) <= 63:
            messagebox.showwarning("Mật khẩu", "Phải từ 8-63 ký tự"); return False
        if pw != pw2:
            messagebox.showwarning("Mật khẩu", "Mật khẩu nhập lại không khớp"); return False
        if self.mgr.engine == ENGINE_HOSTED and not self._sel_source():
            messagebox.showwarning("Thiếu", "Chọn mạng Internet (Hosted Network cần)")
            return False
        return True

    # -------------------------------------------------------- control
    def _toggle(self):
        if self.want_run:
            self.want_run = False
            self._set_status("Đang dừng…", C["warning"])
            self.btn.config(state="disabled")
            threading.Thread(target=self._do_stop, daemon=True).start()
        else:
            if self.mgr.engine == ENGINE_NONE:
                messagebox.showerror("Lỗi", "Không có phương thức phát WiFi khả dụng.")
                return
            if not self._validate():
                return
            self._save()
            self.want_run = True
            self._set_status("Đang khởi động…", C["warning"])
            self.btn.config(text="⏹   Dừng phát")

    def _do_stop(self):
        ok, msg = self.mgr.stop()
        self.q.put(("state", False, msg))

    # -------------------------------------------------------- detect engine
    def _detect_thread(self):
        eng, msg = self.mgr.detect()
        self.q.put(("engine", eng, msg))

    def _on_engine_detected(self, eng, msg):
        self.lbl_engine.config(text=msg)
        if eng == ENGINE_MOBILE:
            self.fr_src.grid_remove()
        elif eng == ENGINE_HOSTED:
            self._refresh_sources()
        if eng == ENGINE_NONE:
            self.btn.config(state="disabled")
            self._set_status("Không thể phát WiFi", C["error"])
        else:
            # Khởi động watchdog
            threading.Thread(target=self._watchdog, daemon=True).start()
            # Auto-start nếu được cấu hình
            if self.cfg["auto_hotspot"] and self.cfg["ssid"]:
                if eng == ENGINE_HOSTED and not self.cfg["source"]:
                    log.info("Auto-start bỏ qua: chưa chọn nguồn Internet")
                else:
                    self.want_run = True
                    self.btn.config(text="⏹   Dừng phát")

    # -------------------------------------------------------- watchdog
    def _watchdog(self):
        log.info("Watchdog bắt đầu")
        while not self.stop_evt.is_set():
            try:
                if self.want_run and not self.busy:
                    if not self.mgr.is_running():
                        self.busy = True
                        self.q.put(("status", "Đang bật WiFi…", C["warning"]))
                        c = self.cfg
                        ok, msg = self.mgr.start(
                            c["ssid"], c["password"], c.get("source", ""))
                        self.busy = False
                        if ok:
                            self.q.put(("state", True, msg))
                        else:
                            self.q.put(("state", False, "Lỗi: " + msg))
                    elif not self.running:
                        self.q.put(("state", True, "Đang phát"))
                    else:
                        cnt = self.mgr.client_count()
                        self.q.put(("clients", cnt))
            except Exception as e:
                self.busy = False
                log.error("Watchdog lỗi: %s", e, exc_info=True)
                self.q.put(("status", f"Lỗi: {e}", C["error"]))

            self.stop_evt.wait(10 if self.running else 5)
        log.info("Watchdog dừng")

    # -------------------------------------------------------- poll queue
    def _poll(self):
        try:
            while True:
                item = self.q.get_nowait()
                if isinstance(item, tuple):
                    kind = item[0]
                    if kind == "state":
                        self._update_state(item[1], item[2])
                    elif kind == "status":
                        self._set_status(item[1], item[2])
                    elif kind == "engine":
                        self._on_engine_detected(item[1], item[2])
                    elif kind == "clients":
                        cnt = item[1]
                        self.lbl_clients.config(
                            text=f"👥 {cnt}" if cnt > 0 else "")
                    elif kind == "_log":
                        self._append_log(item[1])
                elif callable(item):
                    item()
        except queue.Empty:
            pass
        self.root.after(150, self._poll)

    # -------------------------------------------------------- tray
    def _hide(self):
        self.root.withdraw()

    def _tray_show(self, *a):
        self.q.put(lambda: (self.root.deiconify(), self.root.lift()))

    def _tray_toggle(self, *a):
        self.q.put(self._toggle)

    def _tray_quit(self, *a):
        self.q.put(self._quit)

    def _quit(self):
        log.info("Đang thoát…")
        self.want_run = False
        self.stop_evt.set()
        try:
            self.mgr.stop()
        except Exception as e:
            log.error("Lỗi dừng khi thoát: %s", e)
        if self.tray:
            try:
                self.tray.stop()
            except Exception:
                pass
        try:
            self.root.destroy()
        except Exception:
            pass
        log.info("Đã thoát")
        os._exit(0)


# ================================================================ MAIN

def main():
    if not is_admin():
        relaunch_as_admin()
        return
    if not single_instance():
        return
    app = App(minimized="--minimized" in sys.argv)
    app.root.mainloop()


if __name__ == "__main__":
    main()
# wifi_hostpot
