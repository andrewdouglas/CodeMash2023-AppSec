# Automating Almost All Application Security Things with CI/CD -- Even Honeypots! #

## Precompiler Prerequisites ##

The precompiler **requires Docker** (Docker Desktop or otherwise), and 4 container images.  Please execute the following commands to download them prior to the session:

```
docker pull owasp/zap2docker-stable:2.12.0
docker pull bkimminich/juice-shop:v14.3.1
docker pull gitlab/gitlab-ee:15.7.0-ee.0
docker pull owasp/modsecurity-crs:3.3.4-apache-202211240811
```

Recommended system specs:
- 16 GB RAM
- An i5 (or better) x64 processor
- Approximately 8 GB of free drive space for the docker images and scratch space

## Precompiler Labs ##

See the ["Labs"](https://github.com/andrewdouglas/CodeMash2023-AppSec/tree/main/Labs) folder in this repository.  Lab instructions are written using markdown (.md) format, so following along in the browser via GitHub is a good way to go.
