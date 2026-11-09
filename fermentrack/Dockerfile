#
# Dockerfile for Fermentrack image, this version can be generated for both amd64/i386/armv7
#
FROM debian:bullseye-slim
EXPOSE 8080/tcp
#
# Install the needed dependecies, some are only used at the installphase so those will be removed later.
#
ENV DEBIAN_FRONTEND noninteractive
#
# Debian 11 (bullseye) is end-of-life and its packages have moved off the regular mirrors, so install from
# archive.debian.org. (Staying on bullseye keeps Python 3.9, which Fermentrack's pinned dependencies require.)
#
RUN printf '%s\n' \
  'deb http://archive.debian.org/debian bullseye main' \
  'deb http://archive.debian.org/debian bullseye-updates main' \
  'deb http://archive.debian.org/debian-security bullseye-security main' \
  > /etc/apt/sources.list
RUN apt-get update && apt-get install --no-install-recommends -y \
  nginx \
  redis-server \
  python3.9 \
  python3-pip \
  python3-dev \
  python3-venv \
  python3-zmq \
  python3-scipy \
  python3-numpy \
  libbluetooth-dev \
  bluez \
  libcap2-bin\
  libbluetooth3 \
  ca-certificates \
  build-essential \
  git-core \
  avrdude \
  sudo \
  libffi-dev \
  libssl-dev \
  libcap2-bin \
  curl \
# For debugging
# procps
# net-tools
  nano \
  usbutils
#
# Create the users, group and directories for git. Its needed for fermentrack to work correcly.
# User fermentrack is added to the dialout group on order to have access to firmware flashing when running as non root. 
#
RUN groupadd -g 2000 fermentrack && useradd -m -u 2001 -g fermentrack fermentrack
RUN groupadd -g 2002 nginx && useradd --system -M -u 2003 -g nginx nginx -s /usr/sbin/nologin
RUN usermod -a -G dialout fermentrack
#
# Install fermentrack, just make a copy of git and we will handle the run commands in the supervisor
#
WORKDIR /home/fermentrack
USER fermentrack:fermentrack
#
# Clone fermentrack into the image. Override the GitHub repo/branch with build args, e.g.
#   docker compose build --build-arg FERMENTRACK_REF=home-assistant-api
#
ARG FERMENTRACK_GITHUB=carjed/fermentrack
ARG FERMENTRACK_REF=custom
#
# Cache-buster: Docker re-downloads remote ADD sources on every build and invalidates the following layers when the
# content changes. This file contains the branch's head commit, so the clone below re-runs whenever the branch moves
# (and stays cached when it hasn't).
#
ADD https://api.github.com/repos/${FERMENTRACK_GITHUB}/git/refs/heads/${FERMENTRACK_REF} /tmp/fermentrack-ref.json
RUN git clone --branch ${FERMENTRACK_REF} https://github.com/${FERMENTRACK_GITHUB}.git /home/fermentrack/fermentrack
#
# Setup python and install the packages
#
USER root
# With the introduction of docker it uses /app for launching processes
RUN ln -s /home/fermentrack/fermentrack /app
WORKDIR /home/fermentrack
USER fermentrack
ENV VIRTUAL_ENV="/home/fermentrack/venv"
ENV PATH="/home/fermentrack/venv/bin:/usr/bin:/usr/sbin:/bin:/sbin:/home/fermentrack/.local/bin"
ENV PYTHONPATH=":;/home/fermentrack/fermentrack;/home/fermentrack/venv/bin;/home/fermentrack/venv/lib/python3.8/site-packages"
ENV USE_DOCKER=true
RUN python3 -m pip install virtualenv
RUN virtualenv venv
USER fermentrack
RUN python3 -m pip install --no-binary numpy numpy==1.18.4
RUN python3 -m pip install --no-binary pyzmq pyzmq==19.0.1
RUN python3 -m pip install circus
RUN python3 -m pip install --no-cache-dir -r /home/fermentrack/fermentrack/requirements/base.txt
RUN chown -R fermentrack:fermentrack /home/fermentrack/venv
#
# Setup nginx and update config to point towards fermentrack. 
#
USER root
COPY default-fermentrack /etc/nginx/sites-available/default-fermentrack
RUN rm -f /etc/nginx/sites-enabled/default
RUN ln -sf /etc/nginx/sites-available/default-fermentrack /etc/nginx/sites-enabled/default-fermentrack
RUN chown -R nginx:nginx /var/log/nginx
RUN chown -R nginx:nginx /var/lib/nginx
RUN touch /run/nginx.pid
RUN chown nginx:nginx /run/nginx.pid
#
# Setup the startup scripts and configure webservers
#
USER root
COPY entrypoint.sh /home/fermentrack/entrypoint.sh
RUN chmod 700 /home/fermentrack/entrypoint.sh
RUN rm -fR /home/fermentrack/fermentrack/data
#
# Generate secret key for this image build
# 
# This section is from the thorrak/fermentrack-tools project (installation script for fermentrack). 
#
RUN echo $(LC_CTYPE=C tr -dc 'a-zA-Z0-9' < /dev/urandom | fold -w 50 | head -n 1) > /home/fermentrack/fermentrack/DJANGO_SECRET_KEY
RUN echo $(LC_CTYPE=C tr -dc 'a-zA-Z0-9' < /dev/urandom | fold -w 32 | head -n 1) > /home/fermentrack/fermentrack/DJANGO_ADMIN_URL
#
# Setup healtcheck
#
USER root
COPY healthcheck.sh /home/fermentrack/healthcheck.sh
RUN chmod 700 /home/fermentrack/healthcheck.sh
HEALTHCHECK --interval=60s --timeout=10s --retries=3 CMD /home/fermentrack/healthcheck.sh
#
# Copy scripts for debugging (tilt)
#
USER fermentrack
COPY tiltview.py /home/fermentrack/tiltview.py
RUN python3 -m pip install beacontools beaconscanner
#
# Start the supervisor
#
USER root
WORKDIR /home/fermentrack
RUN date >> /home/fermentrack/build_info
ENTRYPOINT ["/home/fermentrack/entrypoint.sh"]
CMD [""]
# EOF