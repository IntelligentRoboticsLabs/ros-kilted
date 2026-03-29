# IRL ROS kilted buildfarm (Pixi + Vinca + Rattler)

This repository is a fork of the upstream RoboStack ROS kilted packaging repo:

- https://github.com/RoboStack/ros-kilted

In this fork, we use the same toolchain (Pixi + Vinca + rattler-build) to build and publish **additional ROS 2 kilted packages** produced by the **Intelligent Robotics Lab (IRL)**:

- https://intelligentroboticslab.gsyc.urjc.es/

The resulting binaries are published to prefix.dev (notably the `irl-kilted` channel) and can also be consumed locally via a file-based channel.

## Documentation

- PlanSys2
    - Build & publish (package creators): [irl-docs/plansys2/buildfarm_plansys2.md](irl-docs/plansys2/buildfarm_plansys2.md)
    - User workspace template (Pixi): [irl-docs/plansys2/pixi.toml](irl-docs/plansys2/pixi.toml) and [irl-docs/plansys2/activate.sh](irl-docs/plansys2/activate.sh)

- EasyNav (EasyNavigation) + NavMap
    - Build & publish (package creators): [irl-docs/easynav/buildfarm_easynav.md](irl-docs/easynav/buildfarm_easynav.md)
    - User workspace template (Pixi): [irl-docs/easynav/pixi.toml](irl-docs/easynav/pixi.toml) and [irl-docs/easynav/activate.sh](irl-docs/easynav/activate.sh)

## About Pixi

Pixi is a developer workflow tool that manages reproducible environments, tasks, and activation hooks on top of the Conda ecosystem.

- Pixi: https://pixi.sh/
- Commands & troubleshooting in this repo: [irl-docs/pixi.md](irl-docs/pixi.md)
- Prefix.dev channels:
    - [`irl-kilted`](https://prefix.dev/channels/irl-kilted)
    - [`robostack-kilted`](https://prefix.dev/channels/robostack-kilted)
