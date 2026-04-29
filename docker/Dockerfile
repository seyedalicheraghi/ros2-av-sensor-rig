FROM ros:humble-ros-base

ARG DEBIAN_FRONTEND=noninteractive
ARG USERNAME=ros
ARG USER_UID=1000
ARG USER_GID=1000

RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        cmake \
        git \
        sudo \
        curl \
        gnupg2 \
        lsb-release \
        udev \
        usbutils \
        v4l-utils \
        python3-pip \
        python3-colcon-common-extensions \
        python3-rosdep \
        python3-vcstool \
        ros-humble-rmw-cyclonedds-cpp \
        ros-humble-rosbag2 \
        ros-humble-rosbag2-storage-mcap \
        ros-humble-rviz2 \
        ros-humble-tf2-ros \
        ros-humble-image-transport-plugins \
        ros-humble-cv-bridge \
        ros-humble-message-filters \
    && rm -rf /var/lib/apt/lists/*

RUN apt-get update && apt-get install -y --no-install-recommends \
        ros-humble-velodyne \
        ros-humble-velodyne-driver \
        ros-humble-velodyne-pointcloud \
    && rm -rf /var/lib/apt/lists/*

RUN apt-get update && apt-get install -y --no-install-recommends \
        ros-humble-realsense2-camera \
        ros-humble-realsense2-description \
    && rm -rf /var/lib/apt/lists/*

RUN apt-get update && apt-get install -y --no-install-recommends \
        ros-humble-usb-cam \
        ros-humble-camera-calibration \
        ros-humble-image-pipeline \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd --gid ${USER_GID} ${USERNAME} \
    && useradd --uid ${USER_UID} --gid ${USER_GID} -m ${USERNAME} \
    && usermod -aG video,dialout,plugdev ${USERNAME} \
    && echo "${USERNAME} ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/${USERNAME} \
    && chmod 0440 /etc/sudoers.d/${USERNAME}

ENV ROS_DISTRO=humble
ENV RMW_IMPLEMENTATION=rmw_cyclonedds_cpp
ENV WORKSPACE=/workspace

WORKDIR ${WORKSPACE}
RUN chown -R ${USERNAME}:${USERNAME} ${WORKSPACE}

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

USER ${USERNAME}

ENTRYPOINT ["/entrypoint.sh"]
CMD ["bash"]
