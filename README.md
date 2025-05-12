```python
# manage.py
#!/usr/bin/env python
import os
import sys

def main():
    """Django's command-line utility for administrative tasks."""
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'pla_sim.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Make sure it's installed and "
            "available on your PYTHONPATH environment variable."
        ) from exc
    execute_from_command_line(sys.argv)

if __name__ == '__main__':
    main()
```

```python
# pla_sim/asgi.py
import os
import django
from channels.routing import get_default_application
from django.core.asgi import get_asgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'pla_sim.settings')
django.setup()

from channels.routing import ProtocolTypeRouter, URLRouter
import pla_sim.routing

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": URLRouter(pla_sim.routing.websocket_urlpatterns),
})
```

```python
# pla_sim/wsgi.py
# NEW FILE to fix the WSGI issue. This ensures Django provides an 'application' object
# that can be used by any WSGI server without the "TypeError" about arguments.

import os
from django.core.wsgi import get_wsgi_application

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'pla_sim.settings')

application = get_wsgi_application()
```

```python
# pla_sim/consumers.py
import json
from channels.generic.websocket import AsyncWebsocketConsumer
from graphql.execution.executors.asyncio import AsyncioExecutor
from strawberry.django.views import AsyncGraphQLView
from strawberry.subscriptions import SUBSCRIPTION_PROTOCOLS

# For demonstration, we adapt a Strawberry subscription approach:
# You can also do with Graphene Subscriptions, but let's keep it minimal.

class GraphQLSubscriptionConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        # Accept the connection
        await self.accept()
        # Here you'd integrate with your subscription manager
        # For example, with Strawberry's subscription manager
        await self.send(json.dumps({"message": "GraphQL Subscriptions connected"}))

    async def receive(self, text_data=None, bytes_data=None):
        # Normally you'd parse the incoming GraphQL subscription request,
        # handle start/stop, etc. This is a stub for demonstration.
        if text_data:
            data = json.loads(text_data)
            # Example echo or subscription ack
            if data.get("type") == "subscribe":
                # A naive subscription simulation
                await self.send(json.dumps({"type": "next", "payload": {"data": {"echo": "Subscription Data!"}}}))
            elif data.get("type") == "stop":
                await self.send(json.dumps({"type": "complete"}))

    async def disconnect(self, close_code):
        pass
```

```python
# pla_sim/routing.py
from django.urls import path
from . import consumers

websocket_urlpatterns = [
    path('ws/subscriptions/', consumers.GraphQLSubscriptionConsumer.as_asgi()),
]
```

```python
# pla_sim/schema.py
import strawberry
from typing import AsyncGenerator

@strawberry.type
class SimulationData:
    status: str
    detail: str

@strawberry.type
class Query:
    hello: str = "Welcome to the PLA SSR GraphQL system."

@strawberry.type
class Subscription:
    @strawberry.subscription
    async def watch_simulation(self, interval: float = 1.0) -> AsyncGenerator[SimulationData, None]:
        # A fake subscription that yields data periodically
        import asyncio
        import time
        start = time.time()
        while time.time() - start < 10:  # 10 seconds for demonstration
            await asyncio.sleep(interval)
            yield SimulationData(status="running", detail=f"Time: {time.time() - start:.1f}s")

schema = strawberry.Schema(Query, subscription=Subscription)
```

```python
# pla_sim/settings.py
import os
from pathlib import Path

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = 'replace-with-your-secret-key'
DEBUG = True
ALLOWED_HOSTS = ['*']

INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'graphene_django',
    'channels',
    'pla_sim',
]

MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]

ROOT_URLCONF = 'pla_sim.urls'

TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [BASE_DIR / 'build'],  # for SSR usage
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]

ASGI_APPLICATION = 'pla_sim.asgi.application'

# IMPORTANT FIX:
# Instead of pointing directly to the function, point to our new wsgi.py's `application` object:
WSGI_APPLICATION = 'pla_sim.wsgi.application'

# Example: Here, we adapt the DB to connect to PostgreSQL on port 5689 as requested (on Windows).
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': 'your_db_name',
        'USER': 'postgres',
        'PASSWORD': 'your_db_password',
        'HOST': '127.0.0.1',
        'PORT': '5689',
    }
}

GRAPHENE = {
    'SCHEMA': 'pla_sim.schema.schema',
    'MIDDLEWARE': [],
}

CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels.layers.InMemoryChannelLayer',
    },
}

STATIC_URL = '/static/'
STATIC_ROOT = BASE_DIR / 'staticfiles'
STATICFILES_DIRS = [
    BASE_DIR / 'build',  # The React build output
]

DEFAULT_AUTO_FIELD = 'django.db.models.BigAutoField'
```

```python
# pla_sim/urls.py
from django.urls import path, include
from django.views.generic import TemplateView
from django.conf import settings
from django.conf.urls.static import static
from . import views

urlpatterns = [
    # GraphQL
    path("graphql", views.GraphQLViewCustom.as_view(), name="graphql"),
    # SSR index
    path('', views.serve_react_app, name='react-app'),
]

# Serve static if needed in dev:
if settings.DEBUG:
    urlpatterns += static(settings.STATIC_URL, document_root=settings.STATIC_ROOT)
```

```python
# pla_sim/views.py
import os
from django.http import HttpResponse
from django.shortcuts import render
from django.views.decorators.csrf import csrf_exempt
from graphene_django.views import GraphQLView
from strawberry.django.views import GraphQLView as StrawberryView
from .schema import schema

class GraphQLViewCustom(StrawberryView):
    # This is enough to serve the schema at /graphql
    schema = schema

@csrf_exempt
def serve_react_app(request):
    """
    Server-Side Render placeholder:
    In a real SSR scenario, you'd run a Node SSR build, capture the HTML, 
    and send it here. For demonstration, we just deliver the built index.html.
    
    If you want to integrate a Node-based SSR approach:
    - Build your React app with SSR (Next.js, or a custom SSR setup).
    - Possibly run a Node server to generate HTML on the fly.
    - Then fetch that HTML here and return it.
    
    For now, we serve the static build's index.html (pure client-side).
    """
    index_path = os.path.join(os.path.dirname(__file__), '..', 'build', 'index.html')
    if os.path.exists(index_path):
        with open(index_path, 'r', encoding='utf-8') as f:
            html_content = f.read()
        return HttpResponse(html_content)
    return HttpResponse("<h1>No React build found. Please run npm run build.</h1>")
```

```javascript
// src/App.js
import React, { useState } from 'react';
import * as d3 from 'd3';
import './App.css';

function App() {
  const [message, setMessage] = useState('');
  const [minutes, setMinutes] = useState(1);

  const runWargame = async () => {
    try {
      const res = await fetch('/api/run_wargame', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ minutes, dt: 1.0, capture_rate: 5 })
      });
      const data = await res.json();
      setMessage(JSON.stringify(data));
    } catch (err) {
      setMessage('Error running wargame simulation');
    }
  };

  const runCAD = async () => {
    try {
      const res = await fetch('/api/run_cad', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ param1: 3.14, param2: 2.72 })
      });
      const data = await res.json();
      setMessage(JSON.stringify(data));
    } catch (err) {
      setMessage('Error running CAD simulation');
    }
  };

  // Example d3 usage (log version for demonstration)
  console.log('Using D3 version:', d3.version);

  return (
    <div style={{ padding: '1em', fontFamily: 'sans-serif' }}>
      <h1>PLA Advanced Simulation UI</h1>
      <div style={{ marginBottom: '1em' }}>
        <label>Minutes to run the wargame: </label>
        <input
          type="number"
          value={minutes}
          onChange={e => setMinutes(parseInt(e.target.value, 10))}
        />
        <button onClick={runWargame}>Run Wargame</button>
        <button onClick={runCAD}>Run CAD</button>
      </div>
      <div>
        <h2>Output</h2>
        <pre style={{ background: '#eee', padding: '1em' }}>{message}</pre>
      </div>
    </div>
  );
}

export default App;
```

```javascript
// src/index.js
import React from 'react';
import ReactDOM from 'react-dom/client';
import App from './App';

const rootElement = document.getElementById('root');
const root = ReactDOM.createRoot(rootElement);
root.render(<App />);
```

```python
# (Optional) pla_sim/db_wrapper.py
# Example minimal "heavy machine learning" wrapper for advanced UI suggestions from a hypothetical Postgres DB on port 5689
# This is a simple placeholder to demonstrate advanced logic could be inserted here.
# In real usage, you'd integrate your ML pipeline (e.g. scikit-learn, TensorFlow, PyTorch, etc.).

import psycopg2
import random

def get_beautiful_things():
    """
    Pretend we query the DB for an ML model or some data, then do a random pick
    for demonstration. In practice, you'd do actual ML logic.
    """
    try:
        connection = psycopg2.connect(
            dbname='your_db_name',
            user='postgres',
            password='your_db_password',
            host='127.0.0.1',
            port='5689',
        )
        cursor = connection.cursor()
        # Hypothetical query
        cursor.execute("SELECT some_data FROM your_table;")
        rows = cursor.fetchall()
        # Fancy ML pipeline or advanced logic here...
        # We'll just pick a random row for demonstration:
        return random.choice(rows) if rows else "No data found"
    except Exception as e:
        return f"Error connecting to DB or processing ML: {str(e)}"
    finally:
        if 'connection' in locals() and connection:
            connection.close()
```

---

**Explanation of Key Fix (WSGI Error):**

* In your original settings, `WSGI_APPLICATION = 'django.core.wsgi.get_wsgi_application'` caused Django’s internal WSGI server to call `get_wsgi_application(environ, start_response)`, which is incorrect because `get_wsgi_application()` is itself just a function returning an `application` object.
* By adding `pla_sim/wsgi.py` and setting `WSGI_APPLICATION = 'pla_sim.wsgi.application'`, Django now knows how to load the actual `application` object for WSGI without passing unwanted arguments, thus eliminating the `TypeError: get_wsgi_application() takes 0 positional arguments but 2 were given`.

Everything else is unchanged, except for the new `wsgi.py`, the updated `WSGI_APPLICATION` in `settings.py`, and (optionally) the demonstration `db_wrapper.py` for the Postgres port 5689 wrapper plus potential machine learning. This should address the error and provide a foundation for the advanced SSR + ML integration you requested.
# Bo_PLA_Postgres_Django_SSR_D3js_served_by_d3graph_flask_ssr_react_build_scripts

