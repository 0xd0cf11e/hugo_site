---
author: "Suweera De Souza"
title: "Debug IDA Pro plugins"
date: "2020-04-26"
description: "Debugging IDA Pro python plugins with VSCode"
tags:
- IDA
- python
- vscode
categories:
- IDA
- vscode
---

## **Table of Contents**
[Requirements](#requirements)

[Overview](#overview)

[Setup](#setup)

1. [Create IDA python plugin to enable debugging](#create-ida-python-plugin-to-enable-debugging)
2. [Configure vscode](#configure-vscode)
3. [Start Debugging!](#start-debugging)

[Resources](#resources)

---

### Requirements

- IDA Pro 7.4+ (with python)
- Visual Studio Code (debugger IDE)
- [ptvsd](https://pypi.org/project/ptvsd/)

# Overview

If vscode is your go to IDE for everything python, then you're probably aware of the module `ptvsd`. It enables debugging python scripts being run on another process by attaching to the Visual Studio debug server.


---

<p>&nbsp;</p>

## Setup

<p>&nbsp;</p>

### Create IDA python plugin to enable debugging

To enable vscode to debug your python plugin, it needs to run in the same directory as that of the plugin. This is the same directory where the plugins are stored for IDA.

`~/idapro-7.x/plugins/`

There are different ways of going about with this. 

What I found easier was to create an IDA python plugin that will enable the debugger, i.e. establish connections for the debug server to attach to.

Below is the plugin script. Save it under the IDA plugins directory. I've named it as `ptvsd_enable.py` with plugin name set to `PTVSD`.

```python
import idaapi
import ptvsd

class DebugPlugin(idaapi.plugin_t):
    # These parameters can be modified as needed.
    # - I've set the flag to PLUGIN_FIX to make sure its in memory when IDA starts.
    # - A hotkey may come in handy if you debug often
    flags = idaapi.PLUGIN_FIX
    comment = "PTVSD Debug Enable"
    help = "Enable debugging using PTVSD"
    wanted_name = "PTVSD"
    wanted_hotkey = ""

    def init(self):
        return idaapi.PLUGIN_KEEP

    def term(self):
        pass

    # The debuggee connection is initialized only when the plugin gets run
    def run(self, arg):
        # Enable the debugger. Raises exception if called more than once.
        try:
            ptvsd.enable_attach()
            # For specifying own address, port, log directory
            # - The default host:port that vscode listens to is localhost:5678
            # - This can be modified as needed; especially if running IDA on a remote server
            #ptvsd.enable_attach(address=('localhost', 5678), log_dir="logs/")
            print("ptvsd debugger enabled")
        except Exception as err:
            print(err)

def PLUGIN_ENTRY():
    return DebugPlugin()
```

Check [resources](#resources) for more info on writing scriptable IDA plugins.

<p>&nbsp;</p>

### Configure vscode

Next is to configure the `launch.json` file in vscode to enable debugging the python plugin scripts.

Below are the configuration settings to set:

```json
{
    "configurations": [
        {
            // Set name as desired; especially if you have a few different configurations
            "name": "Python Attach: IDA python plugin",
            "type": "python",
            "request": "attach",
            // host and port should match that which was set in the PTVSD plugin
            "host": "localhost", // default host
            "port": 5678, // default port
            "pathMappings": [
                {
                    "localRoot": "/path-to/idapro-7.x/plugins/",   // Local root  (where source and debugger running)
                    "remoteRoot": "/path-to/idapro-7.x/plugins/"   // Remote root (where remote code is running)
                }
            ]
        }
    ]
}
```

Check [resources](#resources) for more info on debugging python in vscode

<p>&nbsp;</p>

### Preparing your IDA Python Plugin

To ease the debugging of your plugin, the flag `PLUGIN_UNL` needs to be set. This enables IDA to reload the plugin every time it is run, espcially after making changes to the script and without having to restart IDA everytime.

And finally to call `breakpoint()` in your script where you want the debugger to pause. I was unable to get the debugger to pause at a toggled set breakpoint, and this call was able to set it for me. It needs to get once and from there toggle set breakpoints should work.

Below is a sample python plugin to give an example. I've also saved it as `sample_debuggee.py` under the IDA plugins folder.

```python
import idaapi

def my_debugged_function():
    print("This")
    print("is cool")
    print(" :) ")
    print(" ~The End~")

class SamplePlugin(idaapi.plugin_t):
    # Temporarily set this flag to enable reloading of script
    flags = idaapi.PLUGIN_UNL
    comment = "Sample Debuggee"
    help = "Sample Debuggee"
    wanted_name = "Sample Debuggee"
    wanted_hotkey = ""

    def init(self):
        return idaapi.PLUGIN_KEEP

    def term(self):
        pass

    def run(self, arg):
        # The debugger should pause here.
        # This call can be used anywhere you wish the debugger to pause at
        breakpoint()
        my_debugged_function()

def PLUGIN_ENTRY():
    return SamplePlugin()
```

<p>&nbsp;</p>

## Start Debugging

1) After the plugins are created in the IDA plugins directory and the python debug configuration is set, close any running of instance of IDA and start it up again.<p>&nbsp;</p>

2) Open up a new file or database and run the `PTVSD` plugin. I've made the script to print `ptvsd debugger enabled` if all went well (see below).

<img src="/debug_ida_plugin/ptvsd_connection_set.png" width="1000">
<p>&nbsp;</p>

3) Now run the debug configuration in vscode. Running the vscode debugger first would retun an error since it is attempting to listen for an existing connection at the given host and port. 

<img src="/debug_ida_plugin/run_python_attach.png" width="500">

You should also see this debug menu appear (see below) if the debugger was able to successfully attach to our `PTVSD` plugin.

<img src="/debug_ida_plugin/debug_menu_bar.png" width="300">
<p>&nbsp;</p>

4) *OPTIONAL:*  Run `netstat | grep 5678`. Grep on the port you've provided. This will show if any connections got establised on the host:port we have assigned for debugging. Which would mean you're all set :)

<img src="/debug_ida_plugin/netstat.png" width="700">
<p>&nbsp;</p>

5) Finally, to start debugging your plugin, run it from IDA (after having set `breakpoint()` to where you'd like the debugger to pause).

You should see the debugger pauses as well as the debug menu bar enabling its debug commands. The debugger paused on line 26 on our sample plugin script as seen below:

<img src="/debug_ida_plugin/debugger_paused.png" width="700">
<p>&nbsp;</p>

6) The debugger can be disconnected at anytime. Running `netstat | grep 5678`, we see our `PTVSD` plugin wait for a connection to be established at the port 5678 (see below). This connection listens for a while before closing it upon which the `PTVSD` plugin would have to rerun to start another debugging session.

<img src="/debug_ida_plugin/netstat_wait.png" width="700">
<p>&nbsp;</p>

---

### Resources

IDA Python Plugins

 - Scriptable IDA plugins - https://www.hex-rays.com/blog/scriptable-plugins/
 - Description of different flags used for building IDA Plugins - https://www.hex-rays.com/products/ida/support/sdkdoc/group___p_l_u_g_i_n__.html

VSCode Python

- Python debug configurations - https://code.visualstudio.com/docs/python/debugging