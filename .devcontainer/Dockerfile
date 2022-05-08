FROM quay.io/fedora/fedora:35

ARG USERNAME=vscode
ARG USERID=1000
ARG GROUPID=1000

# Upgrade packages and install new packages
RUN dnf install -y gcc make python3-devel python3-wheel oniguruma-devel fish zsh git wget util-linux util-linux-user which vim powerline powerline-fonts vim-powerline azure-cli
RUN dnf upgrade -y

# Create VSCode user
RUN groupadd --gid ${GROUPID} ${USERNAME}
RUN useradd --comment "VSCode User Account" --gid ${GROUPID} --uid ${USERID} -p ${USERNAME} -G wheel -m ${USERNAME}

# Allow wheel to perform sudo actions with no password
RUN sed -e 's/^%wheel/#%wheel/g' -e 's/^# %wheel/%wheel/g' -i /etc/sudoers

# Set workdir to vscode home
USER ${USERNAME}
WORKDIR /home/${USERNAME}

# Update path at build time
ENV PATH="${PATH}:/home/${USERNAME}/.local/bin" 

# Ensure PIP is upgraded and install packages
RUN echo "export PATH=\"$PATH:/home/${USERNAME}/.local/bin\"" >> /home/${USERNAME}/.bash_profile
RUN source /home/${USERNAME}/.bash_profile
RUN python3 -m pip install --upgrade pip
RUN pip3 install ansible ansible-navigator ansible-lint yamllint tox
