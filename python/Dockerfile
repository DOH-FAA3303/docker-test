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


