# Project: Run Static HTML with Nginx in Docker

This project shows how to use the official nginx Docker image to serve your own static website. By mounting a local directory containing your HTML and assets into the nginx container, you can instantly preview and share your site without any manual nginx configuration.

## Key Concepts

- Using official Docker images (nginx)
- Volume mounting for static site hosting
- Port mapping from container to host
- Windows path compatibility for Docker volume mounts

---

```bash
winpty docker run --name website -v "/i/00_SoftwareDevopment/Devops-Apps-Collection/00 - DockerPrac/01-DockerBasicLearnMaterial/Exercise Files/03_14_before/website:/usr/share/nginx/html" -p 8080:80 --rm nginx
```
