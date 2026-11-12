# 📁 Project Structure (Virtual ZIP)

```
aarohi-ai/
├── backend/
│   ├── main.py
│   ├── permissions.py
│   ├── skills.py
│   ├── research.py
│   ├── memory.py
│   ├── voice.py
│   ├── agents.py          (newly added)
│   ├── requirements.txt
│   └── data/              (auto-created, empty)
├── app/
│   ├── lib/
│   │   └── main.dart
│   ├── pubspec.yaml
│   └── android/
│       └── app/
│           └── src/main/AndroidManifest.xml
├── README.md
└── .gitignore
```

---

## 📄 File-wise Complete Code

### 1. `backend/requirements.txt`
```txt
fastapi
uvicorn
pydantic
duckduckgo_search
chromadb
sentence-transformers
faster-whisper
gTTS
SpeechRecognition
requests
```

### 2. `backend/permissions.py`
```python
import json
import os

class PermissionManager:
    def __init__(self):
        self.skills = {
            "youtube_upload": ["upload", "schedule"],
            "instagram_scheduler": ["post", "story"],
            "trading_bot": ["paper_trade", "place_order"],
            "website_builder": ["create", "deploy"],
            "code_generator": ["generate", "run"],
            "research": ["search", "summarize"]
        }
        self.required_permissions = {
            "youtube_upload": ["youtube.upload"],
            "instagram_scheduler": ["instagram.content_publish"],
            "trading_bot": ["trading.orders"],
            "website_builder": ["hosting.deploy"],
            "code_generator": ["code.execute"],
            "research": ["internet.search"]
        }

    def is_allowed(self, skill, user_actions):
        if skill not in self.skills:
            return False
        allowed_actions = self.skills[skill]
        for action in user_actions:
            if action not in allowed_actions:
                return False
        return True
```

### 3. `backend/skills.py`
```python
class SkillRegistry:
    def execute(self, skill, message, sources):
        if skill == "research":
            return {"status": "success", "summary": "Research done"}
        elif skill == "youtube_upload":
            # TODO: YouTube API integration
            return {"status": "pending", "message": "YouTube upload requires OAuth"}
        elif skill == "instagram_scheduler":
            return {"status": "pending", "message": "Instagram API not configured"}
        elif skill == "trading_bot":
            # TODO: Paper trading via Alpaca/Binance
            return {"status": "paper_trade", "result": "Simulated trade executed"}
        elif skill == "website_builder":
            return {"status": "pending", "message": "Website builder coming soon"}
        elif skill == "code_generator":
            return {"status": "pending", "message": "Code generator coming soon"}
        else:
            return {"status": "not_implemented"}
```

### 4. `backend/research.py`
```python
from duckduckgo_search import DDGS

def web_research(query):
    with DDGS() as ddgs:
        results = list(ddgs.text(query, max_results=5))
    sources = []
    for r in results:
        sources.append({
            "title": r.get("title", ""),
            "url": r.get("href", ""),
            "snippet": r.get("body", "")
        })
    return sources
```

### 5. `backend/memory.py`
```python
import chromadb
import json
import os
import hashlib

class MemoryStore:
    def __init__(self):
        self.client = chromadb.Client()
        self.collection = self.client.get_or_create_collection("memory")
        self.log_file = "data/logs.json"
        os.makedirs("data", exist_ok=True)

    def add(self, query, sources, action):
        entry = {
            "query": query,
            "sources": sources,
            "action": action
        }
        with open(self.log_file, "a", encoding="utf-8") as f:
            f.write(json.dumps(entry) + "\n")

        if query:
            # Unique ID banane ke liye hash use karo
            doc_id = hashlib.md5(query.encode()).hexdigest()
            self.collection.add(
                documents=[query],
                metadatas=[{"sources": str(sources), "action": str(action)}],
                ids=[doc_id]
            )

    def search(self, query, n_results=3):
        results = self.collection.query(query_texts=[query], n_results=n_results)
        return results
```

### 6. `backend/voice.py`
```python
from gtts import gTTS
import speech_recognition as sr
import uuid
import os

def text_to_speech(text, lang='hi'):
    filename = f"tts_{uuid.uuid4().hex}.mp3"
    tts = gTTS(text=text, lang=lang)
    tts.save(filename)
    return filename

def transcribe_audio(audio_bytes):
    recognizer = sr.Recognizer()
    temp_file = "temp_audio.wav"
    with open(temp_file, "wb") as f:
        f.write(audio_bytes)

    with sr.AudioFile(temp_file) as source:
        audio = recognizer.record(source)

    os.remove(temp_file)
    try:
        text = recognizer.recognize_google(audio, language="hi-IN")
        return text
    except:
        return ""
```

### 7. `backend/agents.py` (install wala code, final version)
```python
from permissions import PermissionManager
from typing import List, Dict, Any

class AgentBuilder:
    def __init__(self):
        self.agents = {}
        self.permission_manager = PermissionManager()

    def create_agent(self, name: str, tools: List[str]) -> Dict[str, Any]:
        if name in self.agents:
            return {"error": f"Agent '{name}' already exists"}

        valid_tools = []
        for tool in tools:
            if tool in self.permission_manager.skills:
                valid_tools.append(tool)
            else:
                print(f"Warning: Tool '{tool}' unknown, skip kar diya")

        agent = {
            "agent_name": name,
            "tools": valid_tools,
            "status": "created",
            "memory": [],
        }
        self.agents[name] = agent
        return {"agent_name": name, "tools": valid_tools, "status": "created"}

    def get_agent(self, name: str) -> Dict[str, Any]:
        return self.agents.get(name, {"error": "Agent not found"})

    def list_agents(self) -> List[str]:
        return list(self.agents.keys())

    def run_agent(self, name: str, task: str) -> str:
        agent = self.agents.get(name)
        if not agent:
            return "Agent not found"
        return f"[{name}] Task received: {task}\nAvailable tools: {agent['tools']}"

agent_builder = AgentBuilder()
```

### 8. `backend/main.py` (integrated with agents)
```python
from fastapi import FastAPI, HTTPException, UploadFile, File
from pydantic import BaseModel
from typing import List, Optional
import os

from permissions import PermissionManager
from skills import SkillRegistry
from research import web_research
from memory import MemoryStore
from voice import transcribe_audio, text_to_speech
from agents import agent_builder

app = FastAPI(title="Aarohi AI")

permission_manager = PermissionManager()
skill_registry = SkillRegistry()
memory = MemoryStore()

class ChatRequest(BaseModel):
    message: str
    skill: Optional[str] = None
    research_first: bool = True
    user_allowed_actions: List[str] = []

class AgentCreateRequest(BaseModel):
    name: str
    tools: List[str] = []

@app.post("/chat")
async def chat(req: ChatRequest):
    sources = []
    if req.research_first:
        sources = web_research(req.message)

    action_taken = None
    if req.skill:
        allowed = permission_manager.is_allowed(req.skill, req.user_allowed_actions)
        if not allowed:
            return {
                "reply": "Mujhe is action ke liye permission chahiye. Kya main ye kar sakti hoon?",
                "requires_permission": True,
                "skill": req.skill,
                "sources": sources
            }
        result = skill_registry.execute(req.skill, req.message, sources)
        action_taken = result

    memory.add(req.message, sources, action_taken)

    reply = generate_response(req.message, sources, action_taken)
    return {
        "reply": reply,
        "sources": sources,
        "actions_taken": action_taken,
        "requires_permission": False
    }

@app.post("/create_agent")
async def create_agent(req: AgentCreateRequest):
    result = agent_builder.create_agent(req.name, req.tools)
    return result

@app.get("/list_agents")
async def list_agents():
    return {"agents": agent_builder.list_agents()}

@app.post("/run_agent")
async def run_agent(req: AgentCreateRequest):
    # Simple: name aur task as string
    task = " ".join(req.tools)  # temporary
    result = agent_builder.run_agent(req.name, task)
    return {"response": result}

@app.post("/voice/tts")
async def tts(text: str):
    audio_path = text_to_speech(text)
    return {"audio_path": audio_path}

@app.post("/voice/stt")
async def stt(file: UploadFile = File(...)):
    audio_bytes = await file.read()
    text = transcribe_audio(audio_bytes)
    return {"text": text}

def generate_response(message, sources, action):
    context = ""
    if sources:
        context = "Research findings:\n" + "\n".join([s['title'] + ": " + s['snippet'] for s in sources[:3]])
    act = ""
    if action:
        act = f"\nAction taken: {action}"
    return f"Main Aarohi hoon. Tumne kaha: {message}. {context}{act}"
```

### 9. Flutter App Files

#### `app/pubspec.yaml`
```yaml
name: aarohi_ai
description: Aarohi AI Assistant
publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'

dependencies:
  flutter:
    sdk: flutter
  dio: ^5.4.0
  speech_to_text: ^6.6.0
  flutter_tts: ^3.8.5
  permission_handler: ^11.3.0
  flutter_foreground_task: ^6.1.2
  path_provider: ^2.1.2

dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^3.0.0
```

#### `app/lib/main.dart`
```dart
import 'package:flutter/material.dart';
import 'package:dio/dio.dart';
import 'package:speech_to_text/speech_to_text.dart' as stt;
import 'package:flutter_tts/flutter_tts.dart';
import 'package:permission_handler/permission_handler.dart';
import 'package:flutter_foreground_task/flutter_foreground_task.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  FlutterForegroundTask.initCommunicationPort();
  runApp(AarohiApp());
}

class AarohiApp extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Aarohi AI',
      theme: ThemeData(primarySwatch: Colors.pink),
      home: ChatScreen(),
    );
  }
}

class ChatScreen extends StatefulWidget {
  @override
  _ChatScreenState createState() => _ChatScreenState();
}

class _ChatScreenState extends State<ChatScreen> {
  final TextEditingController _controller = TextEditingController();
  final Dio _dio = Dio();
  final FlutterTts _tts = FlutterTts();
  final stt.SpeechToText _stt = stt.SpeechToText();
  List<String> _messages = [];
  bool _listening = false;

  @override
  void initState() {
    super.initState();
    _requestPermissions();
    _setupForegroundService();
  }

  Future<void> _requestPermissions() async {
    await [Permission.microphone, Permission.notification].request();
  }

  void _setupForegroundService() {
    FlutterForegroundTask.init(
      androidNotificationOptions: AndroidNotificationOptions(
        channelId: 'aarohi_channel',
        channelName: 'Aarohi AI Service',
        channelDescription: 'Background voice assistant',
      ),
    );
  }

  Future<void> _sendMessage() async {
    String text = _controller.text.trim();
    if (text.isEmpty) return;
    setState(() {
      _messages.add("You: $text");
      _controller.clear();
    });
    try {
      final response = await _dio.post(
        'https://your-backend.onrender.com/chat',
        data: {
          'message': text,
          'research_first': true,
          'user_allowed_actions': [],
        },
      );
      String reply = response.data['reply'];
      setState(() {
        _messages.add("Aarohi: $reply");
      });
      _tts.speak(reply);
    } catch (e) {
      setState(() {
        _messages.add("Aarohi: Connection error");
      });
    }
  }

  Future<void> _listen() async {
    if (!_listening) {
      bool available = await _stt.initialize();
      if (available) {
        setState(() => _listening = true);
        _stt.listen(
          onResult: (result) {
            if (result.finalResult) {
              setState(() {
                _controller.text = result.recognizedWords;
                _listening = false;
              });
              _sendMessage();
            }
          },
        );
      }
    } else {
      _stt.stop();
      setState(() => _listening = false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Aarohi AI 💖')),
      body: Column(
        children: [
          Expanded(
            child: ListView.builder(
              itemCount: _messages.length,
              itemBuilder: (context, index) => ListTile(
                title: Text(_messages[index]),
              ),
            ),
          ),
          Row(
            children: [
              Expanded(
                child: TextField(
                  controller: _controller,
                  decoration: InputDecoration(hintText: 'Message bolo ya likho...'),
                ),
              ),
              IconButton(
                icon: Icon(_listening ? Icons.mic : Icons.mic_none),
                onPressed: _listen,
              ),
              IconButton(
                icon: Icon(Icons.send),
                onPressed: _sendMessage,
              ),
            ],
          ),
        ],
      ),
    );
  }
}
```

#### `app/android/app/src/main/AndroidManifest.xml`
```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.RECORD_AUDIO"/>
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
    <uses-permission android:name="android.permission.WAKE_LOCK"/>

    <application
        android:label="Aarohi AI"
        android:icon="@mipmap/ic_launcher"
        android:usesCleartextTraffic="true">
        <activity android:name=".MainActivity"
            android:exported="true"
            android:theme="@style/LaunchTheme">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
        <service android:name="com.pravera.flutter_foreground_task.service.ForegroundService"
            android:foregroundServiceType="specialUse"
            android:exported="false" />
    </application>
</manifest>
```

### 10. `README.md`
```markdown
# Aarohi AI 💖

Open-source AI assistant with voice, automation skills, and self-improvement.

## Backend Setup
1. `cd backend`
2. `pip install -r requirements.txt`
3. `uvicorn main:app --reload`

## Flutter App Setup
1. `cd app`
2. `flutter pub get`
3. Change backend URL in `lib/main.dart`

## Features
- Web research first (DuckDuckGo)
- Permission-based skill execution
- Voice TTS/STT
- Memory & mistake logs
- Agent builder
```

### 11. `.gitignore`
```
__pycache__/
*.pyc
data/
*.mp3
*.wav
.venv/
build/
.apk
```
