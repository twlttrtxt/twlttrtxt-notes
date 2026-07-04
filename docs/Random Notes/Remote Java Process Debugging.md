To start debugging a running Java process, the following steps are required:

1. First off, install `Eclipse` and `Java-Decompiler-Eclipse`.
2. Within Eclipse, go to `Help > Install New Software...`
3. Select the Archive button and upload the `Java-Decompiler-Eclipse` package.
4. Set the OK and install it.

#### Important:
The associations must be set correctly. Under this window in `Eclipse`:
```
Window -> Preferences -> File Associations
```
Set the `*.class`, and `*.class without source` to the new package `JD Class File Viewer (default)`.

#### Creating a big Java Archive
The tool [jarjarbigs](https://github.com/mogwailabs/jarjarbigs) can create a big JAR file from a project, including all of its dependencies:
```bash
python3 jarjarbigs.py /source/dir /destination/dir
```

Then, in `Eclipse` a new Java project must be created using `Java 1.8`.

#### Setting up the process for remote debugging
To enable remote debugging, find the command which started the Java process, and append the following flag onto it:
```bash
-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=127.0.0.1:8000
```

If this process is a Java web-server, the `PID` of the server can be found using `sudo netstat -tulnp`, and its command line call can be read at `cat /proc/<PID>/cmdline`. This command can be then searched in `systemd`.

Alternatively, a `ps aux | grep java` might also find the command.

#### Eclipse Debugging Activation
To enable remote debugging in `Eclipse`, click on the bug icon and go to `Debug Configurations ...`. There, the `Remote Java Application` may be selected, and a `Apply` and `Debug` starts the debugging process. `Navigate > Open Type` lets you search for specific classes!

