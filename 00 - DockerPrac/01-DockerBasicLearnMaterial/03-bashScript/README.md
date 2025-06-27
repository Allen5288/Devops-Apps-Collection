# Project: Bash Script Automation in Docker

This project demonstrates how to package and run a Bash automation script inside a Docker container. The script simulates a simple application lifecycle with start, process, and finish steps, and is useful for learning how to containerize CLI tools or automation tasks.

## Key Concepts

- Writing and running Bash scripts in Docker
- Building minimal images for automation
- Interactive container execution
- Custom entrypoints and process control

---

```bash
docker build -t app .

docker run -it --name=app_container_app app
```
