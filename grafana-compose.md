```sh
➜  investment git:(master) docker inspect grafana |grep compose
                "com.docker.compose.config-hash": "61f2c56c13daa81d6e37fb898d2e1f28d6543ebec317f18233b6d4ce323bc3e7",
                "com.docker.compose.container-number": "1",
                "com.docker.compose.depends_on": "db:service_started:false",
                "com.docker.compose.image": "sha256:008307cdce1d8c58b21156caa601d77756a16ca6e177a03e537b69904efaa6f6",
                "com.docker.compose.oneoff": "False",
                "com.docker.compose.project": "grafana",
                "com.docker.compose.project.config_files": "/home/shen/workspace/grafana/docker-compose.yaml",
                "com.docker.compose.project.working_dir": "/home/shen/workspace/grafana",
                "com.docker.compose.service": "grafana",
                "com.docker.compose.version": "2.28.1",
```
