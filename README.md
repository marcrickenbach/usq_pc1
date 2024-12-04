# USQ PC-1

PC-1 is the first program card for the USQ system. At base
it is a two-channel sequencer.


## VSCode Devcontainer
This project makes use of a Docker Devcontainer for development. 
Below are the steps for cloning this repository and launching
it within the container in VSCode. 

### Building the Container
Ensure that VSCode and Docker are installed and running. 

In the VSCode Command Palette (CMD + SHIFT + P), select:
Dev Containers: Clone Repository in Container Volume... 
and enter the repository's address: 
https://github.com/marcrickenbach/usq_pc1.git

This will create a container with the name usq_pc1 as well
as a Docker volume with a similar name. The container will mount
to /workspaces.

As the Docker image will pull the Zephyr repo to the container
along with other necessary tools, this initial process will take
some time. 

### Building Zephyr and its tools
Open a VSCode terminal. You should be at /workspaces/usq_pc1,
which is inside the cloned repository. From here, run the following 
commands: 
```
$ sudo chown -R $(whoami):$(whoami) /workspaces
$ west init -l
$ west update
```
The first command changes permissions to the user so that we can 
run the west init command in the workspaces directory. 

The -l flag will create a new west workspace using the local west.yml
file at the root of the project directory. This west manifest file
tells the build system what modules we need to build. The west update
command will clone the main revision of Zephyr to the ../zephyr directory
as well as the specified modules to ../modules and create ../west to 
track everything. The idea here is that Zephyr and all of its modules
remain outside of the application repository. 

The Zephyr devcontainer image pre-configures the ZEPHYR_BASE environmental
variable to /workdir/zephyr, which we change in our devcontainer.json to:
/workspaces/zephyr. If for some reason this hasn't been automatically
changed, we can manually change this by running the following command:
```
$ export ZEPHYR_BASE=/workspaces/zephyr
```

### Building PC-1

You should now be able to build the USQ_PC1 project by running the 
following command: 
```
# For a pristine build:
$ west build -p always -b fkmg_pc1
```
```
# For a normal build
$ west build -b fkmg_pc1
```


## Project Outline

APPLICATION LAYER:

Overseeing layer that maintains all threads. This is the
only thread able to directly communicate between multiple
lower-level threads. All modes, processing commands are
dispatched from here. 

    A. VCA Processing (AO)
    Active object layer that receives information, e.g.
    ADC values, scales them or applies other processing. 
    This layer then sends that data to the queue. 
    
        1. i2c write queue (AO)
           An active object layer that handles a queue 
           of processed data to write to the DAC. The
           VCA processing is responsible for processing
           the data to write to the DAC, e.g. for a 
           straight, normal mode, we might take scaled
           ADC CV values, package them and queue them
           for output to the VCA via the DAC. 

            a. DAC7578 i2c driver (Zephyr)
               This layer is thin and interacts directly
               with the hardware. It is based on an
               existing TI DAC driver in Zephyr and
               makes use of the Zephyr API to control
               the I2C Dac. 

    B. ADC Processing (AO)
    Recieves and processes ADC read values and sends
    them to application layer. 

        1. ADC read queue (AO)
        Maintains a queue that handles ADC read timings
        and dispatches these values appropriately. 
        Slight possibility we wont need this layer and
        instead just run the ADC driver off of a timer. 

            a. ADC driver (Zephyr)
            Thin lower-level layer that interacts 
            directly with the ADC peripheral.

## Necessary Modules

Listed in order of importance:
1. Pot Module -- ADC pot readings
2. CV Sampling -- ADC inputs. 
3. LED Driver
   This will be simple as all LEDs are driven directly
   from the microcontroller. We'll just need a wrapper
   layer to map these pins. This only pertains to the
   mode buttons as the channel buttons are directly
   tied to the outputs. 
4. Button Driver
   Only two buttons directly into the mcu, but will need
   to handle double press and long press eventually. 
5. DAC Driver
   The 12-bit dac handles both the VCA voltages and can
   be used to output directly. 

More. 


## Debugging

[TODO] Debugging hasn't yet been tested within this devcontainer. Once that 
is tested and all bugs are figured out, update this section with any steps 
necessary to debug with J-Link.

Install the Cortex-Debug extension in the dev-container. We are debugging using
J-Link's GDB server. Launch the GDB server then start a debugging session in VS
Code.

Here is an example of what the launch.json should look like:

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "J-Link Debug",
            "type": "cortex-debug",
            "request": "launch",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceFolder}/build/zephyr.elf",
            "servertype": "external",
            "gdbTarget": "host.docker.internal:2331",
            "armToolchainPath": "/opt/gcc-arm-none-eabi/bin/",
            "runToEntryPoint": "main",
            "showDevDebugOutput": "raw",
            "preLaunchCommands": [
                "set mem inaccessible-by-default off"
            ],
            "overrideLaunchCommands": [
                "monitor halt",
                "monitor reset",
                "load ${workspaceFolder}/build/zephyr.elf",
            ],
            // "svdFile": "${workspaceFolder}/.vscode/STM32F4x.svd",
        },
        {
            "name": "J-Link Attach",
            "type": "cortex-debug",
            "request": "attach",
            "cwd": "${workspaceFolder}",
            "executable": "${workspaceFolder}/build/zephyr.elf",
            "servertype": "external",
            "gdbTarget": "host.docker.internal:2331",
            "armToolchainPath": "/opt/gcc-arm-none-eabi/bin/",
            // "svdFile": "${workspaceFolder}/.vscode/STM32H750x.svd",
            "showDevDebugOutput": "raw",
        },
    ]
}
```

## Resources

[Zephyr Getting Started](https://docs.zephyrproject.org/latest/develop/getting_started/index.html)
