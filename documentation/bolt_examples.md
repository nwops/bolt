# Bolt examples

Bolt lets you automate almost any task you can think of. These are some of the common scenarios we've come across.

If you'd like to share a real-world use case, reach out to us in the #bolt channel on [Slack](https://slack.puppet.com).

For more usage examples, check out the [Puppet blog](https://puppet.com/blog-tags/puppet-bolt).

## Run a PowerShell script that restarts a service

To show you how you can use Bolt to reuse your existing PowerShell scripts, this guide walks you through running a script with Bolt, and then converting the script to a Bolt task and running that.

**Before you begin**
-   Ensure you’ve already [installed Bolt](bolt_installing.md#) on your Windows machine.
-   Identify a remote Windows target node to work with.
-   Ensure you have Windows credentials for the target node.
-   Ensure you have [configured Windows Remote Management](https://docs.microsoft.com/en-us/windows/desktop/winrm/installation-and-configuration-for-windows-remote-management) on the target node.

The example script, called [restart_service.ps1](https://gist.github.com/RandomNoun7/03dfb910e5d93fefaae6e6c2da625c44#file-restart_service-ps1), performs common task of restarting a service on demand. The process involves these steps:

1.  Run your PowerShell script on a target Windows node.
1.  Create an inventory file to store information about the node.
1.  Convert your script to a task.
1.  Execute your new task.


### 1. Run your PowerShell script on a Windows target node

First, we’ll use Bolt to run the script as-is on a single target node.

1.  Create a Bolt project directory to work in, called `bolt-guide`.
1.  Copy the [`restart_service.ps1`](https://gist.github.com/RandomNoun7/03dfb910e5d93fefaae6e6c2da625c44#file-restart_service-ps1) script into `bolt-guide`.
1.  In the `bolt-guide` directory, run the `restart_service.ps1` script:
    ```
    bolt script run .\restart_service.ps1 W32Time --nodes winrm://<HOSTNAME> -u Administrator -p 
    ```

    ![](restart_service.png)

    **Note:** The `-p` option prompts you to enter a password.

    By running this command, you’ve brought your script under Bolt control and have run it on a remote node. When you ran your script with Bolt, the script was transferred into a temporary directory on the remote node, it ran on that node, and then it was deleted from the node.


### 2. Create an inventory file to store information about your nodes

To run Bolt commands against multiple nodes at once, you need to provide information about the environment by creating an [inventory file](inventory_file.md). The inventory file is a YAML file that contains a list of target nodes and node specific data.

1.  Inside the `bolt-guide` directory, use a text editor to create an `inventory.yaml` file.
1.  Inside the new `inventory.yaml` file, add the following content, listing the fully qualified domain names of the target nodes you want to run the script on, and replacing the credentials in the `winrm` section with those appropriate for your node:
    ```yaml
    groups:
      - name: windows
        nodes:
          - <ADD WINDOWS SERVERS' FQDN>
          - <example.mycompany.com>
        config:
          transport: winrm
          winrm:
            user: Administrator
            Password: <ADD PASSWORD>
    ```

    **Note:** To have Bolt securely prompt for a password, use the `--password` or `-p` flag without supplying any value. This prevents the password from appearing in a process listing or on the console. Alternatively you can use the [``prompt` plugin`](inventory_file_v2.md#) to set configuration values via a prompt.

    You now have an inventory file where you can store information about your nodes.

    You can also configure a variety of options for Bolt in `bolt.yaml`, including global and transport options. For more information, see [Bolt configuration options](bolt_configuration_options.md).


### 3. Convert your script to a Bolt task

To convert the `restart_service.ps1` script to a task, giving you the ability to reuse and share it, create a [task metadata](writing_tasks.md#) file. Task metadata files describe task parameters, validate input, and control how the task runner executes the task.

**Note:** This guide shows you how to convert the script to a task by manually creating the `.ps1` file in a directory called `tasks`. Alternatively, you can use Puppet Development Kit (PDK), to create a new task by using the [`pdk new task` command](https://puppet.com/docs/pdk/1.x/pdk_reference.html#pdk-new-task-command). If you’re going to be creating a lot of tasks, using PDK is worth getting to know. For more information, see the [PDK documentation.](https://puppet.com/docs/pdk/1.x/pdk_overview.html)

1.  In the `bolt-guide` directory, create the following subdirectories:
    ```
    bolt-guide/
    └── modules
        └── gsg
            └── tasks
    ```
1.  Move the `restart_service.ps1` script into the `tasks` directory.
1.  In the `tasks` directory, use your text editor to create a task metadata file — named after the script, but with a `.json` extension, in this example, `restart_service.json`.
1.  Add the following content to the new task metadata file:

    ```json
    {
      "puppet_task_version": 1,
      "supports_noop": false,
      "description": "Stop or restart a service or list of services on a node.",
      "parameters": {
        "service": {
          "description": "The name of the service, or a list of service names to stop.",
          "type": "Variant[Array[String],String]"
        },
        "norestart": {
          "description": "Immediately restart the services after start.",
          "type": "Optional[Boolean]"
        }
      }
    }
    ```

1.  Save the task metadata file and navigate back to the `bolt-guide` directory.

    You now have two files in the `gsg` module’s `tasks` directory: `restart_service.ps1` and `restart_service.json` -- the script is officially converted to a Bolt task. Now that it’s converted, you no longer need to specify the file extension when you call it from a Bolt command.
1.  Validate that Bolt recognizes the script as a task:
    ```
    bolt task show gsg::restart_service
    ```

    ![](bolt_PS_2.png)

    Congratulations! You’ve successfully converted the `restart_service.ps1` script to a Bolt task.

1.  Execute your new task:
    ```
    bolt task run gsg::restart_service service=W32Time --nodes windows
    ```

    ![](bolt_PS_3.png)

    **Note:** `--nodes windows` refers to the name of the group of target nodes that you specified in your inventory file. For more information, see [Specify target nodes](bolt_options.md#).


## Deploy a package with Bolt and Chocolatey

You can use Bolt with Chocolatey to deploy a package on a Windows node by writing a Puppet plan that installs Chocolatey and uses the Chocolatey package provider to install a package. This is all done using content from the Puppet Forge.

**Before you begin** 

- Install Bolt and Puppet Development Kit (PDK).
- Ensure you have Powershell and Windows Remote Management (WinRM) access.

In this example, you:

- Build a project-specific configuration using a Bolt project directory and PDK.
- Download module content from the Puppet Forge.
- Write a Bolt plan to apply Puppet code and orchestrate the deployment of a package resource using the Chocolatey provider.

### 1. Build a project-specific configuration using a Bolt project directory Bolt runs in the context of a project directory. This directory contains all of the configuration, code, and data loaded by Bolt.
    
1. Create a module using the PDK command `pdk new module puppet_choco_tap` 

2. To make the module a Bolt project directory, add a `bolt.yaml` file. 

3. Add the following configuration to the `bolt.yaml` file. 

```
Modulepath: "./modules/:~/.puppetlabs/bolt-code/modules:~/.puppetlabs/bolt-code/site-modules"
```

This means that the project is going to support itself by deploying all module dependencies to a modules folder in the project. Note that you don’t have to create the `./modules` folder.

4. Create an inventory file to store information about your nodes. This is stored as `inventory.yaml` by default in the project directory. Add the following code: 

```
groups:
  - name: windows
    nodes:
      - chocowin0.classroom.puppet.com
      - chocowin1.classroom.puppet.com
    config:
      transport: winrm
      winrm:
        user: Administrator
        Password: <ADD PASSWORD>
```

5. To make sure that your inventory is configured correctly and that you can connect to all the target nodes, run the following command from inside the project directory: 

```
bolt command run 'echo hi' --targets windows
```

**Note:** The `--targets windows` argument refers to the node group defined in the inventory file.

You should get the following output:

```
Started on x.x.x.x...
Started on localhost...
Finished on localhost:
  STDOUT:
    hi
Finished on 0.0.0.0:
  STDOUT:
    hi
Finished on 127.0.0.1:
  STDOUT:
    hi
Successful on 3 nodes: 0.0.0.0:20022,localhost
Ran on 3 nodes in 0.20 seconds
```

### 2. Download the Chocolatey module and its dependencies from the Puppet Forge. 

Bolt uses a `Puppetfile` to install module content from the Forge. A `Puppetfile` is a formatted text file that specifies the modules and data you want in each environment.

1. Create a file named `Puppetfile` in the project directory — which describes the Forge content you want to install: 

```     
# Modules from the Puppet Forge
mod 'puppetlabs-chocolatey', '4.1.0'
## dependencies
mod 'puppetlabs-stdlib', '4.13.1' #install latest
 mod 'puppetlabs-powershell', '2.3.0'
mod 'puppetlabs-registry', '2.1.0'
        
# Modules from Git
# Examples: https://github.com/puppetlabs/r10k/blob/master/doc/puppetfile.mkd#examples
#mod 'apache',
#  :git    => 'https://github.com/puppetlabs/puppetlabs-apache',
#  :commit => 'de290646f97e04b4b8e42c70f6e01e860c394ce7'     
 ```

2. Add the Puppetfile to a version control service, for example, Github.

3. Add the following code to the Puppetfile — with your git user account: 

```     
mod 'puppet_choco_tap',
    :git    =>  'https://github.com/<your git user>/puppet_choco_tap.git'
```

4. From inside the project directory, install the Forge content by running: 

```
bolt puppetfile install
```

After it runs, you can see a `modules` directory inside the project directory, containing the modules you specified in the `Puppetfile`.

### 3. Write a Bolt plan to apply Puppet code and orchestrate the deployment of a package resource using the Chocolatey provider. Plans allow you to run more than one task with a single command.

1. Inside your project directory, in a file called `/plan/installer.pp`, create a plan called `puppet_chocolatey_tap::installer`. 

The folder tree should look like this:

```
├──plan
   └── installer.pp 
```

2. Copy the following code into the `installer.pp` file:

```
plan puppet_choco_tap::installer(
   TargetSpec                                   $nodes,
   String                                       $package,
   Variant[Enum['absent', 'present'], String ]  $ensure = 'present',
 ){

 apply_prep($nodes)

apply($nodes){
  include chocolatey
 }
apply($nodes){
  package { $package :
    ensure    => $ensure,
    provider  => 'chocolatey',
    }
  }
}
```

Take note of the following features of the plan:

- It has three parameters: the `TargetSpec`, a `package` string for the package name, and the `ensure` state of the package which allows for version, absent or present.
- It has the `apply_prep` function call, which is used to install modules needed by `apply` on remote nodes as well as to gather facts about the nodes.
- The first `apply` block installs the Chocolatey package manager. The Chocolatey provider is also deployed as a library with the Puppet agent in `apply_prep`.
- The second `apply` block installs a specified package using the Chocolatey provider.

2. To verify that the `puppet_chocolatey_tap` plan is available, run the following command: 

```
bolt plan show
```

The output looks like this:

```
facts
facts::info
puppet_choco_tap::installer
puppetdb_fact
```

3. Run the plan with the `bolt plan run` command: 

```
bolt plan run puppet_choco_tap::installer package=frogsay --nodes=win --password
```

The output looks like this:

```
Starting: plan puppet_choco_tap::installer
Starting: install puppet and gather facts on chocowin0.classroom.puppet.com, chocowin1.classroom.puppet.com
Finished: install puppet and gather facts with 0 failures in 22.11 sec
Starting: apply catalog on chocowin0.classroom.puppet.com, chocowin1.classroom.puppet.com
Finished: apply catalog with 0 failures in 18.77 sec
Starting: apply catalog on chocowin0.classroom.puppet.com, chocowin1.classroom.puppet.com
Finished: apply catalog with 0 failures in 33.74 sec
Finished: plan puppet_choco_tap::installer in 74.63 sec
```

4. To check that the installation worked, run the following `frogsay` command: 

```        
bolt command run 'frogsay ribbit' --targets windows --password
```

The result will vary on each server, and will look something like this:

```
Started on chocowin1.classroom.puppet.com...
Started on chocowin0.classroom.puppet.com...
Finished on chocowin0.classroom.puppet.com:
  STDOUT:
    
            DO NOT PAINT OVER FROG.
            /
      @..@
     (----)
    ( >__< )
    ^^ ~~ ^^
Finished on chocowin1.classroom.puppet.com:
  STDOUT:
    
            TO PREVENT THE RISK OF FIRE OR ELECTRIC SHOCK, DO NOT ENGAGE WITH FROG
            WHILE AUTOMATIC UPDATES ARE BEING INSTALLED.
            /
      @..@
     (----)
    ( >__< )
    ^^ ~~ ^^
Successful on 2 nodes: chocowin0.classroom.puppet.com,chocowin1.classroom.puppet.com
Ran on 2 nodes in 3.15 seconds
```

That’s it! In this one plan, you have both installed Chocolatey and deployed the package to two nodes. You can do the same thing on any number of nodes by editing the inventory file. Note that Chocolatey will remain installed on your machine.

After you have installed your package, with the help of Bolt, you can use Chocolatey to automate all of the package management tasks for upgrades or uninstalls. You can use Puppet Enterprise to guarantee state across all of your nodes and handle configuration drift — and make sure no one accidentally uninstalls the package that you just installed.

