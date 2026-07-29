# AutoDRIVE - Installer & ROS 2 Bridge (AIROS) 🏎️



This repository provides a detailed guide to installing the AutoDRIVE simulator and configuring the communication bridge between the AutoDRIVE Simulator and ROS 2 Humble. The goal is to enable control and data acquisition for the F1TENTH vehicle for robotics and autonomous navigation projects.



---



## 📋 Prerequisites



Before starting, ensure your environment meets the following requirements:


* **Operating System:** ([Ubuntu 22.04](https://github.com/widegonz/Ubuntu-Installation))

* **Middleware:** ([ROS 2 Humble](https://docs.ros.org/en/humble/Installation.html))



---







## 🕹️ 1. Simulator Installation



Follow these steps to prepare the simulation environment:


1. **Download** the compressed file (.zip) of the simulator for Linux. 
* ⬇️ [Download AutoDRIVE Simulator Linux](https://github.com/Tinker-Twins/AutoDRIVE/releases/download/Simulator-0.3.0/AutoDRIVE_Simulator_Linux.zip)

2. **Save** the file in an accessible location and unzip it (e.g., ~/Downloads/AutoDRIVE_Sim).

3. **Open the terminal** and navigate to the folder where you extracted the simulator.

   ```bash

   cd ~/Downloads/AutoDRIVE_Sim

4. Grant execution permissions to the main binary:

      ```bash

   chmod +x "AutoDRIVE Simulator.x86_64"

5. Start the simulator by running the following command:

     ```bash

   ./"AutoDRIVE Simulator.x86_64"


## 🛠️ 2. ROS 2 BRIDGE (DEVKIT)





### 2.1 Workspace and Virtual Environment Setup


To avoid conflicts between the library versions required by the Devkit and the system defaults (Ubuntu 22.04), we will use a Virtual Environment (venv). This isolates the Python dependencies within a controlled environment.



### 2.2 Create Workspace

Run the following commands to create the folder structure:


   ```bash

mkdir -p ~/autodrive_ws/src

cd ~/autodrive_ws

```

### 2.3 Configure the Virtual Environment (venv)

2.3.1 Create and activate the environment inside your workspace root:

   ```bash

# Create the environment

python3 -m venv venv



# Activate the environment (You must do this in every new terminal)
source venv/bin/activate

```

2.3.2 Install Dependencies

With the environment active (venv), install the necessary libraries using these verified versions for stability. This also installs `colcon` **inside the venv** (important — see note below) and `Pillow`, which the bridge imports directly but which is not declared anywhere in the upstream AutoDRIVE devkit:
```bash

pip install --upgrade pip

pip install eventlet==0.33.3 Flask-SocketIO==4.1.0 python-socketio==4.2.0 \

            python-engineio==3.13.0 gevent-websocket==0.10.1 \

            transforms3d attrdict numpy==1.23.5 opencv-contrib-python \

            colcon-common-extensions Pillow

# Flask-SocketIO 4.1.0 needs an old Flask; unpinned, pip resolves to the
# latest Flask/Jinja2, which removed APIs Flask-SocketIO 4.x still uses.
pip install Flask==1.1.1 Jinja2==3.0.3 itsdangerous==2.0.1 Werkzeug==2.0.3 MarkupSafe==2.0.1

```

> ⚠️ **Do NOT create the venv with `--system-site-packages`.** It sounds convenient (it would let the venv "see" system-installed packages), but in practice it silently pulls in whatever versions of `Flask`/`numpy`/etc. your *other* Python projects have installed globally or in `~/.local`, which is exactly the version conflict this venv exists to avoid. A fully isolated venv works fine here: ROS 2 packages (`rclpy`, etc.) are still visible because `source /opt/ros/humble/setup.bash` exports `PYTHONPATH` pointing at them directly — that's independent of the venv's isolation setting.

**Why install `colcon` inside the venv?** If you only `apt install python3-colcon-common-extensions` (the usual way) and then `colcon build` while the venv is active, `colcon` itself still runs under the **system** Python (that's where the `colcon` executable lives), not your venv's Python. The scripts it generates for `autodrive_incoming_bridge` / `autodrive_outgoing_bridge` get hardcoded with the shebang `#!/usr/bin/python3` — the system interpreter — regardless of which venv is "active" in your shell. At runtime those scripts then can't find `attrdict`, `gevent`, etc., because those only exist in the venv's `site-packages`, and a shebang is an absolute path that ignores `$PATH`/venv activation entirely. Installing `colcon-common-extensions` **inside** the venv (as done above) makes `colcon build` run with the venv's Python, so the generated scripts get the correct shebang (`.../venv/bin/python3`) from the start.

2.3.3 Exclude the venv from colcon's build scan

`colcon build` recursively scans the whole workspace for packages, including `venv/`. Some libraries (e.g. `setuptools`) ship internal test fixtures with their own `setup.py`, which colcon can mistakenly try to build as a real package. Mark the venv as ignored:
```bash

touch ~/autodrive_ws/venv/COLCON_IGNORE

```

2.3.4 Critical Compatibility Patch


The attrdict library has incompatibilities with collection modules in Python 3.10+. To fix the import error without affecting the global system, apply this patch directly to the files inside your venv:
```bash

# Fix Mapping import error in attrdict
find ~/autodrive_ws/venv/lib/python3.10/site-packages/attrdict/ -name "*.py" -exec sed -i 's/from collections import Mapping/from collections.abc import Mapping/g' {} +



# Fix multiple import variants
find ~/autodrive_ws/venv/lib/python3.10/site-packages/attrdict/ -name "*.py" -exec sed -i 's/from collections import Mapping, MutableMapping, Sequence/from collections.abc import Mapping, MutableMapping, Sequence/g' {} +



```







### 2.4 Cloning and Organizing the Workspace



To ensure ROS 2 compiles correctly, we must extract only the necessary metapackage from the official repository and remove redundant files.


### Cloning the Repository:

Navigate to your workspace source folder and clone the Devkit:


```bash

cd ~/autodrive_ws/src

git clone --single-branch --branch AutoDRIVE-Devkit [https://github.com/Tinker-Twins/AutoDRIVE.git](https://github.com/Tinker-Twins/AutoDRIVE.git)

```



Extraction and Cleanup:


To avoid duplicate package errors during compilation, move the ROS 2 folder to the src root and delete the rest of the repository:
```bash

# Move the ROS 2 metapackage
mv ~/autodrive_ws/src/AutoDRIVE/ADSS\ Toolkit/autodrive_ros2 ~/autodrive_ws/src/



# Delete the original repository to avoid conflicts
rm -rf ~/autodrive_ws/src/AutoDRIVE

```



### 2.5 Compiling the Workspace



Once the src folder is organized, proceed to build the packages. Make sure the venv is active — this ensures `colcon` (installed in step 2.3.2, inside the venv) is the one that runs, so the compiled scripts point to the venv's Python instead of the system's.
```bash



# Return to workspace root
cd ~/autodrive_ws



# Activate environment and load ROS 2
source venv/bin/activate

source /opt/ros/humble/setup.bash



# Compile
colcon build --symlink-install



# Source the newly compiled workspace configuration
source install/setup.bash

```

**Sanity check:** confirm the build actually picked up the venv's Python before moving on:
```bash

head -1 ~/autodrive_ws/install/autodrive_f1tenth/lib/autodrive_f1tenth/autodrive_incoming_bridge

```
This should print `#!/home/<you>/autodrive_ws/venv/bin/python3`. If it instead prints `#!/usr/bin/python3`, `colcon` ran with the system Python — re-check that `colcon-common-extensions` is installed in the venv (step 2.3.2) and that the venv is active (`which colcon` should point inside `~/autodrive_ws/venv/bin/`), then remove stale artifacts and rebuild: `rm -rf ~/autodrive_ws/{build,install,log} && colcon build --symlink-install`.





## 🚀 3. Execution and Teleoperation



To start the system, follow this order strictly. We recommend opening independent terminals for each process.


### Step 1: Start the Communication Bridge
You have two modes depending on whether you need sensor visualization or want to save system resources:


**Option A: Bridge with Visualization (RViz)**
Use this to see the LiDAR, camera feed, and the vehicle's 3D model while driving.
```bash

cd ~/autodrive_ws

source venv/bin/activate

source /opt/ros/humble/setup.bash

source install/setup.bash

export PYTHONUNBUFFERED=1

# Launches the bridge and opens RViz automatically
ros2 launch autodrive_f1tenth simulator_bringup_rviz.launch.py

```

**Option B: Simple Bridge (Headless)**
Use this if you only want to teleoperate the car without the graphical overhead of RViz.
```bash



cd ~/autodrive_ws

source venv/bin/activate

source /opt/ros/humble/setup.bash

source install/setup.bash

export PYTHONUNBUFFERED=1

# Launches only the communication bridge
ros2 launch autodrive_f1tenth simulator_bringup_headless.launch.py

```



### Step 2: Run the Simulator



Open a new terminal and launch the simulator:
```bash

cd ~/Downloads/AutoDRIVE_Sim  # Adjust path to your folder

./"AutoDRIVE Simulator.x86_64"

  ```

Actions in the simulator:


* Click the antenna button (Disconnected) until it changes to Connected.

* Verify that the driving mode is set to Autonomous.



### Step 3: Keyboard Control (Teleoperation)



In a new terminal, run the node to control the car using keys:
```bash

cd ~/autodrive_ws

source venv/bin/activate

source /opt/ros/humble/setup.bash

source install/setup.bash

ros2 run autodrive_f1tenth teleop_keyboard

```

Controls: W (Accelerate), S (Reverse), A (Left), D (Right), X (Brake).


Note: You must keep this terminal window selected/focused for the vehicle to respond to keystrokes.

# 🤝 Contributing
Contributions are welcome! If you have a suggestion that would make this better, please fork the repo and create a pull request.

✉️ Contact
Hector La Mota - [LinkedIn](https://www.linkedin.com/in/h%C3%A9ctor-la-mota-5a9b7a2a8/)
Carolina Sanchez
### SLIDES :https://docs.google.com/presentation/d/1Yl-QCOvlGdIBSREwMPxfj50a_lk7E9fm8FWiSNKZ6dg/edit?slide=id.p3#slide=id.p3
**AIROS - ESPOL**
