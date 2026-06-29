# Dockerfile Practical — Flask App, Volume & Networking

## Example 1: Simple Flask App with Dockerfile

**Scenario / Question:** A teammate gave you a small Python Flask app. How do you package it into a Docker image so it runs the same way on any machine, without anyone installing Python or Flask manually?

### Step 1 — Project setup
```bash
mkdir docker-flask-demo && cd docker-flask-demo
```

### Step 2 — Create `app.py`
```python
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def home():
    name = os.environ.get('APP_NAME', 'World')
    return f'<h1>Hello, {name}! Running inside Docker.</h1>'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```
> `0.0.0.0` is required (not `127.0.0.1`) — otherwise the server only listens inside the container and is unreachable from outside.

### Step 3 — Create `requirements.txt`
```
flask
```

### Step 4 — Create `Dockerfile`
```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

ENV APP_NAME=Docker
EXPOSE 5000

CMD ["python", "app.py"]
```
> Dependencies are copied and installed BEFORE the rest of the code. This is intentional caching — if only `app.py` changes later, the slow `pip install` layer is reused instead of re-running.

### Step 5 — Build the image
```bash
docker build -t flask-demo .
```

### Step 6 — Run the container
```bash
docker run -d --name flask-app -p 5000:5000 flask-demo
```
> Open http://localhost:5000 → shows "Hello, Docker!"

### Step 7 — Override environment variable at runtime
```bash
docker stop flask-app && docker rm flask-app
docker run -d --name flask-app -p 5000:5000 -e APP_NAME=Professor flask-demo
```
> Same image, different behavior — no rebuild needed. Refresh browser → "Hello, Professor!"

---

## Example 2: Volume — Persist Data Across Container Restarts

**Scenario / Question:** You write some data inside a container. If the container is deleted, is that data gone forever? How do you make it survive?

### Step 1 — Create a named volume
```bash
docker volume create my-data
```

### Step 2 — Run a container with the volume mounted, write a file
```bash
docker run -d --name box1 -v my-data:/data alpine sleep infinity
docker exec box1 sh -c "echo 'saved data' > /data/notes.txt"
```

### Step 3 — Delete the container completely
```bash
docker rm -f box1
```

### Step 4 — Start a brand-new container using the SAME volume
```bash
docker run -d --name box2 -v my-data:/data alpine sleep infinity
docker exec box2 cat /data/notes.txt
```
> Output: `saved data`. `box1` is gone, but `my-data` is a separate Docker-managed volume — it outlives any single container. This is the core idea behind persistent storage in Docker.

### Cleanup
```bash
docker rm -f box2
docker volume rm my-data
```

---

## Example 3: Networking — Connect Two Containers by Name

**Scenario / Question:** You have two separate containers. How does one reach the other directly by name, the way services usually need to talk to each other?

### Step 1 — Create a custom network
```bash
docker network create my-net
```

### Step 2 — Run two containers on that network
```bash
docker run -d --name box-a --network my-net alpine sleep infinity
docker run -d --name box-b --network my-net alpine sleep infinity
```

### Step 3 — Ping one from the other, using its NAME (not an IP)
```bash
docker exec box-a ping -c 3 box-b
```
> Success — `box-a` resolves `box-b` by name. Docker provides automatic DNS for any containers sharing a custom network. (Note: this does NOT work on the default network — a custom network is required.)

### Cleanup
```bash
docker rm -f box-a box-b
docker network rm my-net
```

---

## Suggestions
- Keep Examples 2 and 3 separate when demoing — combining volumes + networking + a real app in one go (e.g. Flask + Redis) is a good *next* exercise once students are comfortable with each piece alone.
- Always pin image versions in real projects (`python:3.12-slim`, not `python:latest`) for reproducible builds.
- Never bake real secrets (DB passwords, API keys) into the Dockerfile — pass them via `-e` or a secrets manager, as shown here with `APP_NAME`.
