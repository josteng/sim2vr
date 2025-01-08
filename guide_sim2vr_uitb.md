# Guide: Training Biomechanical Models with SIM2VR and User-in-the-Box

This guide explains the process of training a biomechanical model on a Unity task using [SIM2VR](https://github.com/fl0fischer/sim2vr) and [User-in-the-Box](https://github.com/aikkala/user-in-the-box) (uitb).

The guide is divided into three main sections:
- [I. Unity Task Setup](#i-unity-task-setup)
- [II. Python Project Setup and User-in-the-Box Task Creation](#ii-python-project-setup-and-user-in-the-box-task-creation)
- [III. Training and Evaluation of the Simulated User](#iii-training-and-evaluation-of-the-simulated-user)

In the following, unless otherwise stated, it is assumed that you are working on a Linux machine. If you are using a different operating system, you may need to adapt some of the steps.


## I. Unity Task Setup

The first step is to create a Unity environment. Note that in this example we are using Unity 6 and a newly created project. If you are using a different version of Unity, you may need to adapt some of the steps. If you are using an existing project, check this section to ensure that all the necessary components are included.

1. Create a **new Unity project**
    1.  Make sure you have the [Unity Hub](https://unity.com/unity-hub) installed.
    2. To install Unity 6, open the Unity Hub, go to `Installs`, click on `Install Editor`, and select the latest version of Unity 6 (this guide has been tested with version 6000.0.25f1, but there should be no difference to other versions of Unity 6). Accept the defaults.
    3. In `Projects`, click on `New Project` and create a new project using the **Universal 3D** template. Click on `Create` and wait for the project to open.

2. Add **OpenXR** support
    1. Go to `Edit` -> `Project Settings` -> `XR Plugin Management`.
    2. Click on `Install XR Plugin Management`.
    3. Select `OpenXR` and click on `Install`.
    4. In the XR Plugin Management settings, do ***not*** enable the OpenXR plug-in provider. This would cause problems later when building the project for Linux (specifically an error "The only standalone targets supported are Windows x64 and OSX with OpenXR. [...]")
    If you also want to build the project for VR devices, only enable the OpenXR plug-in provider when building for VR devices.

3. Install the **XR Interaction Toolkit**
    1. Go to `Window` -> `Package Manager`.
    2. Select `Unity Registry` from the left hand sidebar.
    3. Locate the `XR Interaction Toolkit` and click `Install`.
    4. Once installed, click on `Samples` in the XR Interaction Toolkit package in the Package Manager.
        1. Import the `Starter Assets` by clicking on `Import`.
        2. _[Optional]_ You may want to import the VR simulator to test the VR environment without needing a physical VR device. For example, you can use the simulator to simulate controller inputs. To add it to your project, import the `XR Device Simulator` by clicking on `Import`.

4. Add an **XR Origin**
    1. Remove the `Main Camera` from the default scene by right clicking on it in the hierarchy and selecting `Delete`. The XR Origin will contain a camera.
    2. In the Project window, go to `Assets` -> `Samples` -> `XR Interaction Toolkit` -> `<version of toolkit>` ->`Starter Assets` -> `Prefabs`.
    3. Drag the `XR Origin (XR Rig)` prefab into the hierarchy.
    4. Click on the newly created `XR Origin (XR Rig)` object to open the inspector. There, in the `XR Origin` component, set the `Tracking Origin Mode` to `Device`. This will make the XR Origin use the tracking origin of the device (e.g., the headset). For alternatives and more information, also see [here](https://docs.unity3d.com/Packages/com.unity.xr.core-utils@2.5/api/Unity.XR.CoreUtils.XROrigin.TrackingOriginMode.html).
    5. > _Note:_ The created XR Origin includes a camera with controllers as well as default controls for locomotion. Alternatively, we could have added a plain XR Origin by right-clicking in the hierarchy and selecting `XR` -> `XR Origin VR`. This would only create an Origin with a camera and an empty XR Interaction Manager. Additional controls would have to be added manually.

5. _[Optional]_ Add an **XR Device Simulator**
    1. > _Note:_ This requires the XR Device Simulator to be installed in step I.3.iv.
    2. In the Project window, go to `Assets` -> `Samples` -> `XR Interaction Toolkit` -> `<version of toolkit>` ->`XR Device Simulator`.
    3. Drag the `XR Device Simulator` prefab into the hierarchy.
    4. _[Verification]_ When opening the newly added XR Device Simulator, you should see the `Global Actions`, `Controller Actions`, and `Hand Actions` automatically mapped to the respective XR device objects.

6. _[Optional]_ Add some **sample objects** and **run the simulations**
    1. To check that the setup is working and that, for example, the camera is set up correctly, you can add some sample objects to the scene. For example, you can add a cube by right-clicking in the hierarchy and selecting `3D Object` -> `Cube`.
    2. To make sure the objects are visible to the camera, you can click on the camera icon in the scene view to see the camera's view.
    3. Click the play button (`>` icon) to start the simulation. You can use the XR Device Simulator to simulate the VR environment (instructions are in the bottom left corner of the game view). Make sure that the objects are visible without the need to move the camera.
    4. If everything works as expected, you can stop the simulation by clicking the play button again.

7. Add the **SIM2VR** framework
    1. Download the [SIM2VR](https://github.com/fl0fischer/sim2vr) repository. E.g. by cloning it with `git clone https://github.com/fl0fischer/sim2vr.git`.
    2. Go to `Assets` (menu entry, not in the project window) -> `Import Package` -> `Custom Package...`.
    3. Select the file `sim2vr.unitypackage` (located in the root folder of the downloaded repository) and click on `Open`.
    4. In the `Import Unity Package` window, leave everything selected and click on `Import`.
    5. In the project window, go to `Assets` -> `sim2vr` -> `Prefabs` and drag the `sim2vr` prefab into the hierarchy.
    6. If you get the error `Assets/sim2vr/Scripts/ZmqServer.cs(3,7): error CS0246: The type or namespace name 'Newtonsoft' could not be found (are you missing a using directive or an assembly reference?)`, the Newtonsoft.Json package required by the SIM2VR framework is missing.
    To fix this:
        1. Go to `Window` -> `Package Manager`.
        2. Click on the `+` symbol in the top left corner and select `Add package from git URL`.
        3. Enter `com.unity.nuget.newtonsoft-json` and click on `Install`. (The latest version should be selected automatically).
        4. > _Note:_ For alternatives to resolve this error, see [here](https://stackoverflow.com/a/75907570/13814816).

8. Create a **custom RLEnv** class (containing logic such as reward calculation, task completion, etc.)
    1. Open the project window and go to `Assets`.
    2. In this folder (or in a custom subfolder if you wish) create a new script. To do this, right-click on or within the folder and select `Create` -> `Scripting` -> `MonoBehaviourScript`. Give the file a name of your choice, we will use `RLEnvCustom` in this example.
    3. Double click on the newly created script to open it in the editor.
    4. You can remove the given `Start` and `Update` methods as they are not needed for the `RLEnv` class.
    5. The class currently extends the `MonoBehaviour` class (`public class RLEnvCustom : MonoBehaviour`). Change this to extend the `RLEnv` class instead (`public class RLEnvCustom : RLEnv`).
    6. You will see an error that some abstract inherited members are not implemented. Your editor will suggest that you implement these members. Accept the suggestion to implement all of the members.
    7. For now, these methods can be left empty. Make sure to remove the default `throw new System.NotImplementedException();` line in each method. For `GetTimeFeature`, you can return a constant value, e.g. `return 0f;`.
    8. > _Note:_ Values such as the reward or whether the task has been completed, are not returned by the methods but set to members in the `RLEnv` class. See the `RLEnv` class for more information.
    9. Save the script and return to the Unity editor.

9. **Configure the `RLEnv` object** inside the `sim2vr` object (this must be done before configuring the `SimulatedUser` object)
    1. Open the `RLEnv` object within the `sim2vr` object in the hierarchy (you need the inspector view of this object on the right).
    2. Click on `Add Component` and search for the `RLEnvCustom` script you created in the previous step. Add this script to the object by clicking on its name. CAUTION: The script name can contain spaces here, e.g. `RL Env Custom`.
    3. Configure the `RL Env Custom` script by filling in the empty fields. You can use the icon with a small circle inside a larger circle to get a drop down menu with appropriate options.
        - Set `Simulated User` to `SimulatedUser` (comes with the `sim2vr` prefab)
        - Set `Logger` to `Logger` (comes with the `sim2vr` prefab). The Logger object needs no further configuration unless you want to change the logging behavior.

10. **Configure the `SimulatedUser`** object within the `sim2vr` object
    1. Open the `SimulatedUser` object within the `sim2vr` object in the hierarchy (you need the inspector view of this object on the right).
    2. Set the following fields:
        - Set `Left Hand Controller` to `Left Controller` (will be shown as `Left Controller (Transform)` after selection)
        - Set `Right Hand Controller` to `Right Controller` (will be shown as `Right Controller (Transform)` when selected)
        - Set `Main Camera` to `Main Camera` (the camera that is part of the XR Origin, make sure you have deleted the default camera before, see step I.4.i)
        - Set `RLEnv` to `RLEnv` (the `RLEnv` object you configured in the previous step). If this option is grayed out, you have not configured the `RLEnv` object correctly. See steps I.7 and I.8.

11. Add a **Recorder** object
    1. > _Note:_ To generate recordings of the Unity task during training later on, we need a Recorder object. This object will record the state of the task at each step. If you do not create it now, you may encounter errors later.
    2. In the hierarchy, right click on the `sim2vr` object and select `Create Empty`. Name this object `Recorder`.
    3. Select the `Recorder` object and in the inspector, add a `Camera` component to it using the `Add Component` button. This camera will be used to render the environment (third person view) for the recordings.
    4. In the recorder, you may want to reduce the camera's `Clipping Planes` to a smaller value, e.g. 0.01 and 10 to speed up the rendering process.
    5. Now click on `Add Component` again and add the `Recorder`script (this is included in the `sim2vr` package).
    6. In the `Recorder (Script)` component, set `Simulated User`to `SimulatedUser`.

12. **Build** the project
    1. Go to `File` -> `Build Profiles`.
    2. Select `Linux` as the target platform. (We assume that the training will be done on a Linux machine, but you can choose another platform if you prefer.)
    3. If this is your first time exporting to Linux, you can click on `Install with Unity` to install the necessary components. Leave `Linux Build Support (Mono)` selected and click on `Install`.
    4. Back in the `Build Profiles` window, click on `Switch Platform` to switch to the Linux platform.
    5. You may see the message "Cannot build player while editor is importing assets or compiling scripts." Just wait for the process to finish, even if there is no progress bar.
    6. Click on `Build` and select a folder where you want to save the build.
        - Do not choose the root folder of the project, but rather create an empty folder that contains only the build files, to make it clear what files are output from the build process. For example, just call this folder `Build`.
        - Choose any name for the output, we will use `UnitySim2VrDemo`here.
        - Click on `Save` to start the build process.
        - When you are prompted "Scene(s) Have Been Modified", click on `Save`.
    7. The build process will take some time. You can continue with the next steps while the build is running, as we will not need the build files immediately.

This completes the setup of the Unity environment, next we will set up the Python project to train the biomechanical model.


## II. Python Project Setup and User-in-the-Box Task Creation
The second step is to create the User-in-the-Box task. This task will connect the Unity environment you just created to the User-in-the-Box framework.

1. _[Optional]_ Select and set up the **Runtime Environment**
    1. You can run the User-in-the-Box framework on your local machine or on a remote server.
    2. If you are using a local installation, skip to the next step.
    3. For remote configuration, consider using Visual Studio Code with Remote Tunnels instead of PyCharm with remote environments, as the latter struggles to synchronize large file sets properly. A detailed guide to VS Code Remote Tunnels can be found [here](https://code.visualstudio.com/docs/remote/tunnels).
        > _Note:_ You may want to clone the repository first (step II.2) and only then set up the remote environment.
        > _Note:_ You can keep the VS Code Tunnel running (so you do not have to start it every time) by using `code tunnel service install`. To stop the service, use `code tunnel service uninstall`.

2. Set up the **User-in-the-Box** framework
    1. We will add the new task directly to the User-in-the-Box repository. You can either use the original repository or a fork.
        - To use the original [User-in-the-Box](https://github.com/aikkala/user-in-the-box) repository, clone it using `git clone https://github.com/aikkala/user-in-the-box.git`.
        - Forking is not explained here, but you can find information on how to fork a repository [here](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/fork-a-repo).
    2. Open the User-in-the-Box repository in your IDE.

3. Initialize the **Python** environment
    1. Make sure you have [Python](https://www.python.org) installed. This tutorial is tested with **Python 3.10**.
    2. Create a new virtual environment:
        ```bash
        python3 -m venv venv
        ```
    3. Activate the virtual environment:
        ```bash
        source venv/bin/activate
        ```
    4. Install the required packages:
        ```bash
        pip install -e .
        ```

4. Set **environment variables**
    1. On Linux (and macOS) we have to set the `MUJOCO_GL` runtime variable. This is needed for the underlying MuJoCo physics engine to use the correct OpenGL backend. We assume Linux, where the value is `egl`, for macOS it would be `cgl`. You can either do this every time before you run the User-in-the-Box framework or set it as a permanent environment variable.
        - To set the variable for the current session only, you can use the following command:
            ```bash
            export MUJOCO_GL=egl
            ```
        - If you want to set it permanently, you can add the line to your `.bashrc` or `.bash_profile` file. We assume bash here, for other shells, the file may be different. Use the following command to add the line to your `.bashrc` file:
            ```bash
            echo "export MUJOCO_GL=egl" >> ~/.bashrc
            ```
            and then use `source ~/.bashrc` to apply the changes. Also, you need to run `source venv/bin/activate` again to reactivate the virtual environment.
    2. > _Note:_ If you encounter problems with the shell output later on, you may also want to set `PYTHONUNBUFFERED=1` as an environment variable. This will prevent the output from being buffered. You can set it in the same way as the `MUJOCO_GL` variable.

5. Add the **Unity simulation** to the User-in-the-Box framework
    1. Go to `uitb/tasks/unity/apps` in the User-in-the-Box repository.
    2. Create a new folder for your task. You can name it whatever you want, we will use `unity-sim2vr-demo-linux` here. The folder will go right next to the existing `beats-vr-linux` and `whac-a-mole-linux` folders.
    3. Copy ***all*** files from the Unity build output folder (see step I.12) into this newly created folder. The only folder you can ignore is the `..._DoNotShip` folder.
    4. Make the `...x86_64` file executable (in our example this is called `UnitySim2VrDemo.x86_64`). You can do this by running the following command in the folder:
        ```bash
        chmod +x UnitySim2VrDemo.x86_64
        ```

6. Create the **configuration** for the task
    1. Go to `uitb/configs` in the User-in-the-Box repository.
    2. We will use the `mobl_arms_beatsvr_bimanual.yaml` file as a template. Copy this file and rename it to your task, e.g. `mobl_arms_unity_sim2vr_demo.yaml`.
    3. In the copied file, make the following changes:
        - At the very top, replace the `simulation_name` field (set to `beatsvr_neural_1e3`) with the name of your task, e.g. `unity_sim2vr_demo`.
        - Look for the `unity_executable` field and set it to the name of the executable you made executable in the previous step. In our case, this is `apps/unity-sim2vr-demo-linux/UnitySim2VrDemo.x86_64`. The path should always start with the `apps` folder.
        - Additional changes to the configuration file can be made as needed.

Now we have all the necessary components set up for the User-in-the-Box task. Next, we will train and evaluate the biomechanical model on the Unity task.


# III. Training and Evaluation of the Simulated User
The third step is to train and evaluate the biomechanical model on the Unity task. This is done using the User-in-the-Box framework.

We will first build the actual task from the configuration, then generate an evaluation based on random actions, and finally train the model. In practice, you can skip the building step later if you want to train the model directly. Here we do the building first to be able to evaluate the task with random actions before training the model.

1. [_Prerequisites_]: Make sure you have loaded the Python environment and set the environment variables as described in II.3 and II.4.

2. **Build the task**
    1. > _Note:_ User-in-the-box does not currently provide a standalone build script separate from the training. But this is quite easy to do.
    2. Somewhere, e.g. in `uitb/test`, create a new Python file, e.g. `builder.py` with the following content:
        ```python
        import sys

        from uitb.simulator import Simulator

        if __name__ == "__main__":
            assert len(sys.argv) == 2, "Script takes only one argument: the path to the config file."
            config_file_path = sys.argv[1]

            simulator_folder = Simulator.build(config_file_path)
        ```
    3. Run the script with the path to the configuration file as an argument. For example, from the project root:
        ```bash
        python3 uitb/test/builder.py uitb/configs/mobl_arms_unity_sim2vr_demo.yaml
        ```
    4. > _Note:_ This will create a built task in the folder `simulators/unity_sim2vr_demo` (you defined the name with the simulation field in the configuration file, see II.6.3).

3. Create a **virtual display**
    1. > _Note:_ To evaluate and train the task on a remote server, you need a virtual display. This is used to render the Unity task, which is then used by the User-in-the-Box framework. You can skip this if you are using your local machine.
    2. If not already installed, install the `Xvfb` package. You can do this with the following command:
        ```bash
        sudo apt-get install xvfb
        ```
    3. Set the display to use with
        ```bash
        export DISPLAY=:1
        ```
        The number can be any number, but it should not be 0, as this is the default display. If you want to run multiple displays for different evaluations, you can use different numbers.
    4. Start a virtual display with
        ```bash
        xdpyinfo -display $DISPLAY > /dev/null || Xvfb $DISPLAY -screen 0 1920x1090x24 &
        ```
        If you see a warning like "xdpyinfo:  unable to open display ":1", you can just re-execute this command. This warning should only appear the first time you start the display. The resolution is set to be compatible with the user-in-the-box / SIM2VR framework.
    5. Tips & Troubleshooting
        - The virtual displays run as processes, you can get a list of running displays with `ps -ef|grep Xvfb`. Note that the last process is the `grep` process to query the list, so you can ignore this. The PIDs (process IDs) are in the second column.
        - If you want to stop a display, you can kill the process with `kill <PID>`, where `<PID>` is the process ID of the display.

4. **Evaluate** the performance of a simulated user
    1. > _Note:_ If no training has been performed yet, the model will perform random actions. This is useful to see if the task is set up correctly and the model can interact with the task.
    2. For the video recordings, you have to install the `ffmpeg` package. You can do this with the following command:
        ```bash
        sudo apt-get install ffmpeg
        ```
    3. Make sure the virtual display is running (see step III.3)!
    4. To evaluate the task we can use the `evaluator.py` script that comes with the User-in-the-Box framework. This script runs the task either with the trained model or with random actions if the model has not been trained yet (which is the case here). It generates logs as well as videos (with the `--record`flag). You can run it like this:
        ```bash
        python3 uitb/test/evaluator.py simulators/unity_sim2vr_demo --num_episodes 1 --logging --action_sample_freq 100 --record
        ```
        - `simulators/unity_sim2vr_demo` is the path to the built task. Make sure you pass the path to the built task here, not the configuration file here.
        - `--num_episodes 1` is the number of episodes to run. You can increase this number to get more data.
        - `--logging` enables logging of the results.
        - `--action_sample_freq 100` specifies how often the actions are sampled. In this case, every 100th step.
        - `--record` enables recording of the task. This will create a video of the task. This uses the Recorder object we created in the Unity task in step I.11.
    5. Generation takes some time (about 30 seconds to two minutes per episode). There is currently no progress output until the final videos are rendered. You can check your CPU usage to see if the task is still running (e.g. with `htop`). If you encounter an error `File "_zmq.py", line 160, in zmq.backend.cython._zmq._check_rc [...]` after waiting a long time and canceling the task with CTRL + C, this typically indicates that uitb failed to connect with the virtual display. In this case, re-running the display setup commands from step III.3 usually resolves the issue.

    6. The resulting files are stored in the `simulators/<your_task>/evaluate` folder. There you will find the log files as well as the video recordings there. If you have evaluated on a remote server, you can copy these files to your local machine using `scp`. For example:
        ```bash
        scp -r <user>@<remote>:</path/to/uitb/simulators/unity_sim2vr_demo/evaluate> <local_directory>
        ```
    Here, you would replace `<user>` with your remote user, `<server>` with the server, and `/path/to/uitb/simulators/unity_sim2vr_demo/evaluate` with the path to the `evaluate` folder on the remote server. The `<local_directory>` is the directory where the files should be copied to on your local machine. To use the current directory, you can just use `.` for the local directory.

5. **Train** the simulated user
    1. It is recommended that you set up Weights & Biases (wandb) to log the training process. For example, you can view the training progress in the Weights & Biases dashboard. Wandb is already included in the User-in-the-Box framework as a dependency. You can login in with the following command:
        ```bash
        wandb login
        ```
        Depending on whether you have a local or remote setup, you will either get a link to log in on your local machine or you will have to copy the link to your local machine to log in. After logging in and later on starting a training, you can see your training progress on Wandb, e.g. on your [Weights & Biases home page](https://wandb.ai/home). If you prefer not to use Wandb, you can skip this step or run `wandb offline` to use local logging only.
    2. Make sure the virtual display is running (see step III.3)!
    3. _[Optional]_ You can use `tmux` (installed with `sudo apt-get install tmux`) to run the training in a separate terminal. This way, you can close the terminal and the training will still run.
        - You can start a new tmux session with just `tmux` or by giving it a name `tmux new -s uitb_training`.
        - After that you would start the training as described in the next step.
        - To detach from a tmux session, you can use `CTRL + b` and then `d`. 
        - To attach to a session again, you can use `tmux attach` to attach to the last session or `tmux attach -t uitb_training` to attach to a specific session.
    4. To train the task, we can use the `trainer.py` script that comes with the User-in-the-Box framework. This script will train the model on the task. You can run it like this:
        ```bash
        python3 uitb/train/trainer.py uitb/configs/mobl_arms_unity_sim2vr_demo.yaml
        ```
        Note that we use the configuration file here. The script will automatically build the task and start the training (overwriting the previously built task). So in general, you can skip the direct build in step 2 if you want to train the model directly.
    5. The results are stored in `PPO_{x}` folders inside the generated task folder. The `x` is a number that increases with each training run. To evaluate the trained simulated user, see step 4.

You have now trained and evaluated a biomechanical model on a Unity task using the User-in-the-Box and SIM2VR frameworks. Next, you can customize the task, the biomechanical model, or the training parameters. You can use the above steps to train and evaluate the model again.