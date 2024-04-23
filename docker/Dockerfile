ARG ROS_DISTRO=rolling

FROM osrf/ros:${ROS_DISTRO}-desktop-full as base

# Set the timezone
ENV TZ="Europe/Berlin"

# Avoid interactive prompts from e.g. apt
ARG DEBIAN_FRONTEND="noninteractive"

# Change the default shell to Bash
SHELL [ "/bin/bash" , "-c" ]

# Install common packages
RUN sudo apt update && sudo apt upgrade -y && sudo apt install -y --no-install-recommends \
    bash-completion \
    locales \
    build-essential \
    cmake \
    git \
    openssh-client \
    python3-argcomplete \
    htop \
    gedit \
    nano \
    terminator \
    net-tools \
    python3-pip \
    tmux \
    wget \
    sudo \
    cron \
    # For NUC
    uhubctl && \
    sudo locale-gen en_US.UTF-8 && \
    sudo rm -rf /var/lib/apt/lists/*

# Create the user
RUN groupadd --gid 1000 angsa \
    && useradd --uid 1000 --gid 1000 --groups dialout -m angsa \
    && echo angsa ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/angsa \
    && chmod 0440 /etc/sudoers.d/angsa

USER angsa

ENV HOME /home/angsa
# Add path for pip packages to be found
ENV PATH="${PATH}:$HOME/.local/bin"

# Add our custom bashrc
COPY --chown=1000:1000 angsa_bashrc $HOME/angsa_bashrc
RUN echo "source ~/angsa_bashrc" >> $HOME/.bashrc

WORKDIR /home/angsa
# Install ros tools
RUN sudo apt update && \
    sudo apt install -y ros-dev-tools ccache && \
    sudo rm -rf /var/lib/apt/lists/*
ENV CCACHE_DIR=$HOME/cache/.ccache
RUN colcon mixin add default https://raw.githubusercontent.com/colcon/colcon-mixin-repository/master/index.yaml && colcon mixin update default

# Openvdb for STVL
# https://github.com/SteveMacenski/spatio_temporal_voxel_layer/issues/232
WORKDIR /tmp
RUN wget -q https://github.com/wyca-robotics/openvdb/releases/download/v8.2.0-debian/libopenvdb-dev_8.2.0-1-wyca_amd64.deb && \
    wget -q https://github.com/wyca-robotics/openvdb/releases/download/v8.2.0-debian/libopenvdb8.2_8.2.0-1-wyca_amd64.deb && \
    sudo apt update && sudo apt install -y liblog4cplus-2.0.5 && \
    sudo apt clean && sudo rm -rf /var/lib/apt/lists/ && \
    sudo dpkg -i libopenvdb8.2_8.2.0-1-wyca_amd64.deb && \
    sudo dpkg -i libopenvdb-dev_8.2.0-1-wyca_amd64.deb


FROM base AS overlay

WORKDIR $HOME/overlay_ws/src
COPY underlay.repos .
RUN vcs import < underlay.repos
RUN git clone https://github.com/ros-planning/navigation2

WORKDIR $HOME/overlay_ws
RUN sudo apt update && rosdep update && rosdep install --from-paths src --ignore-src -y -r && sudo rm -rf /var/lib/apt/lists/*
RUN sudo apt update && sudo apt install ros-rolling-ompl ros-rolling-turtlebot3-gazebo && sudo rm -rf /var/lib/apt/lists/*
RUN --mount=type=cache,target=$CCACHE_DIR,uid=1000 source /opt/ros/rolling/setup.bash && colcon build --symlink-install --mixin ccache release --cmake-args -DBUILD_TESTING=OFF || true && \
    ccache --show-stats --verbose


FROM overlay as dev

RUN sudo apt update && sudo apt install -y --no-install-recommends \
    ipython3 \
    && sudo rm -rf /var/lib/apt/lists/*

# Install fzf for fast bash reverse-search
RUN git clone --depth 1 https://github.com/junegunn/fzf.git $HOME/.fzf \
 && $HOME/.fzf/install --all --no-zsh --no-fish

RUN git clone https://github.com/tonynajjar/ros2-aliases $HOME/.ros2-aliases && \
    echo 'source ~/.ros2-aliases/ros2_utils.bash' >> ~/.bashrc

# Colcon configuration
COPY --chown=1000:1000 defaults.yaml $HOME/.colcon/defaults.yaml

WORKDIR $HOME/overlay_ws/src/navigation2
RUN sudo apt update
