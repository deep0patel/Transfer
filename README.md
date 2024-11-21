```
import os
import base64
import socketio
import eventlet
import eventlet.wsgi
from flask import Flask

#  Configuration
PORT = 32044
FALLBACK_SAVE_DIR = os.path.join(os.getcwd(), "bricks-files")

#  Ensure fallback directory exists
os.makedirs(FALLBACK_SAVE_DIR, exist_ok=True)

# Initialize Flask and Socket.IO
app = Flask(__name__)
sio = socketio.Server(cors_allowed_origins="*", max_http_buffer_size=100 * 1024 * 1024)
app.wsgi_app = socketio.WSGIApp(sio, app.wsgi_app)

# Global variable to store the custom path
CURRENT_CUSTOM_PATH = FALLBACK_SAVE_DIR

def sanitize_path(base_path, file_path):
    """
    Sanitize and validate the save path to prevent directory traversal
    and ensure we're not writing outside of allowed directories.
    """
    # Remove leading slash if present
    file_path = file_path.lstrip('/')
    
    # Normalize the path to remove any .. or .
    base_path = os.path.normpath(base_path)
    full_path = os.path.normpath(os.path.join(base_path, file_path))
    
    # Check if the full path is within the base path
    if not full_path.startswith(os.path.normpath(base_path)):
        raise ValueError(f"Invalid path: Cannot save outside of {base_path}")
    
    return full_path

@sio.on('connect')
def connect(sid, environ):
    global CURRENT_CUSTOM_PATH
    CURRENT_CUSTOM_PATH = FALLBACK_SAVE_DIR
    print("Figma plugin connected")
    sio.emit('pong', 'pong', room=sid)

@sio.on('sending-path')
def handle_path(sid, custom_path):
    global CURRENT_CUSTOM_PATH
    CURRENT_CUSTOM_PATH = custom_path
    print(f"Custom path received and stored: {CURRENT_CUSTOM_PATH}")

@sio.on('code-generation')
def handle_code_generation(sid, data):
    global CURRENT_CUSTOM_PATH
    
    try:
        # Validate input
        if not data or 'files' not in data or not isinstance(data['files'], list):
            sio.emit('code-generation-response', {
                'status': 'error', 
                'error': 'Invalid data'
            }, room=sid)
            return

        # Use the globally stored custom path
        base_path = CURRENT_CUSTOM_PATH
        
        # Ensure base path exists
        os.makedirs(base_path, exist_ok=True)

        # Process and save files
        saved_files = []
        for file_data in data['files']:
            file_content = file_data.get('content', '')
            file_path = file_data.get('path', '')

            try:
                # Sanitize and get full save path
                full_save_path = sanitize_path(base_path, file_path)
                
                # Ensure directory exists
                os.makedirs(os.path.dirname(full_save_path), exist_ok=True)

                # Determine file type and decode accordingly
                if file_path.lower().endswith('.svg'):
                    # For SVG, check if it's base64 encoded and decode if necessary
                    try:
                        # Attempt to decode as base64
                        file_content = base64.b64decode(file_content).decode('utf-8')
                    except Exception:
                        # If decoding fails, assume it's plain text
                        pass
                    
                    # Write SVG content as text
                    with open(full_save_path, 'w', encoding='utf-8') as f:
                        f.write(file_content)

                elif file_path.lower().endswith('.png'):
                    # Decode base64 for PNG files
                    file_content = base64.b64decode(file_content)
                    with open(full_save_path, 'wb') as f:
                        f.write(file_content)
                else:
                    # Write other files as plain text
                    with open(full_save_path, 'w', encoding='utf-8') as f:
                        f.write(file_content)

                saved_files.append(full_save_path)
                print(f"Saved file: {full_save_path}")

            except (PermissionError, ValueError) as path_error:
                print(f"Error saving file {file_path}: {path_error}")
                sio.emit('code-generation-response', {
                    'status': 'error', 
                    'error': str(path_error)
                }, room=sid)
                continue

        print(f"Files saved to {base_path}. Total saved: {len(saved_files)}")
        sio.emit('code-generation-response', {
            'status': 'ok', 
            'savedFiles': saved_files
        }, room=sid)

    except Exception as e:
        print(f"Error processing files: {e}")
        sio.emit('code-generation-response', {
            'status': 'error', 
            'error': str(e)
        }, room=sid)

@sio.on('disconnect')
def disconnect(sid):
    global CURRENT_CUSTOM_PATH
    CURRENT_CUSTOM_PATH = FALLBACK_SAVE_DIR
    print("Figma plugin disconnected")

def run_server():
    print(f"Server is running on http://localhost:{PORT}")
    print(f"Fallback save directory: {FALLBACK_SAVE_DIR}")
    eventlet.wsgi.server(eventlet.listen(('', PORT)), app)

if __name__ == '__main__':
    run_server()

```
