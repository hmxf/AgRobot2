# AgRobot version 2

This Repo is is a branch of [AgRobot 2](https://github.com/dslab-agrobot/AgRobot2). Only file organization structure has been modified, the actual content remains consistent.

Will be merged into mainline version in a future version.

DSLAB AgRobot project from 2022-2024. 

Detailed introduction in [here](https://dslab-agrobot.github.io/AgRobot2/)


The file structure should be:

```
.
├── LICENSE
├── README.md
├── scripts
│   ├── management_scripts
│   └── tools
└── src
    ├── ros_ht_msg
    └── ros_modbus_msg
```
Here the defination of these folders: 


- `src` contains resources for ROS, Modbus, AI functions. Most of them are developed alone and imported as submodules.
- `scripts/management_scripts`, Enviroment management for this robot and corresponding package. e.g., flash the system of the NVDIA-NX, python and ROS setup.
- `scripts/tools` for misc scripts, like file transfer, image crop for AI, blablabla.
