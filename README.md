# 🐳 Docker Dev Environment with SSH Access

This setup provides a Docker container with SSH access, optional Docker daemon exposure, and key-based authentication. Ideal for local testing, DevOps workflows, or remote debugging inside a secure sandbox.

---

## 📁 Initial Setup

```bash
mkdir -p ./docker_dev
cd docker_dev
mkdir -p ssh-keys
touch ./ssh-keys/authorized_keys
```

---

## 📦 Dockerfile

```bash
nano Dockerfile
```

Paste the following content:

<details>
<summary>Click to expand Dockerfile</summary>

```dockerfile
FROM docker:latest

RUN apk update && apk add --no-cache \
    openssh \
    bash \
    curl \
    nano \
    wget \
    git \
    iptables \
    ip6tables \
    && mkdir -p /var/run/sshd /root/.ssh

RUN echo 'root:rootpassword' | chpasswd

RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config && \
    sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config && \
    sed -i 's/#AllowTcpForwarding no/AllowTcpForwarding yes/' /etc/ssh/sshd_config && \
    sed -i 's/AllowTcpForwarding no/AllowTcpForwarding yes/' /etc/ssh/sshd_config && \
    echo 'AllowTcpForwarding yes' >> /etc/ssh/sshd_config

RUN ssh-keygen -A

WORKDIR /workspace

COPY ssh-keys/authorized_keys /root/.ssh/authorized_keys

RUN chmod 700 /root/.ssh && \
    chmod 600 /root/.ssh/authorized_keys

EXPOSE 22 2375 
EXPOSE 16000-16100

CMD ["/bin/sh", "-c", "dockerd-entrypoint.sh & exec /usr/sbin/sshd -D"]
```

</details>

---

## 🐙 Docker Compose

```bash
nano compose.yml
```

Paste this content:

```yaml
services:
  docker_dev:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: docker_dev
    restart: unless-stopped
    hostname: docker_dev
    privileged: true
    ports:
      - "56222:22"
      - "56000-56100:16000-16100"
    volumes:
      - ./workspace:/workspace
    networks:
      - docker_dev

networks:
  docker_dev:
    name: docker_dev
    driver: bridge
```

---

## ▶️ Run the Container

```bash
docker compose up -d --build
docker compose logs -f
```

---

## 🔐 Generate SSH Key Inside Container

```bash
docker exec -it docker_dev ssh-keygen -t ed25519 -f /root/.ssh/id_ed25519 -N ""
```

---

## 📤 Copy SSH Keys to Host

```bash
mkdir -p ssh-keys
docker exec docker_dev cat /root/.ssh/id_ed25519.pub > ssh-keys/id_ed25519.pub
docker exec docker_dev cat /root/.ssh/id_ed25519 > ssh-keys/id_ed25519
chmod 600 ssh-keys/id_ed25519
```

---

## ➕ Add Public Key to Authorized Keys (If Not Already Present)

```bash
docker exec -it docker_dev sh -c 'grep -qxFf /root/.ssh/id_ed25519.pub /root/.ssh/authorized_keys || cat /root/.ssh/id_ed25519.pub >> /root/.ssh/authorized_keys'
```

---

## 🔗 Connect via SSH

```bash
ssh root@localhost -p 56222 -i ./ssh-keys/id_ed25519
```

Or skip host checks:

```bash
ssh -o StrictHostKeyChecking=no root@localhost -p 56222 -i ./ssh-keys/id_ed25519
```

Remote IP example:

```bash
ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null root@<your-ip> -p 56222
```
