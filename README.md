# Docker Hands-on Lab — Command Reference with Explanations

Copy-paste backup for live demo. Run top to bottom. Each command has a short explanation of what it does and why.

## 0. Installation

**Linux**
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER && newgrp docker   # run docker without sudo
sudo systemctl enable --now docker               # start daemon, enable on boot
```
> Installs Docker Engine, adds your user to the docker group so `sudo` isn't needed every time, then starts and enables the background service.

**Windows** — Enable WSL2, then install Docker Desktop:
```powershell
wsl --install
```
> WSL2 gives Windows a real Linux kernel underneath, which Docker Desktop uses as its engine. Install Docker Desktop from docker.com afterward, keeping "Use WSL2" checked.

**macOS**
```bash
brew install --cask docker
```
> Installs Docker Desktop via Homebrew. Launch it from Applications once installed.

## 1. Verify Installation
```bash
docker --version
```
> Confirms the CLI is installed and shows its version.
```bash
docker info
```
> Shows both client AND daemon (server) status — if the Server section errors out, the Docker background service isn't running.
```bash
docker run hello-world
```
> Best single sanity check: Docker pulls the image from Docker Hub (not local yet), creates a container, runs it, and it prints a message and exits — exercising the full pull → create → run flow in one command.

## 2. Images & Containers Basics
```bash
docker pull nginx
```
> Downloads the nginx image only — does not start a container.
```bash
docker images
```
> Lists every image stored locally, with size.
```bash
docker run -d --name my-nginx -p 8080:80 nginx
```
> Starts a container: `-d` runs it in the background, `--name` gives it a readable name, `-p 8080:80` maps host port 8080 → container port 80 (format HOST:CONTAINER).
```bash
docker ps
```
> Lists only RUNNING containers.
```bash
docker ps -a
```
> Lists ALL containers, including stopped/exited ones — useful for spotting crashes.
```bash
docker logs my-nginx
```
> Shows the container's stdout/stderr — exactly what the process printed.
```bash
docker exec -it my-nginx bash
```
> Opens an interactive shell INSIDE the already-running container without restarting it. `exit` leaves the shell; container keeps running.
```bash
docker stop my-nginx
```
> Gracefully stops the container's main process. Container still exists on disk (check with `docker ps -a`).
```bash
docker start my-nginx
```
> Restarts a previously stopped container, keeping its old config and filesystem state.
```bash
docker restart my-nginx
```
> Stop + start in one step — handy after changing a mounted config file.
```bash
docker rm -f my-nginx
```
> Permanently deletes the container (force-removes even if running). The underlying image is untouched.
```bash
docker rmi nginx
```
> Deletes the image from local storage. Fails if any container still depends on it.

## 3. Custom Image (Static Site)
```bash
mkdir docker-demo-static && cd docker-demo-static
mkdir site
echo "<h1>Hello from Docker</h1>" > site/index.html
```
> Sets up a project folder with a simple HTML file to serve.

**Dockerfile**
```dockerfile
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY site/ /usr/share/nginx/html/
EXPOSE 80
```
> `FROM` sets the base image (lightweight Alpine nginx). `RUN` clears nginx's default placeholder page at build time. `COPY` puts our site into nginx's web root. `EXPOSE` documents the listening port (doesn't publish it — `-p` does that at runtime).

```bash
docker build -t my-static-site .
```
> Builds an image from the Dockerfile in the current directory (`.` = build context) and tags it `my-static-site`.
```bash
docker run -d --name static-demo -p 8081:80 my-static-site
```
> Runs a container from our custom image, publishing it on port 8081.
```bash
# edit site/index.html, then:
docker build -t my-static-site .
```
> Rebuilding reuses cached layers for FROM/RUN since they didn't change — only the COPY layer (and after) re-executes. This is Docker's layer caching in action.
```bash
docker stop static-demo && docker rm static-demo
docker run -d --name static-demo -p 8081:80 my-static-site
```
> Replaces the old container with a fresh one based on the updated image, so the browser shows the new content.

## 4. Environment Variables
```bash
docker run -d --name flask-app -p 5000:5000 -e APP_NAME=Professor flask-demo
```
> `-e KEY=VALUE` overrides/sets an environment variable at runtime — the same image can behave differently in dev/staging/prod just by changing this flag, no rebuild needed.

## 5. Volumes & Bind Mounts (Data Persistence)
```bash
docker volume create demo-data
```
> Creates a named volume managed by Docker, independent of any container's lifecycle.
```bash
docker run -d --name db1 -v demo-data:/var/lib/data mysql:8
```
> Mounts the volume into the container at `/var/lib/data` — anything written there is stored in the volume, not the container's writable layer.
```bash
docker exec -it db1 sh -c "echo 'test' > /var/lib/data/file.txt"
```
> Writes a test file into the mounted path, inside the volume.
```bash
docker rm -f db1
```
> Deletes the container entirely — but the volume (and the file inside it) is untouched, since it lives independently.
```bash
docker run -d --name db2 -v demo-data:/var/lib/data mysql:8
docker exec -it db2 cat /var/lib/data/file.txt   # data still exists
```
> A brand-new container, same volume — the file from the deleted container is still there. This proves data persistence beyond a container's life.
```bash
docker volume ls
```
> Lists all volumes Docker currently manages on the host.

Bind mount (live code reload, no rebuild):
```bash
docker run -d --name flask-app -p 5000:5000 -v $(pwd):/app flask-demo
```
> Mounts the current HOST folder directly into the container, overlaying the copied code. Edit files on the host, restart the container, and changes appear instantly — no image rebuild required. Standard pattern for local development.

## 6. Docker Networking
```bash
# Custom bridge network
docker network create app-net
```
> Creates an isolated network. Containers attached to it get automatic DNS — they can reach each other by container NAME instead of IP.
```bash
docker run -d --name db --network app-net mysql:8 -e MYSQL_ROOT_PASSWORD=pass
docker run -d --name web --network app-net -p 8080:80 nginx
```
> Both containers join `app-net`.
```bash
docker exec -it web ping db          # name resolves automatically
```
> Proves name-based resolution works — `web` reaches `db` by name, no IP needed.
```bash
docker network ls
```
> Lists all networks Docker manages on this host.
```bash
docker network inspect app-net
```
> Shows detailed JSON: connected containers, subnet, gateway, etc.

```bash
# Host network
docker run -d --network host --name fast-nginx nginx
```
> Container shares the host's network stack directly — nginx binds straight to host port 80, no `-p` flag possible or needed. Loses isolation; gains raw performance.

## 7. Container Logs & Monitoring
```bash
docker logs -f my-nginx
```
> `-f` follows logs live, streaming new lines as they're written — like `tail -f`.
```bash
docker stats
```
> Live CPU/memory/network usage for all running containers, updated continuously.
```bash
docker inspect my-nginx
```
> Full JSON metadata: IP address, mounts, env vars, restart policy, exit code, everything.

## 8. Resource Limits (CPU / Memory)
```bash
docker run -d --name limited-app --memory=256m --memory-swap=256m --cpus="0.5" nginx
```
> `--memory=256m` caps RAM at 256MB (kernel OOM-kills the container if exceeded). `--memory-swap` equal to `--memory` disables extra swap. `--cpus="0.5"` limits it to half of one CPU core, even if more are free.
```bash
docker stats limited-app
```
> Shows MEM USAGE / LIMIT and CPU % live — confirms the caps are actually enforced.
```bash
docker update --memory=512m --memory-swap=512m limited-app
```
> Adjusts limits on a RUNNING container without recreating it — useful for live tuning.

## 9. Health Checks
**Dockerfile snippet**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost/ || exit 1
```
> Checks every 30s, must respond within 3s, must fail 3 times in a row to be marked unhealthy. `wget --spider` just confirms the server responds without downloading the page.
```bash
docker ps                     # STATUS shows (healthy)/(unhealthy)
```
> Health status appears directly in the STATUS column — no extra command needed.
```bash
docker inspect --format='{{json .State.Health}}' static-demo
```
> Shows the recent history of health check results — useful for diagnosing intermittent failures.

## 10. Database & CMS Deployments

**MySQL**
```bash
docker run -d --name mysql-demo \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=demodb \
  -p 3306:3306 -v mysql-data:/var/lib/mysql mysql:8
```
> `MYSQL_ROOT_PASSWORD` is required by the official image. `MYSQL_DATABASE` auto-creates an empty DB on first boot. The volume persists actual data files outside the container.
```bash
docker exec -it mysql-demo mysql -uroot -prootpass -e "SHOW DATABASES;"
```
> Runs a SQL command directly inside the container to confirm it's working.

**Redis**
```bash
docker run -d --name redis-demo -p 6379:6379 -v redis-data:/data redis:alpine
docker exec -it redis-demo redis-cli
SET greeting "Hello Docker"
GET greeting
```
> Starts Redis with persistent storage, then opens its CLI to set/get a key directly — confirms the cache is live and responding.

**WordPress + MySQL (manual)**
```bash
docker network create wp-net
docker run -d --name wp-mysql --network wp-net \
  -e MYSQL_ROOT_PASSWORD=rootpass -e MYSQL_DATABASE=wordpress \
  -v wp-mysql-data:/var/lib/mysql mysql:8
docker run -d --name wp-site --network wp-net \
  -e WORDPRESS_DB_HOST=wp-mysql -e WORDPRESS_DB_NAME=wordpress \
  -e WORDPRESS_DB_PASSWORD=rootpass -p 8083:80 wordpress
```
> Both containers share `wp-net`, so WordPress reaches MySQL simply via `WORDPRESS_DB_HOST=wp-mysql` (the container name) — same DNS principle as Section 6.

## 11. Docker Compose

**Nginx + Redis** (`docker-compose.yml`)
```yaml
version: "3.9"
services:
  web:
    image: nginx:alpine
    ports:
      - "8080:80"
  redis:
    image: redis:alpine
```
> One file defines both services. Compose auto-creates a shared network so `web` could reach `redis` by name — no manual network setup needed.
```bash
docker compose up -d
```
> Builds (if needed) and starts every service in the background.
```bash
docker compose ps
```
> Lists only containers belonging to this Compose project.
```bash
docker compose logs -f
```
> Follows logs from all services interleaved (add a service name to filter).
```bash
docker compose down -v
```
> Stops/removes containers + network; `-v` also deletes named volumes — full clean slate.

**WordPress + MySQL** (`docker-compose.yml`)
```yaml
version: "3.9"
services:
  wordpress:
    image: wordpress
    ports:
      - "8085:80"
    environment:
      - WORDPRESS_DB_HOST=db
      - WORDPRESS_DB_NAME=wordpress
      - WORDPRESS_DB_PASSWORD=rootpass
    depends_on:
      - db
    volumes:
      - wp-content:/var/www/html
  db:
    image: mysql:8
    environment:
      - MYSQL_ROOT_PASSWORD=rootpass
      - MYSQL_DATABASE=wordpress
    volumes:
      - wp-db-data:/var/lib/mysql
volumes:
  wp-content:
  wp-db-data:
```
> Same two-container WordPress setup as Section 10, but declarative — `depends_on` controls start order, named volumes persist site files and DB data. One `docker compose up -d` replaces two manual `docker run` commands — the clearest argument for using Compose.

## 12. Registry
```bash
docker login
```
> Authenticates with Docker Hub.
```bash
docker tag my-static-site myusername/my-static-site:v1
```
> Renames/tags the local image to match `username/repo:version` format required for pushing.
```bash
docker push myusername/my-static-site:v1
```
> Uploads the image, making it pullable from anywhere.

```bash
# private registry
docker run -d -p 5000:5000 --name local-registry registry:2
```
> Runs your OWN registry locally on port 5000 using Docker's official registry image — useful for private/internal images.
```bash
docker tag my-static-site localhost:5000/my-static-site
docker push localhost:5000/my-static-site
```
> Same tag-and-push flow, pointed at the local private registry instead of Docker Hub.

## 13. Backup & Restore
```bash
# backup volume
docker run --rm -v mysql-data:/data -v $(pwd):/backup alpine \
  tar czf /backup/mysql-data-backup.tar.gz -C /data .
```
> Spins up a disposable alpine container (`--rm` auto-deletes it after) that mounts BOTH the volume to back up and the current host folder, then archives the volume's contents into a `.tar.gz` on the host.
```bash
# restore volume
docker volume create mysql-data-restored
docker run --rm -v mysql-data-restored:/data -v $(pwd):/backup alpine \
  tar xzf /backup/mysql-data-backup.tar.gz -C /data
```
> Creates a fresh volume and extracts the backup into it — any container using this volume sees the restored data.
```bash
# save/load image
docker save -o my-static-site.tar my-static-site
docker load -i my-static-site.tar
```
> `save` packages an image (all layers) into one portable `.tar` file — for transferring images without a registry (USB, internal share). `load` imports it back.

## 14. Troubleshooting
```bash
docker ps -a
```
> Step 1: is the container even running, or did it exit?
```bash
docker logs <container>
```
> Step 2: what did the application itself say before failing?
```bash
docker inspect <container>
```
> Step 3: check exit code, mounts, env vars, network settings in full detail.
```bash
docker exec -it <container> sh
```
> Step 4: get inside a RUNNING container to poke around directly.
```bash
docker stats
```
> Step 5: is it a resource limit (CPU/memory) issue, not a code issue?
```bash
docker events
```
> Step 6: watch real-time daemon events while reproducing the issue.
```bash
docker system prune
```
> Cleans up unused containers, images, and networks to free disk space (run after troubleshooting, not during).

---

## Exercises

1. **Deploy Nginx on port 8080** — `docker run -d -p 8080:80 nginx`
2. **Create a custom image** — build from `nginx:alpine` + your own HTML.
3. **Mount a volume, verify persistence** — write data, `rm` container, attach new container to same volume, confirm data survives.
4. **Connect two containers via network** — custom bridge network, ping by container name.
5. **Compose file for Nginx + Redis** — see Section 11.
6. **Check logs & troubleshoot** — break a container intentionally, diagnose via `docker logs`.
7. **Limit CPU/Memory** — `--memory=256m --cpus="0.5"`, verify with `docker stats`.
