# Docker Sandbox IDE Environment

This `docker-compose.yaml` file helps quickly deploy a IDE environment with a CLI pane and text editor with syntax highlighting. 

Enter the following commands in the current directory to start Cloud9's web IDE to create stylebook `.yamls` files.

```
export DATA_DIR=/tmp
sudo docker-compose up -d
```

> It is assumed you already have [Docker installed](https://docs.docker.com/engine/installation/). You can set any existing directory for `DATA_DIR` environment variable to share persistent data between your IDE and your host. 

Access your web IDE at [`http://localhost:9090`](http://localhost:9090)