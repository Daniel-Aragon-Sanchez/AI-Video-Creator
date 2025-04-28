from moviepy import VideoFileClip, TextClip, CompositeVideoClip, ColorClip
from flask import Flask, request, render_template_string, send_file, jsonify
from google.oauth2.credentials import Credentials
from google.oauth2 import service_account
from googleapiclient.discovery import build
from googleapiclient.http import MediaFileUpload
from pymongo import MongoClient
import os
import json
app = Flask(__name__)
client = MongoClient('mongodb://localhost:27017/')
db = client['content_db']
references = db['references']
prompts = db['prompts']
metadata = db['metadata']
SCOPES = ['https://www.googleapis.com/auth/drive.file']
SERVICE_ACCOUNT_FILE = 'service-account.json'
def get_drive_service():
    credentials = service_account.Credentials.from_service_account_file(
        SERVICE_ACCOUNT_FILE, scopes=SCOPES)
    return build('drive', 'v3', credentials=credentials)

HTML_TEMPLATE = '''
<!DOCTYPE html>
<html>
<head>
    <title>AI Video Creator</title>
    <style>
        body { font-family: Arial; max-width: 800px; margin: 0 auto; padding: 20px; }
        .form-group { margin: 20px 0; }
        input, textarea, select { width: 100%; padding: 8px; }
        button { background: #4CAF50; color: white; padding: 10px 20px; border: none; cursor: pointer; }
        #results { margin-top: 20px; }
    </style>
</head>
<body>
    <h1>AI Video Creator</h1>
    <form method="post" enctype="multipart/form-data">
        <div class="form-group">
            <label>Reference Image ID:</label>
            <input type="text" name="ref_id" required>
        </div>
        <div class="form-group">
            <label>Prompt ID:</label>
            <input type="text" name="prompt_id" required>
        </div>
        <div class="form-group">
            <label>Platform:</label>
            <select name="platform" required>
                <option value="instagram">Instagram</option>
                <option value="tiktok">TikTok</option>
            </select>
        </div>
        <button type="submit">Generate Content</button>
    </form>
    <div id="results"></div>
</body>
</html>
'''

def create_video(ref_image_path, prompt_data, metadata_filters):
    filtered_data = metadata.find_one(metadata_filters)
    
    if prompt_data['platform'] == 'instagram':
        aspect_ratio = (1080, 1080)
        duration = 30
    else:
        aspect_ratio = (1080, 1920)
        duration = 60
    
    video = VideoFileClip(ref_image_path).resize(aspect_ratio)
    
    title = (TextClip(filtered_data['title'], fontsize=70, color='white')
             .set_position('center')
             .set_duration(3))
    
    caption = (TextClip(prompt_data['caption'], fontsize=40, color='white')
               .set_position(('center', 'bottom'))
               .set_duration(duration))
    final_video = CompositeVideoClip([
        video.set_duration(duration),
        title,
        caption
    ])
    output_path = f'output_{prompt_data["platform"]}.mp4'
    final_video.write_videofile(output_path)
    return output_path
def upload_to_drive(file_path, metadata):
    service = get_drive_service()
    file_metadata = {
        'name': os.path.basename(file_path),
        'description': json.dumps(metadata)
    }
    media = MediaFileUpload(file_path, mimetype='video/mp4')
    file = service.files().create(
        body=file_metadata,
        media_body=media,
        fields='id'
    ).execute()
    return file.get('id')

@app.route('/', methods=['GET', 'POST'])
def index():
    if request.method == 'POST':
        try:
            ref_id = request.form['ref_id']
            ref_image = references.find_one({'_id': ref_id})
            if not ref_image:
                return 'Reference image not found', 404
            
            prompt_id = request.form['prompt_id']
            prompt = prompts.find_one({'_id': prompt_id})
            if not prompt:
                return 'Prompt not found', 404
            
            metadata_filters = {
                'title': {'$exists': True},
                'name': {'$exists': True},
                'brand': {'$exists': True}
            }
            
            platform = request.form['platform']
            output_path = create_video(ref_image['path'], {
                'platform': platform,
                'caption': prompt['text']
            }, metadata_filters)
            
            drive_metadata = {
                'platform': platform,
                'prompt_id': prompt_id,
                'ref_id': ref_id,
                'metadata': metadata_filters
            }
            file_id = upload_to_drive(output_path, drive_metadata)         
            if os.path.exists(output_path):
                os.remove(output_path)
            
            return jsonify({
                'status': 'success',
                'file_id': file_id,
                'platform': platform
            })
            
        except Exception as e:
            return str(e), 500
    
    return render_template_string(HTML_TEMPLATE)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
