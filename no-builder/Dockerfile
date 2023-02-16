FROM quay.io/ansible/awx-ee:latest

ARG ANSIBLE_GALAXY_CLI_ROLE_OPTS=--ignore-certs
USER root

ADD _build /build
WORKDIR /build

RUN yum install gcc python38-devel libvirt-devel git-lfs -yml
RUN pip install -r requirements.txt

RUN ansible-galaxy role install $ANSIBLE_GALAXY_CLI_ROLE_OPTS -r requirements.yml --roles-path "/usr/share/ansible/roles"
RUN ANSIBLE_GALAXY_DISABLE_GPG_VERIFY=1 ansible-galaxy collection install -r requirements.yml --collections-path "/usr/share/ansible/collections"

USER 1000
