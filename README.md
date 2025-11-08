#!/usr/bin/env python3
"""
xpecs.py - 

Features (best-effort):
 - Two tabs: System and About
 - Dark neon-green UI
 - Live CPU usage
 - CPU name, cores, frequency, temperature (if available)
 - Memory (total/used)
 - HDD model, size, basic health via smartctl (if installed)
 - Fan speeds (psutil.sensors_fans if available)
 - Motherboard serial (Windows wmi or /sys/class/dmi/id on Linux)
 - Screen resolution
 - Four rating bars (CPU, RAM, Disk, GPU)
 - "Rate my PC" button with five possible messages determined by RAM thresholds & CPU factor
"""
import tkinter as tk
from tkinter import ttk, messagebox
import platform
import threading
import subprocess
import sys
import os
import time
from datetime import datetime

# Optional modules
try:
    import psutil
except Exception:
    psutil = None

# On Windows, try wmi for motherboard serial
is_windows = sys.platform.startswith("win")
has_wmi = False
if is_windows:
    try:
        import wmi
        has_wmi = True
    except Exception:
        has_wmi = False

# -------------------------
# Helper functions (best-effort)
# -------------------------
def safe_run(cmd_list, text=True):
    try:
        out = subprocess.check_output(cmd_list, stderr=subprocess.DEVNULL, universal_newlines=True)
        return out.strip() if text else out
    except Exception:
        return ""

def human_bytes(n):
    try:
        n = float(n)
    except Exception:
        return "N/A"
    units = ["B","KB","MB","GB","TB"]
    i = 0
    while n >= 1024 and i < len(units)-1:
        n /= 1024.0
        i += 1
    if i <= 1:
        return f"{int(n)} {units[i]}"
    else:
        return f"{n:.1f} {units[i]}"

def get_cpu_info():
    info = {}
    info['name'] = platform.processor() or platform.uname().processor or "Unknown CPU"
    try:
        info['logical'] = psutil.cpu_count(logical=True) if psutil else os.cpu_count()
        info['physical'] = psutil.cpu_count(logical=False) if psutil else None
    except Exception:
        info['logical'] = os.cpu_count()
        info['physical'] = None
    # freq
    try:
        if psutil and hasattr(psutil, "cpu_freq"):
            f = psutil.cpu_freq()
            info['freq'] = f.current if f else None
        else:
            info['freq'] = None
    except Exception:
        info['freq'] = None
    # temps
    try:
        if psutil and hasattr(psutil, "sensors_temperatures"):
            temps = psutil.sensors_temperatures()
            # heuristic: look for 'coretemp', 'cpu-thermal', 'acpitz'
            for k in ('coretemp','cpu-thermal','acpitz'):
                if k in temps:
                    info['temp'] = temps[k][0].current
                    break
            else:
                # any sensor
                if temps:
                    first = next(iter(temps.values()))
                    if first:
                        info['temp'] = first[0].current
                    else:
                        info['temp'] = None
                else:
                    info['temp'] = None
        else:
            info['temp'] = None
    except Exception:
        info['temp'] = None
    return info

def get_fans():
    fans = {}
    try:
        if psutil and hasattr(psutil, "sensors_fans"):
            sf = psutil.sensors_fans()
            for k, arr in sf.items():
                fans[k] = [(e.label or "fan", getattr(e, 'current', None)) for e in arr]
    except Exception:
        pass
    return fans

def get_memory_info():
    m = {}
    try:
        if psutil:
            v = psutil.virtual_memory()
            m['total'] = v.total
            m['used'] = v.total - v.available
            m['percent'] = v.percent
        else:
            m['total'] = None
            m['used'] = None
            m['percent'] = None
    except Exception:
        m['total'] = m['used'] = m['percent'] = None
    return m

def get_disks_info():
    # Return list of disks with model/size/health if detectable
    disks = []
    # try platform-specific
    if is_windows:
        # wmic (best-effort)
        out = safe_run(["wmic", "diskdrive", "get", "Model,Size,SerialNumber"], text=True)
        if out:
            lines = [l for l in out.splitlines() if l.strip()]
            if len(lines) >= 2:
                keys_line = lines[0]
                entries = lines[1:]
                # rough parse by columns - simpler: split whitespace-smart
                for ent in entries:
                    parts = ent.split()
                    if not parts:
                        continue
                    # fallback: model may be everything except last token(s) containing size/serial - this is brittle
                    # We'll use WMI per-line query for each index if available
                    # Better: call wmic diskdrive get Model /format:list then parse blocks
                # Try list form
                out2 = safe_run(["wmic", "diskdrive", "get", "/format:list"])
                if out2:
                    blocks = [b for b in out2.split("\n\n") if b.strip()]
                    for b in blocks:
                        d = {}
                        for line in b.splitlines():
                            if '=' in line:
                                k,v = line.split('=',1)
                                d[k.strip()] = v.strip()
                        if d:
                            disks.append({
                                'model': d.get('Model',''),
                                'size': int(d.get('Size')) if d.get('Size') and d.get('Size').isdigit() else None,
                                'serial': d.get('SerialNumber','')
                            })
    else:
        # Linux / macOS: try lsblk
        out = safe_run(["lsblk", "-d", "-o", "NAME,MODEL,SIZE,TYPE"], text=True)
        if out:
            for line in out.splitlines()[1:]:
                if not line.strip():
                    continue
                parts = line.split()
                if len(parts) >= 3:
                    model = " ".join(parts[1:-1])
                    size = parts[-1]
                    disks.append({'model': model, 'size_str': size, 'serial': None})
    # If nothing discovered via above, fallback to psutil partitions
    if not disks:
        try:
            if psutil:
                seen = set()
                for part in psutil.disk_partitions(all=False):
                    device = part.device
                    if device in seen:
                        continue
                    seen.add(device)
                    try:
                        usage = psutil.disk_usage(part.mountpoint)
                        disks.append({'model': device, 'size': usage.total, 'serial': None})
                    except Exception:
                        disks.append({'model': device, 'size': None, 'serial': None})
        except Exception:
            pass

    # Try SMART health via smartctl for the first disk (requires smartmontools installed)
    for d in disks:
        d['health'] = None
        d['smart_ok'] = None
    try:
        # find a common device to run smartctl on
        dev = None
        if is_windows:
            # windows smartctl usage is more complex; skip
            dev = None
        else:
            # common candidates
            for candidate in ("/dev/sda","/dev/nvme0n1","/dev/vda"):
                if os.path.exists(candidate):
                    dev = candidate
                    break
            if not dev and disks:
                # try to infer from disk model field name
                # skip if not found
                pass
        if dev:
            out = safe_run(["smartctl", "-H", dev])
            if out:
                # parse "PASSED" or "FAILED"
                if "PASSED" in out:
                    smart_ok = True
                elif "FAILED" in out:
                    smart_ok = False
                else:
                    smart_ok = None
                if disks:
                    disks[0]['smart_ok'] = smart_ok
                    disks[0]['health'] = "PASSED" if smart_ok else ("FAILED" if smart_ok is False else "Unknown")
    except Exception:
        pass

    return disks

def get_motherboard_serial():
    try:
        if is_windows and has_wmi:
            c = wmi.WMI()
            for board in c.Win32_BaseBoard():
                return getattr(board, "SerialNumber", "") or None
        else:
            # try linux sysfs
            p = "/sys/class/dmi/id/board_serial"
            if os.path.exists(p):
                try:
                    with open(p, "r", errors="ignore") as f:
                        s = f.read().strip()
                        return s or None
                except Exception:
                    pass
            # fallback: dmidecode (may require root)
            out = safe_run(["dmidecode", "-s", "baseboard-serial-number"])
            if out:
                return out.strip()
    except Exception:
        pass
    return None

def get_screen_resolution(root):
    try:
        w = root.winfo_screenwidth()
        h = root.winfo_screenheight()
        return f"{w} x {h}"
    except Exception:
        return "N/A"

def get_gpu_info():
    # Best-effort: try to find GPU via lspci (linux) or wmic (win)
    gpu_name = None
    try:
        if is_windows:
            out = safe_run(["wmic", "path", "win32_VideoController", "get", "name"], text=True)
            if out:
                lines = [l.strip() for l in out.splitlines() if l.strip()]
                if len(lines) >= 2:
                    gpu_name = lines[1]
        else:
            out = safe_run(["lspci"], text=True)
            if out:
                for line in out.splitlines():
                    if "VGA compatible controller" in line or "3D controller" in line:
                        gpu_name = " ".join(line.split()[2:])
                        break
    except Exception:
        gpu_name = None
    return gpu_name

# -------------------------
# Rating logic
# -------------------------
RATE_MESSAGES = {
    1: "your pc is shitty as hell its a ancient bro why are you still using it",
    2: "ewaste xd",
    3: "well, decent but do not expect it to run cs2 or any heavy game like gta5",
    4: "cool pc you can game on it lmao",
    5: "bro did u fucking stole that shit from nasa"
}

def compute_rating_message(ram_bytes, cpu_info, cpu_usage_percent):
    # RAM thresholds per user's specification (interpreted):
    # < 2 GB -> message 1
    # >=2 and <3 -> message 2
    # >=3 and <8 -> message 3
    # >=8 and <16 -> message 4
    # >=16 -> message 5
    gb = (ram_bytes or 0) / (1024**3) if ram_bytes else 0
    if gb < 2:
        base = 1
    elif gb < 3:
        base = 2
    elif gb < 8:
        base = 3
    elif gb < 16:
        base = 4
    else:
        base = 5

    # CPU factor: try to nudge rating up or down
    adjust = 0
    # if CPU has many physical cores or is modern-sounding, boost
    name = (cpu_info.get('name') or "").lower()
    cores = cpu_info.get('physical') or cpu_info.get('logical') or 0
    if cores >= 8:
        adjust += 1
    if any(tok in name for tok in ("ryzen","xeon","epyc","intel(r) core(tm) i7","intel(r) core(tm) i9","core i7","core i9", "ryzen 7","ryzen 9")):
        adjust += 1
    # if CPU usage is very high, degrade
    if cpu_usage_percent is not None:
        try:
            if cpu_usage_percent > 75:
                adjust -= 1
            elif cpu_usage_percent > 50:
                adjust -= 0  # neutral
        except Exception:
            pass

    final = max(1, min(5, base + adjust))
    return RATE_MESSAGES[final], final

# -------------------------
# GUI
# -------------------------
class XPecsApp(tk.Tk):
    def __init__(self):
        super().__init__()
        self.title("xpecs — system inspector")
        self.geometry("820x560")
        self.minsize(700,480)
        # dark neon theme colors
        self.bg = "#0b0f0b"
        self.panel = "#07100a"
        self.neon = "#39ff14"  # neon green
        self.faint = "#2b2b2b"
        self.text_fg = "#dfffdc"
        self.configure(bg=self.bg)

        # style
        style = ttk.Style(self)
        style.theme_use('default')
        style.configure("TNotebook", background=self.bg, borderwidth=0)
        style.configure("TFrame", background=self.bg)
        style.configure("TLabel", background=self.bg, foreground=self.neon, font=("Consolas", 11))
        style.configure("TButton", background=self.faint, foreground=self.neon)
        style.configure("Green.Horizontal.TProgressbar", troughcolor=self.faint, background=self.neon, bordercolor=self.bg, thickness=12)

        # Notebook
        nb = ttk.Notebook(self)
        nb.pack(fill="both", expand=True, padx=12, pady=12)

        # System tab
        sys_tab = ttk.Frame(nb)
        nb.add(sys_tab, text="System")

        # Layout - left column info, right column detailed text
        left = tk.Frame(sys_tab, bg=self.panel)
        left.pack(side="left", fill="y", padx=(10,6), pady=10)

        right = tk.Frame(sys_tab, bg=self.panel)
        right.pack(side="right", expand=True, fill="both", padx=(6,10), pady=10)

        # Left info labels (neon)
        self.labels = {}
        keys = [
            ("CPU", "cpu_name"),
            ("Cores (phy/log)", "cpu_cores"),
            ("CPU Frequency", "cpu_freq"),
            ("CPU Temp", "cpu_temp"),
            ("CPU Usage", "cpu_usage"),
            ("Memory", "memory"),
            ("Motherboard SN", "mobo_sn"),
            ("HDD Model", "hdd_model"),
            ("HDD Size", "hdd_size"),
            ("HDD Health", "hdd_health"),
            ("Fan (example)", "fan_info"),
            ("Screen", "screen_res"),
            ("GPU", "gpu")
        ]
        for (lbl, key) in keys:
            f = tk.Frame(left, bg=self.panel)
            f.pack(fill="x", padx=8, pady=6)
            title = tk.Label(f, text=lbl + ":", font=("Consolas",10,"bold"), fg=self.neon, bg=self.panel, anchor="w", justify="left")
            title.pack(anchor="w")
            val = tk.Label(f, text="Detecting...", font=("Consolas",10), fg=self.text_fg, bg=self.panel, anchor="w", justify="left", wraplength=260)
            val.pack(anchor="w")
            self.labels[key] = val

        # Right: a big console-like text area for detailed snapshot and refresh button
        self.detail_text = tk.Text(right, bg="#071409", fg=self.neon, insertbackground=self.neon, font=("Consolas",10), wrap="none")
        self.detail_text.pack(expand=True, fill="both", padx=6, pady=6)
        btn_frame = tk.Frame(right, bg=self.panel)
        btn_frame.pack(fill="x", padx=6, pady=(0,6))
        refresh_btn = tk.Button(btn_frame, text="Refresh", command=self.refresh_snapshot, bg=self.faint, fg=self.neon)
        refresh_btn.pack(side="left", padx=6)
        export_btn = tk.Button(btn_frame, text="Export Snapshot", command=self.export_snapshot, bg=self.faint, fg=self.neon)
        export_btn.pack(side="left", padx=6)

        # Bottom area: four rate bars and Rate my PC button
        bottom = tk.Frame(self, bg=self.bg)
        bottom.pack(side="bottom", fill="x", padx=12, pady=(0,12))

        bars_frame = tk.Frame(bottom, bg=self.bg)
        bars_frame.pack(side="left", padx=12)
        self.ratings = {}
        for name in ("CPU", "RAM", "Disk", "GPU"):
            lab = tk.Label(bars_frame, text=name, bg=self.bg, fg=self.neon, font=("Consolas",10))
            lab.pack(anchor="w")
            pb = ttk.Progressbar(bars_frame, style="Green.Horizontal.TProgressbar", length=220, maximum=100)
            pb.pack(pady=6)
            self.ratings[name.lower()] = pb

        # Rating label
        self.rate_label = tk.Label(bottom, text="", bg=self.bg, fg=self.neon, font=("Consolas",11,"bold"), wraplength=420, justify="left")
        self.rate_label.pack(side="left", padx=12)

        # Rate my PC button
        rate_btn = tk.Button(bottom, text="Rate my PC", command=self.on_rate_clicked, bg="#0f2a0f", fg=self.neon, font=("Consolas",11,"bold"))
        rate_btn.pack(side="right", padx=18)

        # About tab
        about_tab = ttk.Frame(nb)
        nb.add(about_tab, text="About")
        about_msg = (
            "Created by RazerBlockers.\n"
            "This software is absolutely fucking free.\n"
            "But you know I created that and I would be happy if you support me.\n\n"
            "https://github.com/razerlockers\n"
            "https://x.com/RazerLockers"
        )
        about_label = tk.Label(about_tab, text=about_msg, bg=self.bg, fg=self.neon, font=("Consolas",12), justify="center")
        about_label.pack(expand=True, fill="both", padx=20, pady=40)

        # internal storage
        self.snapshot = {}
        self.root_ref = self

        # start background updater for live CPU usage
        self._stop_threads = False
        self._start_background_updates()
        # initial refresh
        self.refresh_snapshot()

    def _start_background_updates(self):
        def loop():
            while not self._stop_threads:
                try:
                    cpu_pct = None
                    if psutil:
                        cpu_pct = psutil.cpu_percent(interval=1)
                    else:
                        time.sleep(1)
                    # schedule UI update
                    self.after(1, lambda p=cpu_pct: self._update_cpu_usage_label(p))
                except Exception:
                    time.sleep(1)
        t = threading.Thread(target=loop, daemon=True)
        t.start()

    def _update_cpu_usage_label(self, pct):
        if pct is None:
            txt = "N/A"
        else:
            txt = f"{pct:.1f}%"
        lbl = self.labels.get("cpu_usage")
        if lbl:
            lbl.config(text=txt)
        # also update CPU rating bar influence
        self.snapshot.setdefault('cpu_usage', pct)
        # recompute bars (not re-running full snapshot)
        self._update_rating_bars_from_snapshot()

    def refresh_snapshot(self):
        # collect snapshot in separate thread
        def worker():
            snap = {}
            snap['timestamp'] = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S UTC")
            snap['cpu'] = get_cpu_info()
            snap['memory'] = get_memory_info()
            snap['disks'] = get_disks_info()
            snap['fans'] = get_fans()
            snap['mobo_sn'] = get_motherboard_serial()
            snap['screen'] = get_screen_resolution(self.root_ref)
            snap['gpu'] = get_gpu_info()
            snap['cpu_usage'] = psutil.cpu_percent(interval=0.5) if psutil else None
            # store
            self.snapshot = snap
            self.after(1, lambda: self._apply_snapshot(snap))
        threading.Thread(target=worker, daemon=True).start()

    def _apply_snapshot(self, snap):
        c = snap.get('cpu',{})
        mem = snap.get('memory',{})
        disks = snap.get('disks',[])
        fans = snap.get('fans',{})
        # labels
        self.labels['cpu_name'].config(text=c.get('name','Unknown'))
        cpu_cores = f"{c.get('physical') or 'N/A'} / {c.get('logical') or 'N/A'}"
        self.labels['cpu_cores'].config(text=cpu_cores)
        self.labels['cpu_freq'].config(text=f"{c.get('freq'):.2f} MHz" if c.get('freq') else "N/A")
        self.labels['cpu_temp'].config(text=f"{c.get('temp')}°C" if c.get('temp') else "N/A")
        self.labels['cpu_usage'].config(text=f"{snap.get('cpu_usage'):.1f}%" if snap.get('cpu_usage') is not None else "N/A")
        if mem.get('total'):
            self.labels['memory'].config(text=f"{human_bytes(mem['used'])} used / {human_bytes(mem['total'])} ({mem.get('percent')}%)")
        else:
            self.labels['memory'].config(text="N/A")
        self.labels['mobo_sn'].config(text=snap.get('mobo_sn') or "N/A")
        # HDD
        if disks:
            first = disks[0]
            model = first.get('model') or first.get('model','Unknown')
            size = first.get('size') or first.get('size_str') or None
            health = first.get('health') or ("PASSED" if first.get('smart_ok') else ("FAILED" if first.get('smart_ok') is False else "Unknown"))
            self.labels['hdd_model'].config(text=str(model))
            self.labels['hdd_size'].config(text=human_bytes(size) if isinstance(size, (int,float)) else str(size))
            self.labels['hdd_health'].config(text=str(health))
        else:
            self.labels['hdd_model'].config(text="N/A")
            self.labels['hdd_size'].config(text="N/A")
            self.labels['hdd_health'].config(text="N/A")
        # fans
        fans_txt = []
        for k, arr in fans.items():
            for lbl, spd in arr:
                fans_txt.append(f"{lbl}: {spd} RPM" if spd else f"{lbl}: N/A")
        self.labels['fan_info'].config(text="\n".join(fans_txt) if fans_txt else "N/A")
        # screen
        self.labels['screen_res'].config(text=snap.get('screen') or "N/A")
        self.labels['gpu'].config(text=snap.get('gpu') or "N/A")

        # fill detailed snapshot text
        txt = []
        txt.append(f"Snapshot: {snap.get('timestamp')}")
        txt.append("=== CPU ===")
        txt.append(f"Name: {c.get('name')}")
        txt.append(f"Cores (physical/logical): {cpu_cores}")
        txt.append(f"Freq: {c.get('freq') or 'N/A'} MHz")
        txt.append(f"Temp: {c.get('temp') or 'N/A'} °C")
        txt.append("")
        txt.append("=== Memory ===")
        txt.append(f"Total: {human_bytes(mem.get('total')) if mem.get('total') else 'N/A'}")
        txt.append(f"Used: {human_bytes(mem.get('used')) if mem.get('used') else 'N/A'} ({mem.get('percent') or 'N/A'}%)")
        txt.append("")
        txt.append("=== Disks ===")
        for d in disks:
            txt.append(f"Model: {d.get('model')}  Size: {d.get('size') or d.get('size_str')}")
            txt.append(f"  Health: {d.get('health') or d.get('smart_ok') or 'N/A'}")
        txt.append("")
        txt.append("=== Fans ===")
        if fans:
            for k, arr in fans.items():
                for lbl, spd in arr:
                    txt.append(f"{lbl}: {spd} RPM" if spd else f"{lbl}: N/A")
        else:
            txt.append("N/A")
        txt.append("")
        txt.append("=== Motherboard ===")
        txt.append(f"Serial: {snap.get('mobo_sn') or 'N/A'}")
        txt.append("")
        txt.append("=== Screen ===")
        txt.append(f"Resolution: {snap.get('screen')}")
        txt.append("")
        txt.append("=== GPU ===")
        txt.append(f"{snap.get('gpu') or 'N/A'}")
        txt.append("")
        self.detail_text.delete("1.0", "end")
        self.detail_text.insert("1.0", "\n".join(txt))

        # update rating bars and label
        self._update_rating_bars_from_snapshot()

    def _update_rating_bars_from_snapshot(self):
        snap = self.snapshot or {}
        # CPU score: based on cores and frequency and temp
        cpu = snap.get('cpu',{})
        cpu_score = 20
        try:
            cores = cpu.get('physical') or cpu.get('logical') or 0
            if cores >= 8:
                cpu_score += 40
            elif cores >= 4:
                cpu_score += 25
            elif cores >= 2:
                cpu_score += 10
            freq = cpu.get('freq') or 0
            if freq and freq >= 3000:
                cpu_score += 20
            elif freq and freq >= 2000:
                cpu_score += 10
            temp = cpu.get('temp')
            if temp and temp > 85:
                cpu_score -= 10
        except Exception:
            pass
        cpu_score = max(0, min(100, cpu_score))
        self.ratings['cpu'].config(value=cpu_score)

        # RAM score: based on total size
        mem = snap.get('memory',{})
        ram_score = 0
        try:
            total = mem.get('total') or 0
            gb = (total or 0) / (1024**3)
            if gb >= 32:
                ram_score = 100
            elif gb >= 16:
                ram_score = 85
            elif gb >= 8:
                ram_score = 65
            elif gb >= 4:
                ram_score = 40
            elif gb >= 2:
                ram_score = 20
            else:
                ram_score = 10
        except Exception:
            ram_score = 0
        self.ratings['ram'].config(value=ram_score)

        # Disk score: based on presence of NVMe or SSD detection by model string heuristic
        disk_score = 30
        try:
            disks = snap.get('disks') or []
            if disks:
                first = disks[0]
                model = (first.get('model') or "").lower() if first.get('model') else ""
                size = first.get('size') or 0
                if "nvme" in model or "ssd" in model:
                    disk_score = 85
                elif size and isinstance(size, int) and size >= 1024**3 * 512:
                    disk_score = 70
                else:
                    disk_score = 45
                # degrade if smart FAILED
                if first.get('smart_ok') is False:
                    disk_score = min(30, disk_score)
        except Exception:
            pass
        self.ratings['disk'].config(value=disk_score)

        # GPU score: basic heuristic by name or absence
        gpu_score = 0
        try:
            g = snap.get('gpu')
            if not g:
                gpu_score = 10
            else:
                name = str(g).lower()
                if any(k in name for k in ("rtx","rx ","gtx","radeon","geforce","nvidia","amd")):
                    if "rtx" in name or "rx 6" in name:
                        gpu_score = 90
                    else:
                        gpu_score = 70
                else:
                    gpu_score = 40
        except Exception:
            gpu_score = 10
        self.ratings['gpu'].config(value=gpu_score)

        # Update textual rate summary 
        # We'll not auto-change the rate_label here; only update when button clicked,
        # but we can optionally show a compact summary
        summary = f"CPU: {int(cpu_score)}  RAM: {int(ram_score)}  Disk: {int(disk_score)}  GPU: {int(gpu_score)}"
        self.rate_label.config(text=summary)

    def on_rate_clicked(self):
        snap = self.snapshot or {}
        ram_total = snap.get('memory',{}).get('total') or 0
        cpu_info = snap.get('cpu',{})
        cpu_usage = snap.get('cpu_usage')
        msg, score = compute_rating_message(ram_total, cpu_info, cpu_usage)
        # show messagebox and update rate_label with the phrase and level
        self.rate_label.config(text=f"Rating: {score} — {msg}")
        messagebox.showinfo("xpecs — Rating", msg)

    def export_snapshot(self):
        try:
            import tkinter.filedialog as fd
            fn = fd.asksaveasfilename(defaultextension=".txt", filetypes=[("Text file","*.txt")])
            if not fn:
                return
            with open(fn, "w", encoding="utf-8") as f:
                f.write(self.detail_text.get("1.0", "end"))
            messagebox.showinfo("Saved", f"Snapshot saved to: {fn}")
        except Exception as e:
            messagebox.showerror("Error", f"Failed to export: {e}")

    def on_close(self):
        self._stop_threads = True
        self.destroy()

# -------------------------
# Entrypoint
# -------------------------
def main():
    if not psutil:
        print("Warning: psutil not found. For best results install psutil: pip install psutil")
    app = XPecsApp()
    app.protocol("WM_DELETE_WINDOW", app.on_close)
    app.mainloop()

if __name__ == "__main__":
    main()
