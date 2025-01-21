# Django Chat Application

This project demonstrates the development of a real-time chat application using Django and Django Channels. It includes features such as user registration and login, WebSocket-based real-time messaging, and a user-friendly chat interface.

---

## Features
- **User Authentication**: Allows users to sign up, log in, and log out.
- **Collapsible Left Menu**: Displays all registered users.
- **Real-Time Chat**: Enables users to send and receive messages in real-time using WebSockets.
- **Message Storage**: Saves chat messages in the database.
- **Retrieve Old Messages**: Displays previously exchanged messages in the chat interface.
- **Responsive UI**: User-friendly and functional interface.

---

## Setup Instructions

### Step 1: Set Up the Project

#### Install Django and Channels
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate   # Windows
pip install django channels channels-redis
```

#### Create the Project
```bash
django-admin startproject chatapp
cd chatapp
python manage.py startapp chat
```

#### Add the App to Installed Apps
In `chatapp/settings.py`, add:
INSTALLED_APPS = [
    ...
    'chat',
    'channels',
]
```

#### Set Up ASGI
Update `chatapp/asgi.py`:
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
import chat.routing

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AuthMiddlewareStack(
        URLRouter(
            chat.routing.websocket_urlpatterns
        )
    ),
})
```

#### Set Up Redis
Install Redis and configure it in `settings.py`:
CHANNEL_LAYERS = {
    "default": {
        "BACKEND": "channels_redis.core.RedisChannelLayer",
        "CONFIG": {
            "hosts": [("127.0.0.1", 6379)],
        },
    },
}
```

---

### Step 2: User Authentication
Add the following views to `chat/views.py`:
from django.shortcuts import render, redirect
from django.contrib.auth import authenticate, login, logout
from django.contrib.auth.models import User

def signup(request):
    if request.method == "POST":
        username = request.POST["username"]
        password = request.POST["password"]
        user = User.objects.create_user(username=username, password=password)
        user.save()
        return redirect("login")
    return render(request, "chat/signup.html")

def login_view(request):
    if request.method == "POST":
        username = request.POST["username"]
        password = request.POST["password"]
        user = authenticate(request, username=username, password=password)
        if user:
            login(request, user)
            return redirect("chat")
    return render(request, "chat/login.html")

def logout_view(request):
    logout(request)
    return redirect("login")
```

Create templates for `login.html` and `signup.html`.

---

### Step 3: Display Registered Users
Update the `chat_home` view in `chat/views.py`:
from django.contrib.auth.decorators import login_required

@login_required
def chat_home(request):
    users = User.objects.exclude(id=request.user.id)
    return render(request, "chat/home.html", {"users": users})
```

Create `home.html` to list registered users.

---

### Step 4: Enable WebSocket Chat

#### Set Up Routing
Add `chat/routing.py`:
from django.urls import path
from . import consumers

websocket_urlpatterns = [
    path("ws/chat/<str:room_name>/", consumers.ChatConsumer.as_asgi()),
]
```

#### Create a WebSocket Consumer
Add `chat/consumers.py`:
import json
from channels.generic.websocket import AsyncWebsocketConsumer

class ChatConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_name = self.scope["url_route"]["kwargs"]["room_name"]
        self.room_group_name = f"chat_{self.room_name}"

        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name,
        )
        await self.accept()

    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name,
        )

    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json["message"]

        await self.channel_layer.group_send(
            self.room_group_name,
            {
                "type": "chat_message",
                "message": message,
            },
        )

    async def chat_message(self, event):
        message = event["message"]

        await self.send(text_data=json.dumps({"message": message}))
```

---

### Step 5: Store Messages

Add a `Message` model in `chat/models.py`:
from django.db import models
from django.contrib.auth.models import User

class Message(models.Model):
    sender = models.ForeignKey(User, on_delete=models.CASCADE, related_name="sent_messages")
    receiver = models.ForeignKey(User, on_delete=models.CASCADE, related_name="received_messages")
    message = models.TextField()
    timestamp = models.DateTimeField(auto_now_add=True)
```

Save messages in `ChatConsumer`'s `receive` method.

---

### Step 6: Test the Application

Run the server:
python manage.py runserver
```

Access the app at [http://127.0.0.1:8000](http://127.0.0.1:8000).

---

## Technologies Used
- **Backend:** Django, Django Channels
- **WebSocket:** Real-time messaging
- **Database:** SQLite
- **Redis:** Message broker for WebSockets





