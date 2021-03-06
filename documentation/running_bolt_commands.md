# Running basic Bolt commands

Use Bolt commands to connect directly to the systems where you want to execute commands, run scripts, and upload files.

## Run a command on remote nodes

Specify the command you want to run and which nodes to run it on.

When you have credentials on remote systems, you can use Bolt to run commands across those systems.

-   To run a command on a list of nodes:
    ```shell script
    bolt command run <COMMAND> --nodes <NODE NAME>,<NODE NAME>,<NODE NAME>
    ```
-   To run a command on WinRM nodes, indicate the WinRM protocol in the nodes string:
    ```shell script
    bolt command run <COMMAND> --nodes winrm://<WINDOWS.NODE> --user <USERNAME> --password <PASSWORD>
    ```
-   To run a command that contains spaces or shell special characters, wrap the command in single quotation marks:
    ```shell script
    bolt command run 'echo $HOME' --nodes web5.mydomain.edu,web6.mydomain.edu
    ```
    ```shell script
    bolt command run "netstat -an | grep 'tcp.*LISTEN'" --nodes web5.mydomain.edu,web6.mydomain.edu
    ```
-   To run a cross-platform command:
    ```shell script
    bolt command run "echo 'hello world'"
    ```

    **Note:** When connecting to Bolt hosts over WinRM that have not configured SSL for port 5986, passing the `--no-ssl` switch is required to connect to the default WinRM port 5985.


## Running commands with redirection or pipes

When you run one-line commands that include redirection or pipes, pass `bash` or another shell as the command.

Using a shell ensures that the one-liner is run as a single command and that it works correctly with `run-as`. For example, instead of `bolt command run "echo foo > /root/foo" --run-as root`, use `bolt command run "bash -c 'echo foo > /root/foo'" --run-as root`.

## Run a script on remote nodes

Specify the script you want to run and which nodes to run it on.

Use the `bolt script run` command to run existing scripts that you use or to combine the commands that you regularly run as part of sequence. When you run a script with Bolt, the script is transferred into a temporary directory on the remote system, run on that system, and then deleted.

You can run scripts in any language as long as the appropriate interpreter is installed on the remote system. This includes Bash, PowerShell, or Python.

-   To run a script, specify the path to the script, and which nodes to run it on:
    ```shell script
    bolt script run <PATH/TO/SCRIPT> --nodes <NODE NAME>,<NODE NAME>,<NODE NAME>
    ```
    ```shell script
    bolt script run ../myscript.sh --nodes web5.mydomain.edu,web6.mydomain.edu
    ```
-   When executing on WinRM nodes, include the WinRM protocol in the nodes string:
    ```shell script
    bolt script run <PATH/TO/SCRIPT> --nodes winrm://<NODE NAME> --user <USERNAME> --password <PASSWORD>
    ```
-   To pass arguments to a script, specify them after the command. If an argument contain spaces or special characters, you must quote it:
    ```shell script
    bolt script run myscript.sh 'echo hello'
    ```
    Argument values are passed literally and are not interpolated by the shell on the remote host. If you run `bolt script run myscript.sh 'echo $HOME'`, then the script receives the argument `'echo $HOME'`, rather than any interpolated value.


### Requirements for scripts run on remote \*nix systems

A script must include a shebang (`#!`) line specifying the interpreter. For example, for a script written in Bash, provide the path to the Bash interpreter:

```shell script
#!/bin/bash
echo hello
```

### Requirements for scripts run on remote Windows systems

Bolt supports the extensions `.ps1`, `.rb`, and `.pp`. To enable other file extensions, add them to your Bolt configuration file, as follows:

```yaml
winrm:
   extensions: [.py, .pl]
```

## Upload files or directories to remote nodes

Use Bolt to copy files or directories to remote nodes.

**Note:** Most transports are not optimized for file copying, so this command is best limited to small files.

-   To upload a file or directory to a remote node, run the `bolt file upload` command. Specify the local path to the file or directory, the destination location, and the target nodes.

    ```
    bolt file upload <SOURCE> <DESTINATION> --nodes <NODE NAME>,<NODE NAME>
    ```
    
    ```
    bolt file upload my_file.txt /tmp/remote_file.txt --nodes web5.mydomain.edu,web6.mydomain.edu
    ```
