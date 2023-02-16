ARG EE_BASE_IMAGE=thedragoon/python3-10
ARG EE_BUILDER_IMAGE=thedragoon/python3-10

FROM $EE_BASE_IMAGE as galaxy
ARG ANSIBLE_GALAXY_CLI_COLLECTION_OPTS=
ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS=--ignore-certs
USER root

# BEGIN (remove this when we move back to using ansible-builder)
RUN apk update \
  && pip3 install -U pip && pip3 install ansible-core
# END (remove this when we move back to using ansible-builder)

ADD _build /build
WORKDIR /build

RUN ansible-galaxy role install $ANSIBLE_GALAXY_CLI_ROLE_OPTS -r requirements.yml --roles-path "/usr/share/ansible/roles"
RUN ANSIBLE_GALAXY_DISABLE_GPG_VERIFY=1 ansible-galaxy collection install $ANSIBLE_GALAXY_CLI_COLLECTION_OPTS -r requirements.yml --collections-path "/usr/share/ansible/collections"

FROM $EE_BUILDER_IMAGE as builder

COPY --from=galaxy /usr/share/ansible /usr/share/ansible

USER root
# BEGIN (remove this when we move back to using ansible-builder)
RUN pip3 install -U pip && pip3 install ansible-builder wheel \
  && apk add git && git config --global http.sslVerify false
# END (remove this when we move back to using ansible-builder)

ADD _build/requirements.txt requirements.txt
ADD _build/bindep.txt bindep.txt
RUN ansible-builder introspect --sanitize --user-pip=requirements.txt --user-bindep=bindep.txt --write-bindep=/tmp/src/bindep.txt --write-pip=/tmp/src/requirements.txt
# BEGIN (remove this when we move back to using ansible-builder)
ADD scripts/assemble /usr/local/bin/assemble
RUN chmod +x /usr/local/bin/assemble

ADD https://raw.githubusercontent.com/ansible/python-builder-image/main/scripts/get-extras-packages /usr/local/bin
RUN chmod +x /usr/local/bin/get-extras-packages
# END (remove this when we move back to using ansible-builder)

RUN assemble

FROM $EE_BASE_IMAGE
USER root

COPY --from=galaxy /usr/share/ansible /usr/share/ansible

COPY --from=builder /output/ /output/

# BEGIN (remove this when we move back to using ansible-builder)
ADD scripts/install-from-bindep /usr/local/bin/install-from-bindep
RUN chmod +x /usr/local/bin/install-from-bindep

ADD scripts/entrypoint.sh /usr/local/bin/entrypoint
RUN chmod 755 /usr/local/bin/entrypoint

ENTRYPOINT ["entrypoint"]

# NB: this appears to be necessary for container builds based on this container, since we can't rely on the entrypoint
# script to run during a build to fix up /etc/passwd. This envvar value, and the fact that all user homedirs are
# set to /home/runner is an implementation detail that may change with future versions of runner and should not be
# assumed by other code or tools.
ENV HOME=/home/runner

RUN apk update && pip3 install -U pip \
  && apk add git && git config --global http.sslVerify false

RUN install-from-bindep && rm -rf /output/wheels

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

RUN  git -c http.sslVerify=false clone https://github.com/ansible/ansible-runner.git /runner/ansible-runner \
  && cd /runner/ansible-runner/ \
  && pip3 install -e /runner/ansible-runner/ 
USER root
WORKDIR /runner

# COPY --from=quay.io/ansible/receptor:devel /usr/bin/receptor /usr/bin/receptor
ADD scripts/go-excludes go-excludes
RUN apk add go=1.19.5-r0 \
  && git clone https://github.com/ansible/receptor \
  && sed -i 's/quic-go\ v0.28.1/quic-go\ v0.29.2/' receptor/go.mod \
  && cat go-excludes >> receptor/go.mod \
  && rm go-excludes \
  && cd receptor/cmd/receptor-cl/ \
  && go mod tidy \
  && go build -o /usr/bin/receptor \
  && cd /runner \
  && rm -r receptor/ \
  && apk del go \
  && rm -r /home/runner/go 

RUN mkdir -p /var/run/receptor
ADD https://raw.githubusercontent.com/ansible/awx-ee/devel/run.sh /run.sh
RUN sed -i 's/bash/sh/' /run.sh \
  && chmod ug+x /run.sh \
  && deluser --remove-home meshuggah \
  && git lfs install
USER 1000
CMD /run.sh
LABEL ansible-execution-environment=true
