FROM rocker/r-ver:4.4.0

COPY . .


# run the R scripts
RUN Rscript -e "renv::restore()"

CMD ["Rscript", "test.R"]

