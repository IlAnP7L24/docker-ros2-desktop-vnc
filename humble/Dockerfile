# Copyright 2020-2023 Tiryoh<tiryoh@gmail.com>
# 
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
# 
#     http://www.apache.org/licenses/LICENSE-2.0
# 
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
# 
# This Dockerfile is based on https://github.com/AtsushiSaito/docker-ubuntu-sweb
# which is released under the Apache-2.0 license.

FROM ubuntu:jammy-20230816

ARG TARGETPLATFORM
LABEL maintainer="Tiryoh<tiryoh@gmail.com>"

SHELL ["/bin/bash", "-c"]

# Upgrade OS
RUN apt-get update -q && \
    DEBIAN_FRONTEND=noninteractive apt-get upgrade -y && \
    apt-get autoclean && \
    apt-get autoremove && \
    rm -rf /var/lib/apt/lists/*

# Install Ubuntu Mate desktop
RUN apt-get update -q && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
        ubuntu-mate-desktop && \
    apt-get autoclean && \
    apt-get autoremove && \
    rm -rf /var/lib/apt/lists/*

# Add Package
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
        tigervnc-standalone-server tigervnc-common \
        supervisor wget curl gosu git sudo python3-pip tini \
        build-essential vim sudo lsb-release locales \
        bash-completion tzdata terminator && \
    apt-get autoclean && \
    apt-get autoremove && \
    rm -rf /var/lib/apt/lists/*

# noVNC and Websockify
RUN git clone https://github.com/AtsushiSaito/noVNC.git -b add_clipboard_support /usr/lib/novnc
RUN pip install git+https://github.com/novnc/websockify.git@v0.10.0
RUN ln -s /usr/lib/novnc/vnc.html /usr/lib/novnc/index.html

# Set remote resize function enabled by default
RUN sed -i "s/UI.initSetting('resize', 'off');/UI.initSetting('resize', 'remote');/g" /usr/lib/novnc/app/ui.js

# Disable auto update and crash report
RUN sed -i 's/Prompt=.*/Prompt=never/' /etc/update-manager/release-upgrades
RUN sed -i 's/enabled=1/enabled=0/g' /etc/default/apport

# Enable apt-get completion
RUN rm /etc/apt/apt.conf.d/docker-clean

# Install Firefox
RUN DEBIAN_FRONTEND=noninteractive add-apt-repository ppa:mozillateam/ppa -y && \
    echo 'Package: *' > /etc/apt/preferences.d/mozilla-firefox && \
    echo 'Pin: release o=LP-PPA-mozillateam' >> /etc/apt/preferences.d/mozilla-firefox && \
    echo 'Pin-Priority: 1001' >> /etc/apt/preferences.d/mozilla-firefox && \
    apt-get update -q && \
    apt-get install -y \
    firefox && \
    apt-get autoclean && \
    apt-get autoremove && \
    rm -rf /var/lib/apt/lists/*

# Install VSCodium
RUN wget -qO - https://gitlab.com/paulcarroty/vscodium-deb-rpm-repo/raw/master/pub.gpg \
    | gpg --dearmor \
    | dd of=/usr/share/keyrings/vscodium-archive-keyring.gpg && \
    echo 'deb [ signed-by=/usr/share/keyrings/vscodium-archive-keyring.gpg ] https://download.vscodium.com/debs vscodium main' \
    | tee /etc/apt/sources.list.d/vscodium.list && \
    apt-get update -q && \
    apt-get install -y codium && \
    apt-get autoclean && \
    apt-get autoremove && \
    rm -rf /var/lib/apt/lists/*

# Install ROS
ENV ROS_DISTRO humble
# desktop or ros-base
ARG INSTALL_PACKAGE=desktop

RUN apt-get update -q && \
    apt-get install -y curl gnupg2 lsb-release && \
    curl -sSL https://raw.githubusercontent.com/ros/rosdistro/master/ros.key -o /usr/share/keyrings/ros-archive-keyring.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/ros-archive-keyring.gpg] http://packages.ros.org/ros2/ubuntu $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/ros2.list > /dev/null && \
    apt-get update -q && \
    apt-get install -y ros-${ROS_DISTRO}-${INSTALL_PACKAGE} \
    python3-argcomplete \
    python3-colcon-common-extensions \
    python3-rosdep python3-vcstool && \
    rosdep init && \
    rm -rf /var/lib/apt/lists/*

RUN rosdep update

# Install simulation package only on amd64
# Not ready for arm64 for now (July 28th, 2020)
# https://github.com/Tiryoh/docker-ros2-desktop-vnc/pull/56#issuecomment-1196359860
RUN if [ "$TARGETPLATFORM" = "linux/amd64" ]; then \
    apt-get update -q && \
    apt-get install -y \
    ros-${ROS_DISTRO}-gazebo-ros-pkgs \
    ros-${ROS_DISTRO}-ros-ign && \
    rm -rf /var/lib/apt/lists/*; \
    fi

#############################################
# --- Install required packages ---
RUN curl -sSL http://get.gazebosim.org | sh
RUN sudo apt update && sudo apt install software-properties-common \
             # ROS2 navigation and localization packages
             ros-$ROS_DISTRO-navigation2 \
             ros-$ROS_DISTRO-nav2-bringup \
             ros-$ROS_DISTRO-turtlebot3 \
             ros-$ROS_DISTRO-turtlebot3-gazebo \
             ros-$ROS_DISTRO-joint-state-publisher-gui \
             ros-$ROS_DISTRO-xacro \
             ros-$ROS_DISTRO-gazebo-ros-pkgs \
             ros-$ROS_DISTRO-rqt-robot-steering \ 
             ros-$ROS_DISTRO-robot-localization \
             ros-$ROS_DISTRO-vision-opencv \ 
             ros-$ROS_DISTRO-message-filters \
             # Install orb-slam required packages
             libopencv-dev opencv-data \
             libglew-dev ffmpeg libboost-system-dev libeigen3-dev -y


#############################################
# --- Build Pangolin from source ---
RUN git clone https://github.com/stevenlovegrove/Pangolin.git
RUN mkdir -p /home/${USERNAME}/Pangolin/build
RUN cd Pangolin/build && cmake .. && cmake --build .


#############################################
# --- Build ORB-SLAM3 from source
WORKDIR /home/${USERNAME}

RUN git clone https://github.com/UZ-SLAMLab/ORB_SLAM3.git ~/ORB_SLAM3

# Change to c++14 for compile error
# Ref : https://github.com/UZ-SLAMLab/ORB_SLAM3/issues/387
# Ref : https://github.com/Soldann/MORB_SLAM/issues/3
RUN sed -i 's/++11/++14/g' ORB_SLAM3/CMakeLists.txt && \
    sed -i 's/++11/++14/g' ORB_SLAM3/Thirdparty/g2o/CMakeLists.txt && \
    sed -i 's/++11/++14/g' ORB_SLAM3/Thirdparty/Sophus/CMakeLists.txt

RUN cd ORB_SLAM3 && ./build.sh && \
    cd Thirdparty/Sophus/build && sudo make install


#############################################
# --- Build ros2-orbslam3 package ---
RUN mkdir -p /home/${USERNAME}/${ROS_ORB_WS}/src

WORKDIR /home/${USERNAME}/${ROS_ORB_WS}

RUN colcon build && git clone https://github.com/zang09/ORB_SLAM3_ROS2.git ./src/orbslam3_ros2

# Fix path error and colcon build error
# Ref : https://github.com/alsora/ros2-ORB_SLAM2/issues/8#issuecomment-1461570970
RUN sed -i 's/foxy\/lib\/python3.8/humble\/lib\/python3.10/g' src/orbslam3_ros2/CMakeLists.txt && \
    sed -i 's/Pangolin)/Pangolin OpenCV)/g' src/orbslam3_ros2/CMakeLists.txt && \
    sed -i 's/Install\/ORB_SLAM\/ORB_SLAM3/ORB_SLAM3/g' src/orbslam3_ros2/CMakeModules/FindORB_SLAM3.cmake && \
    # Source and build the package
    . /opt/ros/humble/setup.sh && colcon build --symlink-install --packages-select orbslam3


COPY ./entrypoint.sh /
ENTRYPOINT [ "/bin/bash", "-c", "/entrypoint.sh" ]

ENV USER ubuntu
ENV PASSWD ubuntu

