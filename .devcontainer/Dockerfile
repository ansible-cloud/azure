FROM quay.io/scottharwell/vscode-ansible:latest

ARG AZURE_COLLECTION_VERSION="v1.10.0"
ARG CONTAINER_USER="vscode"

# Ensure PIP is upgraded
RUN runuser -l ${CONTAINER_USER} -c "pip3 install pip --upgrade"

# Download Azure Collection Depedendencies - Note AZ CLI and collection have conlficting dependencies and cannot be installed together.
RUN runuser -l ${CONTAINER_USER} -c "git clone https://github.com/ansible-collections/azure.git /tmp/azure"
RUN runuser -l ${CONTAINER_USER} -c "cd /tmp/azure; git checkout ${AZURE_COLLECTION_VERSION}"
RUN runuser -l ${CONTAINER_USER} -c "pip3 install -r /tmp/azure/requirements-azure.txt"
RUN runuser -l ${CONTAINER_USER} -c "ansible-galaxy collection install azure.azcollection"