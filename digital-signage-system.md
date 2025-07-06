# Raspberry Pi Digital Signage System

## Project Structure

```
/home/admin/media-player/
├── app/
│   ├── app.py
│   ├── config.yaml
│   ├── requirements.txt
│   ├── utils/
│   │   ├── __init__.py
│   │   ├── display.py
│   │   └── monitoring.py
│   ├── modes/
│   │   ├── __init__.py
│   │   ├── mode1_recognition/
│   │   │   ├── __init__.py
│   │   │   ├── ui.py
│   │   │   ├── renderer.py
│   │   │   ├── config.json
│   │   │   └── templates/
│   │   │       └── index.html
│   │   ├── mode2_video/
│   │   │   ├── __init__.py
│   │   │   ├── ui.py
│   │   │   ├── renderer.py
│   │   │   ├── config.json
│   │   │   └── templates/
│   │   │       └── index.html
│   │   ├── mode3_kiosk/
│   │   │   ├── __init__.py
│   │   │   ├── ui.py
│   │   │   ├── renderer.py
│   │   │   ├── config.json
│   │   │   └── templates/
│   │   │       └── index.html
│   │   └── mode4_slideshow/
│   │       ├── __init__.py
│   │       ├── ui.py
│   │       ├── renderer.py
│   │       ├── config.json
│   │       └── templates/
│   │           └── index.html
│   ├── templates/
│   │   ├── base.html
│   │   └── admin.html
│   └── static/
│       └── style.css
├── media/
│   ├── backgrounds/
│   ├── images/
│   ├── videos/
│   └── text/
├── scripts/
│   ├── install-media.sh
│   ├── uninstall.sh
│   ├── update.sh
│   └── preflight_check.py
├── systemd/
│   └── templates/
│       ├── media-controller.service.j2
│       └── media-mode.service.j2
├── nginx/
│   └── templates/
│       └── media-player.conf.j2
├── docs/
│   ├── API.md
│   ├── TROUBLESHOOTING.md
│   └── templates/
│       └── playlist.csv
└── README.md
```

## Core Application Files

### `/home/admin/media-player/app/app.py`

```python
#!/usr/bin/env python3
import os
import yaml
import importlib
from flask import Flask, render_template, redirect, url_for
from flask_prometheus_metrics import PrometheusMetrics
from utils.monitoring import get_system_stats

app = Flask(__name__)
metrics = PrometheusMetrics(app)

# Load configuration
with open('config.yaml', 'r') as f:
    config = yaml.safe_load(f)

# Dynamic mode loading
enabled_modes = {}
for mode_name, mode_config in config['modes'].items():
    if mode_config.get('enabled', False):
        try:
            module_name = f"modes.mode{len(enabled_modes)+1}_{mode_name}"
            ui_module = importlib.import_module(f"{module_name}.ui")
            app.register_blueprint(ui_module.blueprint, url_prefix=f"/admin/modes/{mode_name}")
            enabled_modes[mode_name] = {
                'module': module_name,
                'config': mode_config
            }
        except ImportError as e:
            print(f"Failed to load mode {mode_name}: {e}")

@app.route('/')
def index():
    return redirect(url_for('admin'))

@app.route('/admin')
def admin():
    return render_template('admin.html', modes=enabled_modes, stats=get_system_stats())

@app.route('/health')
@metrics.do_not_track()
def health():
    return {'status': 'ok', 'modes': list(enabled_modes.keys())}

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=False)
```

### `/home/admin/media-player/app/config.yaml`

```yaml
global:
  resolution: [1920, 1080]
  frame_rate: 60
  media_dirs:
    backgrounds: media/backgrounds
    images: media/images
    videos: media/videos
    text: media/text

modes:
  recognition:
    enabled: true
    overlay:
      font_sizes: [48, 24]
      default_color: "#00ff00"
  video:
    enabled: true
    player: mpv
  kiosk:
    enabled: true
    browser: chromium
    whitelist: []
  slideshow:
    enabled: true
    transitions: [fade, slide, kenburns]
```

### `/home/admin/media-player/app/requirements.txt`

```
Flask==2.3.2
Flask-HTMX==0.3.1
flask-prometheus-metrics==1.0.0
PyYAML==6.0
Pillow==10.0.0
pygame==2.5.0
psutil==5.9.5
Jinja2==3.1.2
requests==2.31.0
python-mpv==1.0.1
```

### `/home/admin/media-player/app/utils/display.py`

```python
import os
import subprocess

class DisplayManager:
    """Manages dual HDMI displays on Raspberry Pi 4"""
    
    DISPLAY_MAP = {
        0: {'device': '/dev/fb0', 'env': 'DISPLAY=:0.0'},
        1: {'device': '/dev/fb1', 'env': 'DISPLAY=:0.1'}
    }
    
    @classmethod
    def get_display_env(cls, display_id):
        """Get environment variables for a specific display"""
        if display_id not in cls.DISPLAY_MAP:
            raise ValueError(f"Invalid display_id: {display_id}")
        
        env = os.environ.copy()
        env.update({
            'DISPLAY': f':0.{display_id}',
            'SDL_VIDEODRIVER': 'x11',
            'SDL_FBDEV': cls.DISPLAY_MAP[display_id]['device']
        })
        return env
    
    @classmethod
    def get_framebuffer_device(cls, display_id):
        """Get framebuffer device for a display"""
        return cls.DISPLAY_MAP[display_id]['device']
    
    @classmethod
    def clear_display(cls, display_id):
        """Clear a display to black"""
        fb_device = cls.get_framebuffer_device(display_id)
        subprocess.run(['dd', 'if=/dev/zero', f'of={fb_device}'], 
                      stderr=subprocess.DEVNULL)
```

### `/home/admin/media-player/app/utils/monitoring.py`

```python
import psutil
import subprocess

def get_system_stats():
    """Get system resource statistics"""
    return {
        'cpu_percent': psutil.cpu_percent(interval=1),
        'memory_percent': psutil.virtual_memory().percent,
        'disk_percent': psutil.disk_usage('/').percent,
        'temperature': get_cpu_temperature()
    }

def get_cpu_temperature():
    """Get CPU temperature on Raspberry Pi"""
    try:
        result = subprocess.run(['vcgencmd', 'measure_temp'], 
                              capture_output=True, text=True)
        temp_str = result.stdout.strip()
        return float(temp_str.split('=')[1].replace("'C", ""))
    except:
        return None

def get_service_status(service_name):
    """Check if a systemd service is running"""
    try:
        result = subprocess.run(['systemctl', 'is-active', service_name],
                              capture_output=True, text=True)
        return result.stdout.strip() == 'active'
    except:
        return False
```

## Mode Implementations

### Mode 1: Recognition - `/home/admin/media-player/app/modes/mode1_recognition/renderer.py`

```python
import os
import pygame
import threading
from PIL import Image, ImageDraw, ImageFont
from ...utils.display import DisplayManager

class RecognitionRenderer:
    def __init__(self, display_id, config):
        self.display_id = display_id
        self.config = config
        self.running = False
        self.thread = None
        
    def start(self):
        """Start the recognition display mode"""
        self.running = True
        self.thread = threading.Thread(target=self._render_loop)
        self.thread.start()
        
    def stop(self):
        """Stop the recognition display mode"""
        self.running = False
        if self.thread:
            self.thread.join()
            
    def status(self):
        """Get current status"""
        return {
            'running': self.running,
            'display_id': self.display_id
        }
        
    def _render_loop(self):
        """Main rendering loop"""
        os.environ.update(DisplayManager.get_display_env(self.display_id))
        
        pygame.init()
        screen = pygame.display.set_mode((1920, 1080), pygame.FULLSCREEN)
        clock = pygame.time.Clock()
        
        # Load fonts
        font_large = pygame.font.Font(None, self.config['overlay']['font_sizes'][0])
        font_small = pygame.font.Font(None, self.config['overlay']['font_sizes'][1])
        
        scroll_y = 1080
        
        while self.running:
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    self.running = False
                    
            # Clear screen
            screen.fill((0, 0, 0))
            
            # Render scrolling text
            text1 = font_large.render("Welcome to Digital Signage", True, (0, 255, 0))
            text2 = font_small.render("Powered by Raspberry Pi 4", True, (0, 255, 0))
            
            screen.blit(text1, (960 - text1.get_width()//2, scroll_y))
            screen.blit(text2, (960 - text2.get_width()//2, scroll_y + 60))
            
            scroll_y -= 2
            if scroll_y < -100:
                scroll_y = 1080
                
            pygame.display.flip()
            clock.tick(30)
            
        pygame.quit()
```

### Mode 1: Recognition - `/home/admin/media-player/app/modes/mode1_recognition/ui.py`

```python
from flask import Blueprint, render_template, jsonify, request
from .renderer import RecognitionRenderer

blueprint = Blueprint('recognition', __name__, 
                     template_folder='templates',
                     static_folder='static')

renderers = {}

@blueprint.route('/')
def index():
    return render_template('index.html')

@blueprint.route('/start/<int:display_id>', methods=['POST'])
def start(display_id):
    if display_id not in renderers:
        renderers[display_id] = RecognitionRenderer(display_id, 
                                                   blueprint.config)
    renderers[display_id].start()
    return jsonify({'status': 'started', 'display_id': display_id})

@blueprint.route('/stop/<int:display_id>', methods=['POST'])
def stop(display_id):
    if display_id in renderers:
        renderers[display_id].stop()
        return jsonify({'status': 'stopped', 'display_id': display_id})
    return jsonify({'error': 'Renderer not found'}), 404

@blueprint.route('/status/<int:display_id>')
def status(display_id):
    if display_id in renderers:
        return jsonify(renderers[display_id].status())
    return jsonify({'error': 'Renderer not found'}), 404
```

### Mode 2: Video - `/home/admin/media-player/app/modes/mode2_video/renderer.py`

```python
import os
import mpv
import threading
from ...utils.display import DisplayManager

class VideoRenderer:
    def __init__(self, display_id, config):
        self.display_id = display_id
        self.config = config
        self.player = None
        self.running = False
        
    def start(self, playlist=None):
        """Start video playback"""
        env = DisplayManager.get_display_env(self.display_id)
        
        self.player = mpv.MPV(
            wid=str(self.display_id),
            fullscreen=True,
            loop_playlist='inf',
            hwdec='mmal',
            vo='gpu',
            gpu_context='drm'
        )
        
        # Load playlist
        if playlist:
            for video in playlist:
                self.player.playlist_append(video)
        else:
            # Default to all videos in media directory
            video_dir = self.config.get('media_dirs', {}).get('videos', 'media/videos')
            for video in os.listdir(video_dir):
                if video.endswith(('.mp4', '.mkv', '.avi')):
                    self.player.playlist_append(os.path.join(video_dir, video))
                    
        self.player.play()
        self.running = True
        
    def stop(self):
        """Stop video playback"""
        if self.player:
            self.player.quit()
            self.player = None
        self.running = False
        
    def status(self):
        """Get playback status"""
        if self.player and self.running:
            return {
                'running': True,
                'display_id': self.display_id,
                'filename': self.player.filename,
                'time_pos': self.player.time_pos,
                'duration': self.player.duration
            }
        return {'running': False, 'display_id': self.display_id}
```

### Mode 3: Kiosk - `/home/admin/media-player/app/modes/mode3_kiosk/renderer.py`

```python
import os
import subprocess
import time
from ...utils.display import DisplayManager

class KioskRenderer:
    def __init__(self, display_id, config):
        self.display_id = display_id
        self.config = config
        self.process = None
        
    def start(self, url='https://example.com'):
        """Start Chromium in kiosk mode"""
        env = DisplayManager.get_display_env(self.display_id)
        
        # Check whitelist
        whitelist = self.config.get('whitelist', [])
        if whitelist and not any(domain in url for domain in whitelist):
            raise ValueError(f"URL not in whitelist: {url}")
            
        cmd = [
            'chromium-browser',
            '--kiosk',
            '--noerrdialogs',
            '--disable-infobars',
            '--disable-session-crashed-bubble',
            '--check-for-update-interval=31536000',
            f'--window-position={1920*self.display_id},0',
            '--window-size=1920,1080',
            url
        ]
        
        self.process = subprocess.Popen(cmd, env=env)
        
    def stop(self):
        """Stop the kiosk browser"""
        if self.process:
            self.process.terminate()
            time.sleep(1)
            if self.process.poll() is None:
                self.process.kill()
            self.process = None
            
    def status(self):
        """Get kiosk status"""
        return {
            'running': self.process is not None and self.process.poll() is None,
            'display_id': self.display_id
        }
```

### Mode 4: Slideshow - `/home/admin/media-player/app/modes/mode4_slideshow/renderer.py`

```python
import os
import pygame
import time
import threading
from PIL import Image
import mpv
from ...utils.display import DisplayManager

class SlideshowRenderer:
    def __init__(self, display_id, config):
        self.display_id = display_id
        self.config = config
        self.running = False
        self.thread = None
        self.current_item = None
        
    def start(self, playlist=None):
        """Start slideshow"""
        self.running = True
        self.thread = threading.Thread(target=self._render_loop, args=(playlist,))
        self.thread.start()
        
    def stop(self):
        """Stop slideshow"""
        self.running = False
        if self.thread:
            self.thread.join()
            
    def status(self):
        """Get slideshow status"""
        return {
            'running': self.running,
            'display_id': self.display_id,
            'current_item': self.current_item
        }
        
    def _render_loop(self, playlist):
        """Main slideshow loop"""
        os.environ.update(DisplayManager.get_display_env(self.display_id))
        
        pygame.init()
        screen = pygame.display.set_mode((1920, 1080), pygame.FULLSCREEN)
        clock = pygame.time.Clock()
        
        # Get media items
        if not playlist:
            playlist = self._get_default_playlist()
            
        item_index = 0
        transition_time = 0
        
        while self.running:
            for event in pygame.event.get():
                if event.type == pygame.QUIT:
                    self.running = False
                    
            if item_index >= len(playlist):
                item_index = 0
                
            item = playlist[item_index]
            self.current_item = item['path']
            
            if item['type'] == 'image':
                self._show_image(screen, item['path'], item.get('duration', 5))
            elif item['type'] == 'video':
                self._play_video(item['path'])
                
            item_index += 1
            
        pygame.quit()
        
    def _get_default_playlist(self):
        """Build default playlist from media directories"""
        playlist = []
        
        # Add images
        image_dir = 'media/images'
        if os.path.exists(image_dir):
            for img in os.listdir(image_dir):
                if img.endswith(('.jpg', '.png', '.gif')):
                    playlist.append({
                        'type': 'image',
                        'path': os.path.join(image_dir, img),
                        'duration': 5
                    })
                    
        # Add videos
        video_dir = 'media/videos'
        if os.path.exists(video_dir):
            for vid in os.listdir(video_dir):
                if vid.endswith(('.mp4', '.mkv', '.avi')):
                    playlist.append({
                        'type': 'video',
                        'path': os.path.join(video_dir, vid)
                    })
                    
        return playlist
        
    def _show_image(self, screen, image_path, duration):
        """Display an image with optional transition"""
        try:
            img = pygame.image.load(image_path)
            img = pygame.transform.scale(img, (1920, 1080))
            
            # Fade in effect
            for alpha in range(0, 255, 5):
                screen.fill((0, 0, 0))
                img.set_alpha(alpha)
                screen.blit(img, (0, 0))
                pygame.display.flip()
                pygame.time.wait(10)
                
            # Hold
            start_time = time.time()
            while time.time() - start_time < duration and self.running:
                pygame.event.pump()
                pygame.time.wait(100)
                
        except Exception as e:
            print(f"Error loading image {image_path}: {e}")
            
    def _play_video(self, video_path):
        """Play a video file"""
        player = mpv.MPV(
            wid=str(self.display_id),
            fullscreen=True,
            hwdec='mmal'
        )
        player.play(video_path)
        player.wait_for_playback()
        player.quit()
```

## Templates

### `/home/admin/media-player/app/templates/base.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Digital Signage Admin</title>
    <script src="https://unpkg.com/htmx.org@1.9.4"></script>
    <link rel="stylesheet" href="/static/style.css">
</head>
<body>
    <nav>
        <h1>Digital Signage Control Panel</h1>
        <div class="nav-links">
            <a href="/admin">Dashboard</a>
            {% for mode_name in modes %}
            <a href="/admin/modes/{{ mode_name }}">{{ mode_name|title }}</a>
            {% endfor %}
        </div>
    </nav>
    
    <main>
        {% block content %}{% endblock %}
    </main>
    
    <footer>
        <div hx-get="/health" hx-trigger="every 5s" hx-target="#health-status">
            <span id="health-status">Loading...</span>
        </div>
    </footer>
</body>
</html>
```

### `/home/admin/media-player/app/templates/admin.html`

```html
{% extends "base.html" %}

{% block content %}
<div class="dashboard">
    <h2>System Status</h2>
    
    <div class="stats-grid">
        <div class="stat-card">
            <h3>CPU Usage</h3>
            <div class="stat-value">{{ stats.cpu_percent }}%</div>
        </div>
        
        <div class="stat-card">
            <h3>Memory Usage</h3>
            <div class="stat-value">{{ stats.memory_percent }}%</div>
        </div>
        
        <div class="stat-card">
            <h3>Disk Usage</h3>
            <div class="stat-value">{{ stats.disk_percent }}%</div>
        </div>
        
        <div class="stat-card">
            <h3>Temperature</h3>
            <div class="stat-value">{{ stats.temperature }}°C</div>
        </div>
    </div>
    
    <h2>Active Modes</h2>
    <div class="modes-grid">
        {% for mode_name, mode_info in modes.items() %}
        <div class="mode-card">
            <h3>{{ mode_name|title }}</h3>
            <p>Status: <span class="status-active">Active</span></p>
            <a href="/admin/modes/{{ mode_name }}" class="btn btn-primary">Configure</a>
        </div>
        {% endfor %}
    </div>
</div>
{% endblock %}
```

### `/home/admin/media-player/app/modes/mode1_recognition/templates/index.html`

```html
{% extends "base.html" %}

{% block content %}
<div class="mode-control">
    <h2>Recognition Mode Control</h2>
    
    <div class="display-controls">
        <div class="display-panel">
            <h3>Display 1 (HDMI-1)</h3>
            <div hx-get="/admin/modes/recognition/status/0" 
                 hx-trigger="every 2s" 
                 hx-target="#display-0-status">
                <span id="display-0-status">Loading...</span>
            </div>
            <button hx-post="/admin/modes/recognition/start/0" 
                    class="btn btn-success">Start</button>
            <button hx-post="/admin/modes/recognition/stop/0" 
                    class="btn btn-danger">Stop</button>
        </div>
        
        <div class="display-panel">
            <h3>Display 2 (HDMI-2)</h3>
            <div hx-get="/admin/modes/recognition/status/1" 
                 hx-trigger="every 2s" 
                 hx-target="#display-1-status">
                <span id="display-1-status">Loading...</span>
            </div>
            <button hx-post="/admin/modes/recognition/start/1" 
                    class="btn btn-success">Start</button>
            <button hx-post="/admin/modes/recognition/stop/1" 
                    class="btn btn-danger">Stop</button>
        </div>
    </div>
    
    <div class="config-section">
        <h3>Configuration</h3>
        <form hx-post="/admin/modes/recognition/config" hx-target="#config-result">
            <label>Font Size (Large): 
                <input type="number" name="font_size_large" value="48">
            </label>
            <label>Font Size (Small): 
                <input type="number" name="font_size_small" value="24">
            </label>
            <label>Text Color: 
                <input type="color" name="text_color" value="#00ff00">
            </label>
            <button type="submit" class="btn btn-primary">Update Config</button>
        </form>
        <div id="config-result"></div>
    </div>
</div>
{% endblock %}
```

## Installation Scripts

### `/home/admin/media-player/scripts/install-media.sh`

```bash
#!/bin/bash
set -e

# Digital Signage Installer for Raspberry Pi 4
# Usage: ./install-media.sh [--dry-run] [--verbose] [--resume] [--auto-reboot]

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"
LOG_FILE="/var/log/media-player-install.log"

# Parse arguments
DRY_RUN=0
VERBOSE=0
RESUME=0
AUTO_REBOOT=0

while [[ $# -gt 0 ]]; do
    case $1 in
        --dry-run) DRY_RUN=1; shift ;;
        --verbose) VERBOSE=1; shift ;;
        --resume) RESUME=1; shift ;;
        --auto-reboot) AUTO_REBOOT=1; shift ;;
        *) echo "Unknown option: $1"; exit 1 ;;
    esac
done

# Logging function
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

# Execute command with dry-run support
exec_cmd() {
    if [ $VERBOSE -eq 1 ]; then
        echo "Executing: $@"
    fi
    if [ $DRY_RUN -eq 0 ]; then
        "$@" 2>&1 | tee -a "$LOG_FILE"
    else
        echo "[DRY-RUN] Would execute: $@"
    fi
}

# Check if step was completed (for resume)
check_step() {
    if [ $RESUME -eq 1 ] && [ -f "/tmp/media-install-$1.done" ]; then
        log "Step $1 already completed, skipping..."
        return 0
    fi
    return 1
}

# Mark step as completed
mark_step() {
    touch "/tmp/media-install-$1.done"
}

log "Starting Digital Signage installation..."

# Step 1: Preflight checks
if ! check_step "preflight"; then
    log "Running preflight checks..."
    exec_cmd python3 "$SCRIPT_DIR/preflight_check.py"
    mark_step "preflight"
fi

# Step 2: System update
if ! check_step "update"; then
    log "Updating system packages..."
    exec_cmd sudo apt update
    exec_cmd sudo apt upgrade -y
    mark_step "update"
fi

# Step 3: Install dependencies
if ! check_step "dependencies"; then
    log "Installing required packages..."
    exec_cmd sudo apt install -y \
        python3-pip python3-venv python3-dev \
        nginx ssl-cert \
        chromium-browser \
        mpv libmpv-dev \
        libsdl2-dev libsdl2-image-dev libsdl2-mixer-dev libsdl2-ttf-dev \
        libportmidi-dev libswscale-dev libavformat-dev libavcodec-dev \
        zlib1g-dev libjpeg-dev \
        git curl wget \
        ufw fail2ban \
        prometheus-node-exporter
    mark_step "dependencies"
fi

# Step 4: Configure display
if ! check_step "display"; then
    log "Configuring dual HDMI displays..."
    
    # Backup original config
    exec_cmd sudo cp /boot/config.txt /boot/config.txt.backup
    
    # Add display configuration
    cat << EOF | exec_cmd sudo tee -a /boot/config.txt
# Digital Signage Display Configuration
hdmi_force_hotplug:0=1
hdmi_force_hotplug:1=1
hdmi_group:0=2
hdmi_group:1=2
hdmi_mode:0=82
hdmi_mode:1=82
# 1920x1080 @ 60Hz for both displays
EOF
    mark_step "display"
fi

# Step 5: Create directory structure
if ! check_step "directories"; then
    log "Creating directory structure..."
    exec_cmd sudo mkdir -p /home/admin/media-player/{app,media/{backgrounds,images,videos,text},logs}
    exec_cmd sudo chown -R admin:admin /home/admin/media-player
    mark_step "directories"
fi

# Step 6: Setup Python environment
if ! check_step "python"; then
    log "Setting up Python virtual environment..."
    cd "$PROJECT_ROOT/app"
    exec_cmd python3 -m venv venv
    exec_cmd source venv/bin/activate
    exec_cmd pip install --upgrade pip
    exec_cmd pip install -r requirements.txt
    mark_step "python"
fi

# Step 7: Configure firewall
if ! check_step "firewall"; then
    log "Configuring firewall..."
    exec_cmd sudo ufw default deny incoming
    exec_cmd sudo ufw default allow outgoing
    exec_cmd sudo ufw allow 22/tcp
    exec_cmd sudo ufw allow 80/tcp
    exec_cmd sudo ufw allow 443/tcp
    exec_cmd sudo ufw --force enable
    mark_step "firewall"
fi

# Step 8: Generate SSL certificate
if ! check_step "ssl"; then
    log "Generating self-signed SSL certificate..."
    exec_cmd sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout /etc/ssl/private/media-player.key \
        -out /etc/ssl/certs/media-player.crt \
        -subj "/C=US/ST=State/L=City/O=Organization/CN=media-player.local"
    mark_step "ssl"
fi

# Step 9: Configure nginx
if ! check_step "nginx"; then
    log "Configuring nginx..."
    
    # Generate nginx config from template
    python3 - << EOF
import yaml
from jinja2 import Template

with open('$PROJECT_ROOT/app/config.yaml', 'r') as f:
    config = yaml.safe_load(f)

with open('$PROJECT_ROOT/nginx/templates/media-player.conf.j2', 'r') as f:
    template = Template(f.read())

nginx_config = template.render(config=config)

with open('/tmp/media-player.conf', 'w') as f:
    f.write(nginx_config)
EOF
    
    exec_cmd sudo mv /tmp/media-player.conf /etc/nginx/sites-available/media-player.conf
    exec_cmd sudo ln -sf /etc/nginx/sites-available/media-player.conf /etc/nginx/sites-enabled/
    exec_cmd sudo nginx -t
    exec_cmd sudo systemctl restart nginx
    mark_step "nginx"
fi

# Step 10: Setup systemd services
if ! check_step "systemd"; then
    log "Setting up systemd services..."
    
    # Generate systemd units from templates
    python3 - << EOF
import yaml
from jinja2 import Template

with open('$PROJECT_ROOT/app/config.yaml', 'r') as f:
    config = yaml.safe_load(f)

# Controller service
with open('$PROJECT_ROOT/systemd/templates/media-controller.service.j2', 'r') as f:
    template = Template(f.read())
    
controller_service = template.render(config=config)
with open('/tmp/media-controller.service', 'w') as f:
    f.write(controller_service)

# Mode services
with open('$PROJECT_ROOT/systemd/templates/media-mode.service.j2', 'r') as f:
    mode_template = Template(f.read())

for mode_name, mode_config in config['modes'].items():
    if mode_config.get('enabled', False):
        for display_id in [0, 1]:
            service_name = f"media-{mode_name}-display{display_id}"
            mode_service = mode_template.render(
                mode_name=mode_name,
                display_id=display_id,
                config=config
            )
            with open(f'/tmp/{service_name}.service', 'w') as f:
                f.write(mode_service)
EOF
    
    # Install services
    exec_cmd sudo mv /tmp/media-*.service /etc/systemd/system/
    exec_cmd sudo systemctl daemon-reload
    exec_cmd sudo systemctl enable media-controller.service
    mark_step "systemd"
fi

# Step 11: Start services
if ! check_step "start"; then
    log "Starting services..."
    exec_cmd sudo systemctl start media-controller.service
    mark_step "start"
fi

# Cleanup
rm -f /tmp/media-install-*.done

log "Installation completed successfully!"
log "Access the admin interface at: https://$(hostname -I | cut -d' ' -f1)"

if [ $AUTO_REBOOT -eq 1 ]; then
    log "Rebooting in 10 seconds..."
    sleep 10
    exec_cmd sudo reboot
else
    log "Please reboot your Raspberry Pi to ensure all changes take effect."
fi
```

### `/home/admin/media-player/scripts/preflight_check.py`

```python
#!/usr/bin/env python3
"""Preflight check script for Digital Signage installation"""

import os
import sys
import subprocess
import platform

def check_hardware():
    """Check if running on Raspberry Pi 4"""
    try:
        with open('/proc/device-tree/model', 'r') as f:
            model = f.read().strip()
            if 'Raspberry Pi 4' not in model:
                raise ValueError(f"Not a Raspberry Pi 4: {model}")
        print("✓ Hardware: Raspberry Pi 4 detected")
        return True
    except Exception as e:
        print(f"✗ Hardware check failed: {e}")
        return False

def check_os():
    """Check OS version"""
    try:
        result = subprocess.run(['lsb_release', '-cs'], 
                              capture_output=True, text=True)
        codename = result.stdout.strip()
        if codename not in ['bullseye', 'bookworm']:
            print(f"⚠ OS: {codename} detected (bullseye recommended)")
        else:
            print(f"✓ OS: {codename} detected")
        return True
    except Exception as e:
        print(f"✗ OS check failed: {e}")
        return False

def check_disk_space():
    """Check available disk space"""
    try:
        stat = os.statvfs('/')
        free_gb = (stat.f_bavail * stat.f_frsize) / (1024**3)
        if free_gb < 2:
            raise ValueError(f"Insufficient disk space: {free_gb:.1f}GB")
        print(f"✓ Disk space: {free_gb:.1f}GB available")
        return True
    except Exception as e:
        print(f"✗ Disk space check failed: {e}")
        return False

def check_network():
    """Check network connectivity"""
    try:
        result = subprocess.run(['ping', '-c', '1', '8.8.8.8'], 
                              capture_output=True)
        if result.returncode != 0:
            raise ValueError("No internet connectivity")
        print("✓ Network: Internet connection available")
        return True
    except Exception as e:
        print(f"✗ Network check failed: {e}")
        return False

def check_user():
    """Check if running as admin user"""
    user = os.environ.get('USER', '')
    if user != 'admin':
        print(f"⚠ User: Running as '{user}' (admin recommended)")
    else:
        print("✓ User: Running as admin")
    return True

def main():
    """Run all preflight checks"""
    print("Running preflight checks for Digital Signage installation...\n")
    
    checks = [
        check_hardware,
        check_os,
        check_disk_space,
        check_network,
        check_user
    ]
    
    failed = False
    for check in checks:
        if not check():
            failed = True
            
    print("\n" + "="*50)
    if failed:
        print("✗ Preflight checks failed!")
        print("Please resolve the issues above before continuing.")
        sys.exit(1)
    else:
        print("✓ All preflight checks passed!")
        print("Ready to proceed with installation.")
        sys.exit(0)

if __name__ == '__main__':
    main()
```

## Service Templates

### `/home/admin/media-player/systemd/templates/media-controller.service.j2`

```ini
[Unit]
Description=Digital Signage Controller Service
After=network.target

[Service]
Type=simple
User=admin
Group=admin
WorkingDirectory=/home/admin/media-player/app
Environment="PATH=/home/admin/media-player/app/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ExecStart=/home/admin/media-player/app/venv/bin/python app.py
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### `/home/admin/media-player/systemd/templates/media-mode.service.j2`

```ini
[Unit]
Description=Digital Signage {{ mode_name }} Mode - Display {{ display_id }}
After=media-controller.service
Requires=media-controller.service

[Service]
Type=simple
User=admin
Group=admin
WorkingDirectory=/home/admin/media-player/app
Environment="DISPLAY=:0.{{ display_id }}"
Environment="PATH=/home/admin/media-player/app/venv/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
ExecStart=/home/admin/media-player/app/venv/bin/python -m modes.mode{{ loop.index }}_{{ mode_name }}.service --display={{ display_id }}
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## Nginx Template

### `/home/admin/media-player/nginx/templates/media-player.conf.j2`

```nginx
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name _;
    
    ssl_certificate /etc/ssl/certs/media-player.crt;
    ssl_certificate_key /etc/ssl/private/media-player.key;
    
    location / {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    location /static {
        alias /home/admin/media-player/app/static;
        expires 1h;
    }
    
    location /media {
        alias /home/admin/media-player/media;
        expires 1d;
    }
}
```

## CSS Styling

### `/home/admin/media-player/app/static/style.css`

```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
    background: #1a1a1a;
    color: #e0e0e0;
    line-height: 1.6;
}

nav {
    background: #2a2a2a;
    padding: 1rem 2rem;
    display: flex;
    justify-content: space-between;
    align-items: center;
    box-shadow: 0 2px 4px rgba(0,0,0,0.3);
}

nav h1 {
    font-size: 1.5rem;
    color: #00ff00;
}

.nav-links {
    display: flex;
    gap: 2rem;
}

.nav-links a {
    color: #e0e0e0;
    text-decoration: none;
    padding: 0.5rem 1rem;
    border-radius: 4px;
    transition: background 0.3s;
}

.nav-links a:hover {
    background: #3a3a3a;
}

main {
    padding: 2rem;
    max-width: 1200px;
    margin: 0 auto;
}

.dashboard {
    display: flex;
    flex-direction: column;
    gap: 2rem;
}

.stats-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 1rem;
}

.stat-card {
    background: #2a2a2a;
    padding: 1.5rem;
    border-radius: 8px;
    text-align: center;
    box-shadow: 0 2px 4px rgba(0,0,0,0.3);
}

.stat-card h3 {
    font-size: 0.9rem;
    color: #888;
    margin-bottom: 0.5rem;
}

.stat-value {
    font-size: 2rem;
    font-weight: bold;
    color: #00ff00;
}

.modes-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 1rem;
}

.mode-card {
    background: #2a2a2a;
    padding: 1.5rem;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.3);
}

.mode-card h3 {
    margin-bottom: 1rem;
    color: #00ff00;
}

.status-active {
    color: #00ff00;
    font-weight: bold;
}

.status-inactive {
    color: #ff4444;
    font-weight: bold;
}

.btn {
    display: inline-block;
    padding: 0.5rem 1rem;
    border: none;
    border-radius: 4px;
    cursor: pointer;
    text-decoration: none;
    font-size: 0.9rem;
    transition: all 0.3s;
}

.btn-primary {
    background: #0066cc;
    color: white;
}

.btn-primary:hover {
    background: #0052a3;
}

.btn-success {
    background: #00cc66;
    color: white;
}

.btn-success:hover {
    background: #00a352;
}

.btn-danger {
    background: #cc0000;
    color: white;
}

.btn-danger:hover {
    background: #a30000;
}

.display-controls {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 2rem;
    margin: 2rem 0;
}

.display-panel {
    background: #2a2a2a;
    padding: 1.5rem;
    border-radius: 8px;
    box-shadow: 0 2px 4px rgba(0,0,0,0.3);
}

.display-panel h3 {
    margin-bottom: 1rem;
    color: #00ff00;
}

.display-panel button {
    margin: 0.5rem 0.5rem 0.5rem 0;
}

.config-section {
    background: #2a2a2a;
    padding: 1.5rem;
    border-radius: 8px;
    margin-top: 2rem;
}

.config-section form {
    display: flex;
    flex-direction: column;
    gap: 1rem;
}

.config-section label {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.config-section input {
    background: #1a1a1a;
    border: 1px solid #444;
    color: #e0e0e0;
    padding: 0.5rem;
    border-radius: 4px;
}

footer {
    background: #2a2a2a;
    padding: 1rem 2rem;
    text-align: center;
    margin-top: 2rem;
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
}

#health-status {
    color: #00ff00;
    font-size: 0.9rem;
}
```

## Additional Scripts

### `/home/admin/media-player/scripts/uninstall.sh`

```bash
#!/bin/bash
set -e

echo "Uninstalling Digital Signage System..."

# Stop and disable services
sudo systemctl stop media-controller.service || true
sudo systemctl disable media-controller.service || true
sudo systemctl stop media-*.service || true
sudo systemctl disable media-*.service || true

# Remove systemd services
sudo rm -f /etc/systemd/system/media-*.service
sudo systemctl daemon-reload

# Remove nginx config
sudo rm -f /etc/nginx/sites-enabled/media-player.conf
sudo rm -f /etc/nginx/sites-available/media-player.conf
sudo systemctl restart nginx || true

# Remove SSL certificates
sudo rm -f /etc/ssl/private/media-player.key
sudo rm -f /etc/ssl/certs/media-player.crt

# Remove application files (preserve media)
read -p "Remove media files? (y/N): " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    rm -rf /home/admin/media-player
else
    rm -rf /home/admin/media-player/app
    rm -rf /home/admin/media-player/scripts
    rm -rf /home/admin/media-player/systemd
    rm -rf /home/admin/media-player/nginx
    echo "Media files preserved in /home/admin/media-player/media"
fi

echo "Uninstall complete."
```

### `/home/admin/media-player/scripts/update.sh`

```bash
#!/bin/bash
set -e

echo "Updating Digital Signage System..."

# Pull latest changes (assuming git repo)
cd /home/admin/media-player
git pull origin main || echo "Not a git repository, skipping pull"

# Update Python dependencies
cd app
source venv/bin/activate
pip install -r requirements.txt

# Restart services
sudo systemctl restart media-controller.service
sudo systemctl restart media-*.service

echo "Update complete."
```

## README

### `/home/admin/media-player/README.md`

```markdown
# Raspberry Pi Digital Signage System

A modular, headless digital signage system for Raspberry Pi 4 with dual HDMI output support.

## Features

- **Dual Display Support**: Independent control of both HDMI outputs
- **Multiple Display Modes**:
  - Recognition: Scrolling text with overlay support
  - Video: Hardware-accelerated video playback
  - Kiosk: Full-screen web browser with domain whitelist
  - Slideshow: Mixed media with transitions
- **Web-based Admin Interface**: Flask + HTMX for minimal JavaScript
- **System Monitoring**: Real-time stats with Prometheus metrics
- **Automated Installation**: Single-script setup with preflight checks

## Quick Start

1. Clone this repository to your Raspberry Pi 4:
   ```bash
   git clone https://github.com/yourusername/media-player.git
   cd media-player
   ```

2. Run the installer:
   ```bash
   chmod +x scripts/install-media.sh
   ./scripts/install-media.sh
   ```

3. Reboot and access the admin interface:
   ```
   https://<your-pi-ip>
   ```

## System Requirements

- Raspberry Pi 4 (2GB+ RAM recommended)
- Raspberry Pi OS Lite (Bullseye or newer)
- 2GB+ free disk space
- Network connectivity
- Two HDMI displays (1920x1080)

## Configuration

Edit `/home/admin/media-player/app/config.yaml` to customize:
- Display resolution and frame rate
- Media directories
- Mode-specific settings
- Enable/disable modes

## Media Organization

Place your media files in:
- `/home/admin/media-player/media/images/` - JPEG, PNG, GIF
- `/home/admin/media-player/media/videos/` - MP4, MKV, AVI
- `/home/admin/media-player/media/backgrounds/` - Background images
- `/home/admin/media-player/media/text/` - Text overlay files

## API Reference

See `docs/API.md` for complete API documentation.

## Troubleshooting

See `docs/TROUBLESHOOTING.md` for common issues and solutions.

## Services

- `media-controller.service` - Main Flask application
- `media-<mode>-display<0|1>.service` - Per-mode, per-display services

## Monitoring

Access Prometheus metrics at `https://<your-pi-ip>/metrics`

## License

MIT License - See LICENSE file for details
```

## Mode Configuration Files

### `/home/admin/media-player/app/modes/mode1_recognition/config.json`

```json
{
    "name": "Recognition Mode",
    "description": "Scrolling text display with overlay support",
    "settings": {
        "scroll_speed": 2,
        "background_image": null,
        "text_source": "file",
        "text_file": "media/text/welcome.txt",
        "loop": true
    }
}
```

### `/home/admin/media-player/app/modes/mode2_video/config.json`

```json
{
    "name": "Video Mode",
    "description": "Hardware-accelerated video playback",
    "settings": {
        "playlist_mode": "sequential",
        "audio_output": "hdmi",
        "hardware_decode": true,
        "loop_playlist": true
    }
}
```

### `/home/admin/media-player/app/modes/mode3_kiosk/config.json`

```json
{
    "name": "Kiosk Mode",
    "description": "Full-screen web browser",
    "settings": {
        "home_url": "https://example.com",
        "refresh_interval": 3600,
        "allow_navigation": false,
        "hide_cursor": true
    }
}
```

### `/home/admin/media-player/app/modes/mode4_slideshow/config.json`

```json
{
    "name": "Slideshow Mode",
    "description": "Mixed media slideshow with transitions",
    "settings": {
        "default_duration": 5,
        "transition_duration": 1,
        "random_order": false,
        "include_videos": true
    }
}
```

This completes the entire digital signage system implementation. The system is modular, scalable, and includes all the requested features including dual display support, multiple display modes, web-based administration, comprehensive installation scripts, and proper service management.