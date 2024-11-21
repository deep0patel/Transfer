import socketio
import eventlet
import os
import base64

# Create a Socket.IO server
sio = socketio.Server(cors_allowed_origins="*")

# Default directory to save files
DEFAULT_SAVE_DIR = os.path.join(os.getcwd(), "bricks-files")
custom_save_dir = DEFAULT_SAVE_DIR  # Variable to hold the custom path

# Ensure the default directory exists
if not os.path.exists(DEFAULT_SAVE_DIR):
    os.makedirs(DEFAULT_SAVE_DIR)

# Helper function to write files
def save_file(file_path, content, is_base64=False):
    # Ensure directory exists
    os.makedirs(os.path.dirname(file_path), exist_ok=True)

    # Write file content
    mode = "wb" if is_base64 else "w"
    with open(file_path, mode) as f:
        if is_base64:
            f.write(base64.b64decode(content))
        else:
            f.write(content)

# Handle connection event
@sio.event
def connect(sid, environ):
    print(f"Figma plugin connected: {sid}")
    sio.emit("pong", "pong", to=sid)

# Handle receiving a custom path
@sio.event
def sending_path(sid, custom_path):
    global custom_save_dir
    print(f"Custom path received: {custom_path}")
    if os.path.isabs(custom_path):
        custom_save_dir = custom_path
    else:
        print("Invalid custom path provided. Falling back to default.")

# Handle code generation
@sio.event
def code_generation(sid, data):
    global custom_save_dir
    try:
        # Validate incoming data
        if not data or not isinstance(data.get("files"), list):
            print("Invalid data received")
            return {"status": "error", "error": "Invalid data structure"}

        files = data["files"]

        # Use custom path if valid, otherwise fallback to default
        save_dir = custom_save_dir if os.path.exists(custom_save_dir) else DEFAULT_SAVE_DIR

        for file in files:
            file_content = file.get("content")
            file_path = file.get("path")

            if not file_content or not file_path:
                print("Invalid file structure")
                continue

            # Construct the full file save path
            full_save_path = os.path.join(save_dir, file_path)

            # Detect if content is base64-encoded
            is_base64 = file_path.endswith((".png", ".svg"))
            save_file(full_save_path, file_content, is_base64=is_base64)

        print(f"Files saved successfully to {save_dir}")
        return {"status": "ok"}
    except Exception as e:
        print(f"Error processing files: {e}")
        return {"status": "error", "error": str(e)}

# Handle disconnection
@sio.event
def disconnect(sid):
    print(f"Figma plugin disconnected: {sid}")

# Wrap the server with a WSGI app
app = socketio.WSGIApp(sio)

if __name__ == "__main__":
    PORT = 32044
    print(f"Server is running on http://localhost:{PORT}")
    eventlet.wsgi.server(eventlet.listen(("0.0.0.0", PORT)), app)
