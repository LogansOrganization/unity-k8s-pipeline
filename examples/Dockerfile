# Copy this to the repo root of a new game repo (as `Dockerfile`). Packages
# the compiled server build (produced by the Job this pipeline runs) into a
# minimal runtime image. Only needs changing if the server needs extra native
# dependencies beyond libc.
FROM ubuntu:22.04
COPY buildServer/ .
RUN chmod +x Server
EXPOSE 7777/udp
ENTRYPOINT ["./Server", "-batchmode", "-nographics"]
