# Nginx Container — Mount Custom index.html via Volume

**Scenario:** You want nginx to serve your own webpage instead of its default welcome page — but without writing a Dockerfile or building a new image, just for a quick local test.

**Question:** How do you make the stock nginx image show YOUR content, using only a volume mount?

This example serves a custom webpage in nginx WITHOUT building a custom image — just by mounting a local HTML file straight into the container using a bind mount.

### Step 1 — Create a project folder and your HTML file
```bash
mkdir nginx-volume-demo && cd nginx-volume-demo
```

Create `index.html`:
```html
<!DOCTYPE html>
<html>
<head><title>Nginx Volume Demo</title></head>
<body style="font-family:sans-serif; text-align:center; margin-top:80px;">
  <h1>Hello from a mounted volume!</h1>
  <p>This file lives on the host, not inside the image.</p>
</body>
</html>
```

### Step 2 — Run nginx with the file bind-mounted in
```bash
docker run -d --name nginx-vol -p 8080:80 \
  -v $(pwd)/index.html:/usr/share/nginx/html/index.html \
  nginx:alpine
```
> `-v $(pwd)/index.html:/usr/share/nginx/html/index.html` maps the single host file directly to nginx's default page location. No Dockerfile, no build step — the stock `nginx:alpine` image now serves YOUR content.

### Step 3 — Verify
```bash
docker ps
```
Open **http://localhost:8080** → shows "Hello from a mounted volume!"

### Step 4 — Prove it's live (no rebuild needed)
```bash
echo '<h1>Updated content!</h1>' > index.html
```
> Refresh the browser — the change appears immediately. Since it's a bind mount, nginx is reading directly from the host file in real time; nothing needs to be rebuilt or restarted.

### Step 5 — Mount an entire folder instead (multi-file sites)
```bash
mkdir site
mv index.html site/
echo "<h1>About page</h1>" > site/about.html

docker rm -f nginx-vol
docker run -d --name nginx-vol -p 8080:80 \
  -v $(pwd)/site:/usr/share/nginx/html \
  nginx:alpine
```
> Mounting the whole folder lets you add/edit any number of files (about.html, images, CSS) without touching the container at all.

### Cleanup
```bash
docker rm -f nginx-vol
```

---

## Suggestions
- This bind-mount approach is great for **quick local previews**, but for anything shipped to production, bake the HTML into a custom image instead (see the Dockerfile Practical README) — bind mounts depend on the host filesystem being present, which doesn't travel with the image.
- Mounting a single file (Step 2) vs. a whole folder (Step 5): use the folder approach as soon as you have more than one file, since adding new files later doesn't require changing the `docker run` command.
- Combine this with a **named volume** instead of a bind mount if you want nginx's content to persist independently of any host folder (e.g. content uploaded by another container) — bind mounts tie you to a specific host path, named volumes don't.
