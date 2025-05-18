#!/bin/bash

# --- Helper Functions ---
check_command() {
  if ! command -v "$1" &> /dev/null; then
    echo "Error: '$1' is not installed. Please install it (e.g., 'sudo apt install $1') and try again."
    exit 1
  else
    echo "'$1' found."
  fi
}

# --- Step 1/6: Checking Prerequisites ---
echo "Step 1/6: Checking for essential tools..."
check_command python3
check_command pip3
check_command pactl
check_command wget
check_command tar
check_command build-essential
check_command git
check_command pkg-config

# Check for FFmpeg later, as we attempt to install it

echo "Prerequisites check complete."
echo "-------------------------------------------------------------------------"

# --- Step 2/6: Setting up Virtual Environment and Installing Flask ---
echo "Step 2/6: Setting up virtual environment and installing Flask..."
PROJECT_DIR="audio_streamer"
if [ -d "$PROJECT_DIR" ]; then
    echo "Project directory '$PROJECT_DIR' already exists."
else
    echo "Creating project directory '$PROJECT_DIR'..."
    mkdir "$PROJECT_DIR"
fi
cd "$PROJECT_DIR"

if [ -d "venv" ]; then
    echo "Virtual environment already exists. Activating..."
    source venv/bin/activate
else
    echo "Creating virtual environment..."
    python3 -m venv venv
    source venv/bin/activate
    echo "Installing Flask..."
    pip3 install Flask
    echo "Flask installed."
fi
echo "Virtual environment setup and Flask installed."
echo "-------------------------------------------------------------------------"

# --- Step 3/6: Installing FFmpeg ---
echo "Step 3/6: Installing FFmpeg..."
if check_command ffmpeg; then
    echo "FFmpeg is already installed."
else
    echo "FFmpeg not found. Attempting a comprehensive installation..."
    sudo apt update
    sudo apt install -y libpulse-dev libasound2-dev libv4l-dev libx11-dev libxext-dev libxfixes-dev libxv-dev libxvidcore-0-dev libx264-dev libmp3lame-dev libopus-dev libvpx-dev zlib1g-dev
    FFMPEG_VERSION="$(wget -qO- https://api.github.com/repos/FFmpeg/FFmpeg/releases/latest | grep tag_name | cut -d '"' -f 4)"
    if [ -z "$FFMPEG_VERSION" ]; then
        FFMPEG_VERSION="n6.1.1"
    fi
    echo "Downloading FFmpeg version: $FFMPEG_VERSION"
    wget "https://github.com/FFmpeg/FFmpeg/archive/$FFMPEG_VERSION.tar.gz" -O ffmpeg-$FFMPEG_VERSION.tar.gz
    tar -xzf "ffmpeg-$FFMPEG_VERSION.tar.gz"
    cd "FFmpeg-$FFMPEG_VERSION"
    ./configure --enable-gpl --enable-version3 --enable-nonfree --enable-libpulse --enable-libmp3lame --enable-libopus --enable-libvpx --enable-x11grab --enable-alsa --enable-libv4l2 --enable-zlib
    make -j$(nproc)
    if [ $? -ne 0 ]; then
        echo "Error: FFmpeg build failed. Please check the output above for errors."
        exit 1
    fi
    sudo make install
    cd ../..
    rm -rf "$PROJECT_DIR/FFmpeg-$FFMPEG_VERSION" "$PROJECT_DIR/ffmpeg-$FFMPEG_VERSION.tar.gz"
    if check_command ffmpeg; then
        echo "FFmpeg installed successfully."
    else
        echo "Error: FFmpeg installation failed."
        exit 1
    fi
fi
echo "FFmpeg installation complete."
echo "-------------------------------------------------------------------------"

# --- Step 4/6: Setting up Audio Capture Sink (PulseAudio) ---
echo "Step 4/6: Setting up Audio Capture Sink (PulseAudio)..."
NULL_SINK_NAME="virtual-audio-capture"
NULL_SINK_MODULE="module-null-sink"

if ! pactl list short sinks | grep -q "$NULL_SINK_NAME"; then
    echo "Creating PulseAudio virtual sink '$NULL_SINK_NAME'..."
    if pactl load-module "$NULL_SINK_MODULE" sink_name="$NULL_SINK_NAME" sink_properties='device.description="Virtual Audio Capture"' > /dev/null 2>&1; then
        echo "PulseAudio virtual sink created successfully."
    else
        echo "Error: Failed to create PulseAudio virtual sink. Please check PulseAudio configuration."
    fi
else
    echo "PulseAudio virtual sink '$NULL_SINK_NAME' already exists."
fi
echo "PulseAudio sink setup complete."
echo "-------------------------------------------------------------------------"

# --- Step 5/6: Creating Flask Application (audio.py) ---
echo "Step 5/6: Creating Flask application 'audio.py'..."
cat > audio.py <<EOL
from flask import Flask, Response
import subprocess
import os

app = Flask(__name__)

AUDIO_SOURCE = "virtual-audio-capture.monitor"

def generate():
    process = subprocess.Popen(
        ['ffmpeg', '-f', 'pulse', '-i', AUDIO_SOURCE, '-vn', '-acodec', 'pcm_s16le', '-f', 'wav', '-'],
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE
    )
    try:
        while True:
            chunk = process.stdout.read(1024)
            if not chunk:
                break
            yield chunk
    finally:
        process.stdout.close()
        process.stderr.close()
        process.terminate()
        process.wait()
        if process.returncode != 0:
            try:
                error_output = process.stderr.read().decode()
                print(f"FFmpeg error on exit: {error_output}")
            except ValueError:
                print("FFmpeg process exited with an error, but stderr was already closed.")

@app.route('/stream')
def stream():
    return Response(generate(), mimetype='audio/wav')

@app.route('/')
def index():
    return "Simple Raw WAV Audio Streaming Server - Go to /stream to listen"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True, threaded=True)
EOL
echo "Flask application 'audio.py' created."
echo "-------------------------------------------------------------------------"

# --- Step 6/6: Running and Next Steps ---
echo "Step 6/6: Running the audio stream and next steps..."
echo ""
echo "To start the audio stream, run: python audio.py"
echo "Then, open your web browser and go to http://localhost:5000/stream"
echo ""
echo "IMPORTANT: You need to manually configure your system's audio output (e.g., browser audio)"
echo "to go to the '$NULL_SINK_NAME' sink. You can do this using the PulseAudio Volume Control"
echo "(pavucontrol) or similar audio settings for your desktop environment."
echo ""
echo "To potentially automate the audio loopback (you might need to adjust the sink name):"
echo "  pactl load-module module-loopback sink='$NULL_SINK_NAME' source='$(pactl get-default-sink).monitor'"
echo ""
echo "To stop the virtual sink and loopback later:"
echo "  pactl unload-module module-null-sink"
echo "  pactl unload-module module-loopback"
echo "-------------------------------------------------------------------------"
echo "Setup complete!"
