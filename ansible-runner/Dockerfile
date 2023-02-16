ARG PYTHON_BASE_IMAGE=python:3.10-alpine3.17
ARG PYTHON_BUILDER_IMAGE=docker.io/thedragoon/python-builder

ARG ANSIBLE_BRANCH=""
ARG ZUUL_SIBLINGS=""

FROM $PYTHON_BUILDER_IMAGE as builder
# =============================================================================
ARG ANSIBLE_BRANCH
ARG ZUUL_SIBLINGS
RUN git -c http.sslVerify=false clone https://github.com/ansible/ansible-runner.git /tmp/src
COPY ./bindep.txt /tmp/src/bindep.txt

RUN if [ "$ANSIBLE_BRANCH" != "" ] ; then \
      echo "Installing requirements.txt / upper-constraints.txt for Ansible $ANSIBLE_BRANCH" ; \
      cp /tmp/src/tools/requirements-$ANSIBLE_BRANCH.txt /tmp/src/requirements.txt ; \
      cp /tmp/src/tools/upper-constraints-$ANSIBLE_BRANCH.txt /tmp/src/upper-constraints.txt ; \
    else \
      echo "Installing requirements.txt" ; \
      cp /tmp/src/tools/requirements.txt /tmp/src/requirements.txt ; \
    fi \
    && cp /tmp/src/tools/build-requirements.txt /tmp/src/build-requirements.txt

# NOTE(pabelanger): For downstream builds, we compile everything from source
# over using existing wheels. Do this upstream too so we can better catch
# issues.

# Commenting due to failures in CI
# ENV PIP_OPTS=--no-build-isolation

RUN assemble

FROM $PYTHON_BASE_IMAGE
# =============================================================================

COPY --from=builder /output/ /output
RUN sed -i 's/\!\/bin\/bash/\!\/bin\/sh/' /output/install-from-bindep \
  && sed -i 's/"${PKGMGR:-}"/\/sbin\/apk/' /output/install-from-bindep \
  && sed -i 's/$PKGMGR install -y/$PKGMGR add/' /output/install-from-bindep \
  && sed -i 's/$PKGMGR update -y/$PKGMGR update/' /output/install-from-bindep \
  && head -n -5 /output/install-from-bindep > /output/install-from-bindep-edited \
  && mv /output/install-from-bindep-edited /output/install-from-bindep \
  && chmod u+x /output/install-from-bindep \
  && /output/install-from-bindep \
  && pip install dumb-init \
  && rm -rf /output

# In OpenShift, container will run as a random uid number and gid 0. Make sure things
# are writeable by the root group.
RUN for dir in \
      /home/runner \
      /home/runner/.ansible \
      /home/runner/.ansible/tmp \
      /runner \
      /home/runner \
      /runner/env \
      /runner/inventory \
      /runner/project \
      /runner/artifacts ; \
    do mkdir -m 0775 -p $dir ; chmod -R g+rwx $dir ; chgrp -R root $dir ; done && \
    for file in \
      /home/runner/.ansible/galaxy_token \
      /etc/passwd \
      /etc/group ; \
    do touch $file ; chmod g+rw $file ; chgrp root $file ; done

WORKDIR /runner

# NB: this appears to be necessary for container builds based on this container, since we can't rely on the entrypoint
# script to run during a build to fix up /etc/passwd. This envvar value, and the fact that all user homedirs are
# set to /home/runner is an implementation detail that may change with future versions of runner and should not be
# assumed by other code or tools.
ENV HOME=/home/runner

ADD scripts/entrypoint.sh /bin/entrypoint
RUN sed -i 's/bash/sh/g' /bin/entrypoint \
  && chmod +x /bin/entrypoint

ENTRYPOINT ["entrypoint"]
CMD ["ansible-runner", "run", "/runner"]
