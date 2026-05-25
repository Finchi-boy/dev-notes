# 🛠️ Dev Cheatsheet

> Personal reference for commands and setups I always have to look up.
> Add new sections freely — structure: `## Topic` → `### Subtopic` → code block + note.

---

## Table of Contents

- [Poetry](#poetry)
- [Git & GitHub](#git--github)
- [Docker](#docker)
- [JWT (JSON Web Token)](#jwt-json-web-token)

---

## Poetry

### Init Poetry for dependency management only (no new project scaffold)

Use this when you already have a project folder and just want Poetry to manage packages — without it generating `pyproject.toml` from scratch interactively.

```bash
poetry init
```

Walks you through creating `pyproject.toml` step by step. Skip fields you don't need by pressing Enter.

If you want zero prompts:

```bash
poetry init --no-interaction
```

### Install dependencies into the project (without creating a venv yourself)

```bash
poetry install
```

### Add a package

```bash
poetry add requests
poetry add pytest --group dev      # dev dependency
```

### Remove a package

```bash
poetry remove requests
```

### Run a command inside the Poetry-managed venv

```bash
poetry run python script.py
poetry run pytest
```

### Activate the venv shell

```bash
poetry shell
```

### Tell Poetry to use a specific Python version

```bash
poetry env use python3.11
```

### Use Poetry WITHOUT creating a virtualenv (e.g. inside Docker or CI)

```bash
poetry config virtualenvs.create false
```

### Show current venv info

```bash
poetry env info
```

### List installed packages

```bash
poetry show
poetry show --tree    # with dependency tree
```

---

## Git & GitHub

### Start a repo

```bash
git init
git remote add origin https://github.com/user/repo.git
```

### Everyday flow

```bash
git status
git add .
git commit -m "message"
git push origin main
```

### Undo last commit (keep changes staged)

```bash
git reset --soft HEAD~1
```

### Undo last commit (keep changes unstaged)

```bash
git reset HEAD~1
```

### Discard all local changes

```bash
git checkout -- .
```

### Branches

```bash
git checkout -b feature/my-feature    # create + switch
git checkout main                      # switch
git branch -d feature/my-feature      # delete local
git push origin --delete feature/my-feature  # delete remote
```

### Rebase instead of merge (clean history)

```bash
git fetch origin
git rebase origin/main
```

### Stash work in progress

```bash
git stash
git stash pop        # restore latest stash
git stash list       # see all stashes
```

### See what changed

```bash
git log --oneline --graph --all    # visual branch history
git diff HEAD~1                    # diff vs previous commit
git show abc1234                   # inspect a specific commit
```

### Rename current branch to main

```bash
git branch -M main
```

### Clone a single branch

```bash
git clone --branch develop --single-branch https://github.com/user/repo.git
```

### Tag a release

```bash
git tag v1.0.0
git push origin v1.0.0
```

---

## Docker

### Build an image

```bash
docker build -t my-app .
docker build -t my-app:1.0 -f Dockerfile.prod .   # custom tag + Dockerfile
```

### Run a container

```bash
docker run my-app
docker run -it my-app bash            # interactive shell
docker run -p 8080:80 my-app          # map host:container port
docker run -d my-app                  # detached (background)
docker run --rm my-app                # auto-remove after exit
docker run -v $(pwd):/app my-app      # mount current dir to /app
```

### List containers and images

```bash
docker ps                  # running containers
docker ps -a               # all containers (including stopped)
docker images              # all images
```

### Stop / remove

```bash
docker stop <container_id>
docker rm <container_id>
docker rmi <image_id>
docker rm $(docker ps -aq)            # remove ALL stopped containers
docker rmi $(docker images -q)        # remove ALL images
```

### Shell into a running container

```bash
docker exec -it <container_id> bash
docker exec -it <container_id> sh     # if bash isn't available
```

### Docker Compose

```bash
docker compose up              # start all services
docker compose up -d           # detached
docker compose up --build      # rebuild images before starting
docker compose down            # stop and remove containers
docker compose down -v         # also remove volumes
docker compose logs -f         # follow logs
docker compose exec web bash   # shell into a service
```

### Inspect / debug

```bash
docker logs <container_id>
docker logs -f <container_id>         # follow live
docker inspect <container_id>         # full JSON config
docker stats                          # live resource usage
```

### Clean up everything unused

```bash
docker system prune                   # stopped containers, dangling images, unused networks
docker system prune -a                # also removes unused images
docker volume prune                   # unused volumes
```

---

## JWT (JSON Web Token)

### Structure

A JWT token consists of three Base64-encoded parts separated by dots:

```
HEADER.PAYLOAD.SIGNATURE
```

Decode any token at https://jwt.io

**Header** — algorithm used to sign the token:
```json
{ "alg": "HS256", "typ": "JWT" }
```

**Payload (Claims)** — data stored in the token. Publicly readable — never store sensitive data like passwords here:
```json
{
  "sub": "maciej@example.com",
  "iat": 1716000000,
  "exp": 1716086400
}
```

**Signature** — guarantees the token wasn't tampered with:
```
HMACSHA256(base64(header) + "." + base64(payload), secret)
```

---

### Auth flow

```
POST /api/auth/login { email, password }
  → server verifies credentials → generates JWT
  → client stores token

GET /api/tasks
  Authorization: Bearer eyJhbGc...
  → server verifies signature → extracts email → handles request
```

---

### JJWT 0.12.x (Java)

Dependencies (`build.gradle.kts`):
```kotlin
implementation("io.jsonwebtoken:jjwt-api:0.12.6")
runtimeOnly("io.jsonwebtoken:jjwt-impl:0.12.6")
runtimeOnly("io.jsonwebtoken:jjwt-jackson:0.12.6")
```

Build a signing key from a string secret:
```java
SecretKey key = Keys.hmacShaKeyFor(secret.getBytes());
```

Generate a token:
```java
String token = Jwts.builder()
    .subject("maciej@example.com")
    .issuedAt(new Date())
    .expiration(new Date(System.currentTimeMillis() + 86400000)) // 24h
    .signWith(key)
    .compact();
```

Parse and verify a token:
```java
Claims claims = Jwts.parser()
    .verifyWith(key)
    .build()
    .parseSignedClaims(token)
    .getPayload();

String email = claims.getSubject();
```

> Exceptions thrown on invalid tokens: `ExpiredJwtException`, `MalformedJwtException`, `SignatureException`.

Store the secret as an environment variable, never hardcode it:
```yaml
# application.yml
jwt:
  secret: ${JWT_SECRET:local-dev-secret-change-in-production}
  expiration: 86400000
```

<!-- ======================================================
     ADD NEW SECTIONS BELOW THIS LINE
     Template:

## Topic Name

### What I was trying to do

```bash
the command or snippet
```

> Optional note about gotchas or context.

======================================================= -->
