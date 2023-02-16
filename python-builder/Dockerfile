ARG CONTAINER_IMAGE=python:3.10-alpine3.17
ARG REMOTE_SOURCE=.
ARG REMOTE_SOURCE_DIR=/remote-source
ARG REMOTE_SOURCE_APP_DIR=$REMOTE_SOURCE_DIR

FROM $CONTAINER_IMAGE as builder
# =============================================================================
ARG REMOTE_SOURCE_DIR
COPY $REMOTE_SOURCE $REMOTE_SOURCE_DIR

FROM $CONTAINER_IMAGE
# =============================================================================
ARG REMOTE_SOURCE_APP_DIR

RUN apk update \
  && apk add git py3-wheel

RUN git -c http.sslVerify=false clone https://github.com/ansible/python-builder-image.git $REMOTE_SOURCE_APP_DIR

RUN mkdir /output \ 
  && sed -i 's/\!\/bin\/bash/\!\/bin\/sh/' $REMOTE_SOURCE_APP_DIR/scripts/assemble \
  && sed -i 's/"${PKGMGR:-}"/\/sbin\/apk/' $REMOTE_SOURCE_APP_DIR/scripts/assemble \
  && sed -i 's/$PKGMGR install -y/$PKGMGR add/' $REMOTE_SOURCE_APP_DIR/scripts/assemble \
  && sed -i 's/$PKGMGR update -y/$PKGMGR update/' $REMOTE_SOURCE_APP_DIR/scripts/assemble \
  && head -n -5 $REMOTE_SOURCE_APP_DIR/scripts/assemble > $REMOTE_SOURCE_APP_DIR/scripts/assemble-edited \
  && cp $REMOTE_SOURCE_APP_DIR/scripts/assemble-edited /usr/local/bin/assemble \
  && chmod u+x /usr/local/bin/assemble \
  && cp $REMOTE_SOURCE_APP_DIR/scripts/get-extras-packages /usr/local/bin/get-extras-packages \
  && cp $REMOTE_SOURCE_APP_DIR/scripts/install-from-bindep /output/install-from-bindep

WORKDIR $REMOTE_SOURCE_APP_DIR

RUN ls -l \
  && cat build-requirements.txt requirements.txt | sort > upper-constraints.txt \
  && pip3 install --no-cache-dir -r build-requirements.txt -c upper-constraints.txt \
  && pip3 install --no-cache-dir -r requirements.txt -c upper-constraints.txt

WORKDIR /
RUN rm -rf $REMOTE_SOURCE_APP_DIR
