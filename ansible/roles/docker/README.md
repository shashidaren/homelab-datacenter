# Docker Role

## 📌 Purpose

Installs and configures Docker Engine on Rocky Linux 9 servers.

---

## ⚙️ What this role does

* Removes Podman and Buildah
* Installs Docker CE and dependencies
* Adds Docker repository
* Starts and enables Docker service
* Adds user to docker group

---

## 📂 Variables

| Variable    | Default      | Description                |
| ----------- | ------------ | -------------------------- |
| docker_user | ansible_user | User added to docker group |

---

## ▶️ Usage

```yaml
- hosts: docker
  roles:
    - docker
```

---

## 🧪 Validation

After running:

```bash
docker --version
docker compose version
docker run hello-world
```

---

## ⚠️ Notes

* Requires Rocky Linux 9 / EL9
* Requires internet access for Docker repo
* User must re-login after group assignment

