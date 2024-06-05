# tl;dr
testing out docker with an R and a Python script. I had a ton of issues with using conda in docker, so this readme will detail those lessons learned. I will also detail lessons learned about running both R and Python in one dockerfile, why it was a mess, and why I switch to using docker compose

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
