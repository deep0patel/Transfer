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

# splitter

```
from bs4 import BeautifulSoup
import uuid
from collections import defaultdict

def inline_to_external_css(html_path, output_html_path, css_output_path):
    # Step 1: Parse HTML content
    with open(html_path, 'r', encoding='utf-8') as file:
        soup = BeautifulSoup(file.read(), "html.parser")

    # Initialize CSS content and map to store unique styles
    css_content = ""
    style_map = defaultdict(list)  # Maps styles to classes
    css_classes = {}  # Maps elements to assigned classes

    # CSS reset for default HTML properties that could conflict with parent divs
    reset_css = """
    body, h1, h2, h3, h4, h5, h6, p, ul, ol, li, table, th, td, blockquote, hr, button, input, select, textarea, img {
        margin: 0;
        padding: 0;
        border: 0;
        outline: 0;
        font-size: 100%;
        vertical-align: baseline;
        background: transparent;
    }
    img, video, svg {
        max-width: 100%;
        height: auto;
    }
    """

    css_content += reset_css

    # Step 2: Find all elements with inline styles
    for element in soup.find_all(style=True):
        inline_style = element["style"].strip()

        # Check if this style has been used before
        if inline_style in style_map:
            # Use an existing class if the style matches
            unique_class = style_map[inline_style][0]
        else:
            # Create a new unique class for this style
            unique_class = f"class_{uuid.uuid4().hex[:8]}"
            style_map[inline_style].append(unique_class)

            # Add the CSS rule to the stylesheet content
            css_content += f".{unique_class} {{ {inline_style} }}\n"

        # Remove the inline style and apply the unique class
        del element["style"]

        # Merge with any existing classes
        existing_classes = element.get("class", [])
        element["class"] = existing_classes + [unique_class]

    # Step 3: Save the modified HTML with link to the external CSS
    # Convert the modified HTML to string
    modified_html = str(soup)

    # Add link to the new external CSS
    modified_html = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Converted Page</title>
    <link rel="stylesheet" href="./assets/styles.css">
</head>
<body>
""" + modified_html + """
</body>
</html>
"""

    # Step 4: Write the CSS to a new file with UTF-8 encoding
    with open(css_output_path, "w", encoding='utf-8') as css_file:
        css_file.write(css_content)

    # Step 5: Write the modified HTML to the output path with UTF-8 encoding
    with open(output_html_path, "w", encoding='utf-8') as html_file:
        html_file.write(modified_html)
        
    print(f"Conversion complete. HTML saved to {output_html_path}, CSS saved to {css_output_path}.")

# Usage
inline_to_external_css("./GeneratedComponent.html", "./mockup.html", "./assets/styles.css")

```
https://teams.microsoft.com/l/meetup-join/19%3ameeting_MjA3OTI4NjUtODc4OS00NGFjLWJlZTAtY2M5MmIxN2ZlMmUw%40thread.v2/0?context=%7b%22Tid%22%3a%22404b1967-6507-45ab-8a6d-7374a3f478be%22%2c%22Oid%22%3a%2269230837-0af5-4c95-8a71-df527c50a22b%22%7d

```import boto3
import base64
import json

# if no cli or credentials cannot be confirmed in bmo environment use the code below
"""bedrock = boto3.client(
    service_name='bedrock-runtime',
    region_name='us-east-1',  # Replace with your preferred region
    aws_access_key_id='YOUR_ACCESS_KEY',
    aws_secret_access_key='YOUR_SECRET_KEY'
)"""
bedrock = boto3.client(
    service_name='bedrock-runtime',
    region_name='us-east-1'
)

def encode_image(image_path):
    with open(image_path, "rb") as image_file:
        return base64.b64encode(image_file.read()).decode('utf-8')

def generate_angular_code(html, CSS, image_path):
    # Encode the image
    base64_image = encode_image(image_path)

    # Prepare the prompt
    prompt = f"""
    Given the following HTML and CSS code, and the attached Figma screen image,
    please generate Angular code with interactions that implements this design:

    HTML:
    {html}

    CSS:
    {CSS}

    Please analyze the provided Figma screen image and incorporate any additional
    styling or layout details into the Angular components. Include appropriate
    interactions based on common UI patterns visible in the image.

    Provide the complete Angular component code, including the TypeScript class
    with methods for interactions.
    """

    # Prepare the request body
    body = json.dumps({
        "prompt": prompt,
        "max_tokens_to_sample": 4000,
        "temperature": 0.5,
        "top_p": 0.9,
        "anthropic_version": "bedrock-2023-05-31",
        "image": base64_image
    })

    # selecting model
    response = bedrock.invoke_model(
        body=body,
        modelId='anthropic.claude-3-sonnet-20240229-v1:0',  # Use the appropriate model ID
        contentType='application/json',
        accept='application/json'
    )

    # parse and return the response
    response_body = json.loads(response['body'].read())
    return response_body['completion']

# basic main, can be changed to import the files or select the directory later
if __name__ == "__main__":
    # Get user input for HTML/CSS
    print("Enter your HTML code (press Enter twice to finish):")
    html_line= []
    while True:
        line = input()
        if line == "":
            break
        html_line.append(line)
    html = "\n".join(html_line)

    print("Enter your CSS code (press Enter twice to finish):")
    CSS_line = []
    while True:
        line = input()
        if line == "":
            break
        CSS_line.append(line)
    css = "\n".join(CSS_line)
    # Get user input for image path
    image_path = input("Enter the path to your Figma screen image: ")

    # Generate Angular code
    try:
        angular_code = generate_angular_code(html, css, image_path)
        print("\nGenerated Angular Code:")
        print(angular_code)
    except Exception as e:
        print(f"An error occurred: {str(e)}")
```
```You are front end developer who excels at converting Figma design to given tech stack.  You are given 3 inputs and set of instructions you have to follow.


<Instructions>

Analyze the Design image given to you. Identify all the UI elements.

Analyze the HTML and CSS file given to you.

Follow the best UI development practice to generate new html and CSS file while keeping accessibility in mind making sure all elements are keyboard accessible and can be read through screen readers.

New code should be extremely  responsive. be very careful not to change file paths or asset names.

Change the class names in CSS to the appropriate names which follows naming conventions.

All the UI element should be functioning to the extent where user can see it working.

Make sure to keep the new generated code look exactly as the design given to you.

</Instructions>

<Input>

HTML:

"""

HTML FILE

"""

CSS:

"""

CSS FILE

"""

</Input>

<Output>

The expected output is a fully functional website (HTML and CSS code) that is as similar as possible to the image provided. Little to no human intervention should be necessary to make the code similar to the image (assume that the user does not understand how to code, so make the code as accurate and extensive as necessary).

</Output>```
