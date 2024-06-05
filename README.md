# tl;dr
testing out docker with an R and a Python script. I had a ton of issues with using conda in docker, so this readme will detail those lessons learned. I will also detail lessons learned about running both R and Python in one dockerfile, why it was a mess, and why I switch to using docker compose

# toc

- [conda](#conda)
- [docker-compose](#docker-compose)
- [how to build it](#build-it-all)
- [example output](#example-output)

# conda
one does not simply activate a conda env in docker. this site lays it out very well, I highly recommend reading it to understand why you can't just run `conda activate env` in docker. https://pythonspeed.com/articles/activate-conda-dockerfile/

1. Install conda
I found the most lightweight way to install this is by pulling from an existing docker image
https://stackoverflow.com/questions/69874405/installing-conda-packages-in-docker-via-dockerfile

```
# run the python scripts. first install and activate conda
COPY --from=continuumio/miniconda3:4.12.0 /opt/conda /opt/conda

# set path
ENV PATH=/opt/conda/bin:$PATH

# create an environment from the yaml
RUN set -ex && \
  conda env create --name dkr_env --file=dkr_env.yml

```

2. Activate the conda env you created

```
# Make RUN commands use the new environment:
SHELL ["conda", "run", "-n", "dkr_env", "/bin/bash", "-c"]
```


3. Run your python script within the env context

```
# The code to run when container is started:
ENTRYPOINT ["conda", "run", "--no-capture-output", "-n", "dkr_env", "python", "test.py"]
```

# docker compose

Running a separate R and Python script in the same Dockerfile didn't work well. Turns out, Docker doesn't really recommend running multiple scripts in one container. You can totally do it, but it works much easier if you have one trigger script that executes all the scripts, or something like docker compose. See their documentation for that here https://docs.docker.com/config/containers/multi-service_container/

The answer in this post https://stackoverflow.com/questions/27409761/docker-multiple-dockerfiles-in-project was helpful and encouraged me to make separate dockerfiles for each process, and then compose their builds with docker compose.

## structure

take a look at the directory, I have a main `docker-compose.yml`, there are separate folders for python and r, and dockerfiles in each.

```
.
├── docker-compose.yml
├── python
│   ├── Dockerfile
│   ├── dkr_env.yml
│   └── test.py
└── r
    ├── Dockerfile
    ├── renv
    ├── renv.lock
    └── test.R

3 directories, 9 files

```

the compose **did not** like it when I named the folders with capital letters. I could not use _R_ but had to name it _r_. 

## docker-compose.yml

This is the main trigger. It references the main folders and then since there are Dockerfiles in each folder I can reference the build by declaring the root of the folder, like `./r` for the r folder:

```
services:
  python:
    build: ./python
  r:
    build: ./r
```

## r Dockerfile

I found R and renv to be far more simple than using python/conda :( 

the rocker project https://rocker-project.org/ is fantastic and has lots of options to install different things in R. I'm using it to install R and ubuntu. Then,

- copy all the files from the r folder
- install the packages
- then run the script

```
FROM rocker/r-ver:4.4.0

COPY . .


# run the R scripts
RUN Rscript -e "renv::restore()"

CMD ["Rscript", "test.R"]

```
## python Dockerfile

For python there are probably easier ways to install it, but for conda i dont think it gets easier. Please see the [conda](#conda) section for more details.

- install ubuntu
- build ubuntu packages
- install python
- copy over all files from the python folder
- install conda
- declare the conda path
- make a conda env and install the packages from the environment.yml
- you can't run conda activate, so you'll need to write workarounds and make sure the container runs your script within the conda context

```
FROM ubuntu:latest

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && apt-get install -y --no-install-recommends build-essential

RUN apt-get install -y python3

COPY . .

# run the python scripts. first install and activate conda
COPY --from=continuumio/miniconda3:4.12.0 /opt/conda /opt/conda

ENV PATH=/opt/conda/bin:$PATH

# Usage examples
RUN set -ex && \
  conda env create --name dkr_env --file=dkr_env.yml


# Make RUN commands use the new environment:
SHELL ["conda", "run", "-n", "dkr_env", "/bin/bash", "-c"]

# CMD ["python", "test.py"]

# The code to run when container is started:
ENTRYPOINT ["conda", "run", "--no-capture-output", "-n", "dkr_env", "python", "test.py"]

```

# build it all

here are helpful commands on how to build with docker compose https://github.com/tldr-pages/tldr/blob/main/pages/common/docker-compose.md

```
docker compose up --build
```

# example output

here's what you should see as an output

```
(base) frankthetank:~/Projects/personal/docker-test$ docker compose up --build
[+] Building 1.4s (24/24) FINISHED                                            docker:default
 => [r internal] load .dockerignore                                                     0.0s
 => => transferring context: 2B                                                         0.0s
 => [r internal] load build definition from Dockerfile                                  0.0s
 => => transferring dockerfile: 154B                                                    0.0s
 => [r internal] load metadata for docker.io/rocker/r-ver:4.4.0                         1.2s
 => [python internal] load build definition from Dockerfile                             0.0s
 => => transferring dockerfile: 735B                                                    0.0s
 => [python internal] load .dockerignore                                                0.0s
 => => transferring context: 2B                                                         0.0s
 => [python internal] load metadata for docker.io/continuumio/miniconda3:4.12.0         1.2s
 => [python internal] load metadata for docker.io/library/ubuntu:latest                 1.2s
 => [r auth] rocker/r-ver:pull token for registry-1.docker.io                           0.0s
 => [python auth] continuumio/miniconda3:pull token for registry-1.docker.io            0.0s
 => [python auth] library/ubuntu:pull token for registry-1.docker.io                    0.0s
 => [r 1/3] FROM docker.io/rocker/r-ver:4.4.0@sha256:77679d3f7b4d774d87be238641dc42619  0.0s
 => [r internal] load build context                                                     0.0s
 => => transferring context: 15.11kB                                                    0.0s
 => [python stage-0 1/6] FROM docker.io/library/ubuntu:latest@sha256:e3f92abc0967a6c19  0.0s
 => [python internal] load build context                                                0.0s
 => => transferring context: 89B                                                        0.0s
 => [python] FROM docker.io/continuumio/miniconda3:4.12.0@sha256:977263e8d1e476972fdda  0.0s
 => CACHED [python stage-0 2/6] RUN apt-get update && apt-get install -y --no-install-  0.0s
 => CACHED [python stage-0 3/6] RUN apt-get install -y python3                          0.0s
 => CACHED [python stage-0 4/6] COPY . .                                                0.0s
 => CACHED [python stage-0 5/6] COPY --from=continuumio/miniconda3:4.12.0 /opt/conda /  0.0s
 => CACHED [python stage-0 6/6] RUN set -ex &&   conda env create --name dkr_env --fil  0.0s
 => CACHED [r 2/3] COPY . .                                                             0.0s
 => CACHED [r 3/3] RUN Rscript -e "renv::restore()"                                     0.0s
 => [r] exporting to image                                                              0.0s
 => => exporting layers                                                                 0.0s
 => => writing image sha256:8035eb67524ba1008d1b1fc671644a760b3412ad4dcaae9e2a6213aea4  0.0s
 => => naming to docker.io/library/docker-test-r                                        0.0s
 => [python] exporting to image                                                         0.0s
 => => exporting layers                                                                 0.0s
 => => writing image sha256:bdfd4d1cdca9505449215a7b60ed99c7a9c6aed9c61faafcf28c2c2a9c  0.0s
 => => naming to docker.io/library/docker-test-python                                   0.0s
[+] Running 3/1
 ✔ Network docker-test_default     Created                                              0.1s
 ✔ Container docker-test-r-1       Created                                              0.1s
 ✔ Container docker-test-python-1  Created                                              0.1s
Attaching to docker-test-python-1, docker-test-r-1
docker-test-python-1  | hello python
docker-test-python-1 exited with code 0
docker-test-r-1       |
docker-test-r-1       | NOTE: Dependency discovery took 77 seconds during snapshot.
docker-test-r-1       | Consider using .renvignore to ignore files, or switching to explicit snapshots.
docker-test-r-1       | See `?renv::dependencies` for more information.
docker-test-r-1       |
docker-test-r-1       | - //usr/share/i18n/charmaps              : 233 files
docker-test-r-1       | - //lib/x86_64-linux-gnu/gconv           : 256 files
docker-test-r-1       | - //usr/share/doc                        : 261 files
docker-test-r-1       | - //etc/ssl/certs                        : 276 files
docker-test-r-1       | - //sys/kernel/slab                      : 323 files
docker-test-r-1       | - //usr/share/i18n/locales               : 362 files
docker-test-r-1       | - //lib/x86_64-linux-gnu                 : 435 files
docker-test-r-1       | - //bin                                  : 445 files
docker-test-r-1       | - //usr/include/linux                    : 564 files
docker-test-r-1       | - //usr/share/bash-completion/completions: 826 files
docker-test-r-1       | - //var/lib/dpkg/info                    : 1196 files
docker-test-r-1       |
docker-test-r-1       | - The project is out-of-sync -- use `renv::status()` for details.
docker-test-r-1       |
docker-test-r-1       | Attaching package: ‘dplyr’
docker-test-r-1       |
docker-test-r-1       | The following objects are masked from ‘package:stats’:
docker-test-r-1       |
docker-test-r-1       |     filter, lag
docker-test-r-1       |
docker-test-r-1       | The following objects are masked from ‘package:base’:
docker-test-r-1       |
docker-test-r-1       |     intersect, setdiff, setequal, union
docker-test-r-1       |
docker-test-r-1       | [1] "hello R"
docker-test-r-1 exited with code 0
```
