from flask import Flask, render_template, request, jsonify
import requests
import json
import os

app = Flask(__name__)

# Store chat messages in memory (in production, use a database)
messages = []

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/api/chat', methods=['POST'])
def chat():
    try:
        data = request.get_json()
        user_message = data.get('message', '')

        if not user_message.strip():
            return jsonify({'error': 'Empty message'}), 400

        # Add user message to history
        messages.append({'role': 'user', 'content': user_message})

        # Call Gemini API
        api_key = "AIzaSyAISWb5xwoEIoKyxR9jd5VrqfwmDCt7Qcw"
        api_url = f"https://generativelanguage.googleapis.com/v1beta/models/gemini-2.0-flash:generateContent?key={api_key}"

        # Prepare chat history for Gemini API
        contents = []
        for msg in messages:
            if msg['role'] == 'user':
                contents.append({'role': 'user', 'parts': [{'text': msg['content']}]})
            else:
                contents.append({'role': 'model', 'parts': [{'text': msg['content']}]})

        payload = {'contents': contents}

        response = requests.post(
            api_url,
            headers={'Content-Type': 'application/json'},
            json=payload
        )

        if response.status_code == 200:
            result = response.json()
            if (result.get('candidates') and
                len(result['candidates']) > 0 and
                result['candidates'][0].get('content') and
                result['candidates'][0]['content'].get('parts') and
                len(result['candidates'][0]['content']['parts']) > 0):

                ai_response = result['candidates'][0]['content']['parts'][0]['text']
                messages.append({'role': 'model', 'content': ai_response})

                return jsonify({
                    'response': ai_response,
                    'success': True
                })
            else:
                return jsonify({'error': 'Invalid API response structure'}), 500
        else:
            return jsonify({'error': f'API error: {response.status_code}'}), 500

    except Exception as e:
        return jsonify({'error': str(e)}), 500

@app.route('/api/messages')
def get_messages():
    return jsonify(messages)

@app.route('/api/clear', methods=['POST'])
def clear_messages():
    global messages
    messages = []    
    return jsonify({'success': True})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000, debug=True)
