# Docker Exercises — Questions & Answers

## Exercise 1: Deploy Nginx on Port 8080

**Scenario:** You need a quick web server running locally to test something, reachable at port 8080.

**Question:** How do you run nginx so it's reachable at http://localhost:8080?

**Answer:**
```bash
docker pull nginx
docker run -d --name my-nginx -p 8080:80 nginx
```
> `-p 8080:80` maps host port 8080 to nginx's default port 80 inside the container. Verify with `docker ps` and by opening the browser.

---

## Exercise 2: Create a Custom Docker Image

**Scenario:** The stock nginx image shows its default welcome page. You want it to show YOUR own simple HTML page instead, baked permanently into the image.

**Question:** How do you build your own image instead of using a stock one as-is?

**Answer:**
```bash
mkdir my-site && cd my-site
echo "<h1>My Custom Page</h1>" > index.html
```
```dockerfile
# Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```
```bash
docker build -t my-custom-nginx .
docker run -d -p 8081:80 my-custom-nginx
```
> `docker build -t <name> .` builds an image from the Dockerfile in the current folder (`.` = build context). `COPY` bakes your file into the image permanently — unlike a volume mount, this content travels WITH the image anywhere it's deployed.

---

## Exercise 3: Mount a Volume and Verify Data Persistence

**Scenario:** You want to confirm that data doesn't disappear when a container is removed — proving Docker volumes truly persist independently of containers.

**Question:** How do you prove that data in a volume survives container removal?

**Answer:**
```bash
docker volume create demo-data
docker run -d --name box1 -v demo-data:/data alpine sleep infinity
docker exec box1 sh -c "echo 'hello' > /data/file.txt"
docker rm -f box1
```
```bash
docker run -d --name box2 -v demo-data:/data alpine sleep infinity
docker exec box2 cat /data/file.txt
```
> Output: `hello`. `box1` was deleted entirely, but `box2` (a brand-new container using the same volume `demo-data`) still sees the file — proving the volume, not the container, is what holds the data.

---

## Exercise 4: Connect Two Containers Using a Docker Network

**Scenario:** Two containers need to talk to each other — for example, an app container and a database container — without hardcoding IP addresses, which can change.

**Question:** How do containers talk to each other by name instead of IP?

**Answer:**
```bash
docker network create app-net
docker run -d --name db --network app-net mysql:8 -e MYSQL_ROOT_PASSWORD=pass
docker run -d --name web --network app-net nginx
docker exec web ping -c 3 db
```
> Both containers join the custom bridge network `app-net`. Docker's built-in DNS lets `web` resolve `db` by container name — the `ping` succeeds without ever specifying an IP address. (The default bridge network does NOT support this; a custom network is required.)

---

## Exercise 5: Create a Docker Compose File for Nginx + Redis

**Scenario:** Your app needs two services running together — a web server and a cache — and starting them manually with two `docker run` commands each time is tedious.

**Question:** How do you define and run a two-service stack with one file?

**Answer:**
```yaml
# docker-compose.yml
version: "3.9"
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
  redis:
    image: redis:alpine
```
```bash
docker compose up -d
docker compose ps
```
> Compose reads the YAML, creates a shared network automatically, and starts both services with one command — no manual `docker network create` or separate `docker run` calls needed.

---

## Exercise 6: Check Container Logs and Troubleshoot Issues

**Scenario:** You ran a container and it doesn't seem to be working — maybe it exited immediately or isn't responding as expected.

**Question:** A container keeps exiting. How do you find out why and fix it?

**Answer:**
```bash
docker ps -a                      # confirm it's "Exited", note the exit code
docker logs <container_name>      # read the application's own error output
docker inspect <container_name>   # check env vars, mounts, command — common misconfig spots
```
> Example: a MySQL container exits immediately because `MYSQL_ROOT_PASSWORD` wasn't set. `docker logs` shows the exact error ("Database is uninitialized and password option is not specified"). Fix by adding `-e MYSQL_ROOT_PASSWORD=yourpass` and re-running.

---

## Exercise 7: Limit Container CPU and Memory

**Scenario:** Several containers run on the same machine. One container should never be allowed to consume all the CPU or memory and starve the others.

**Question:** How do you cap a container's resource usage and confirm the limit works?

**Answer:**
```bash
docker run -d --name limited-app --memory=256m --memory-swap=256m --cpus="0.5" nginx
docker stats limited-app
```
> `--memory=256m` caps RAM at 256MB (the kernel kills the container if it tries to exceed this). `--cpus="0.5"` limits it to half of one CPU core. `docker stats` shows live MEM USAGE / LIMIT and CPU % — confirming the cap is enforced, not just configured.

---

## Suggestions
- Practice these in order — each builds on Docker concepts from the previous one (image → container → volume → network → compose → debugging → limits).
- For Exercise 6, intentionally break something yourself first (wrong env var, wrong port, missing file) rather than waiting for a real bug — it's the fastest way to learn the diagnostic flow.
- Combine Exercises 4 and 5 as a bonus round: try converting your manual two-container network setup into a Compose file.
