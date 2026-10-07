# Running Microsoft SQL Server on Linux using Docker
**Study Guide & Troubleshooting Log**

## 1. Summary of What We Achieved
We successfully set up a Microsoft SQL Server database environment on a Linux distribution (CachyOS) using Docker. Because SQL Server Management Studio (SSMS) is Windows-only, we bypassed this limitation by containerizing the database engine and connecting to it using a modern, cross-platform client (Visual Studio Code with the MSSQL extension). 

## 2. Step-by-Step: Setting Up the Docker Environment
To recreate this environment from scratch on a Linux machine, follow these steps:

### A. Install and Start Docker
Docker must be installed and its background service (daemon) must be running.
1. Install Docker: `sudo pacman -S docker` (or your distribution's equivalent).
2. Enable and start the service: `sudo systemctl enable --now docker`

### B. Deploy the SQL Server Container
We download the official Microsoft image and run it as an isolated container.
1. Run the container:
   `sudo docker run -e "ACCEPT_EULA=Y" -e 'MSSQL_SA_PASSWORD=ServidorSQL2026' -p 1433:1433 --name mssql -d mcr.microsoft.com/mssql/server:2022-latest`
   * `-e "ACCEPT_EULA=Y"`: Agrees to Microsoft's terms.
   * `-e 'MSSQL_SA_PASSWORD=...'`: Sets the System Administrator (`sa`) password. Single quotes prevent the Linux terminal from misinterpreting special characters.
   * `-p 1433:1433`: Maps your computer's port 1433 to the container's port 1433.
   * `--name mssql`: Names the container for easy management.
   * `-d`: Runs the container in the background (detached mode).

### C. Connect via Visual Studio Code
1. Install the **SQL Server (mssql)** extension in VS Code.
2. Add a new connection with these details:
   * **Server:** `localhost`
   * **Authentication Type:** `SQL Login`
   * **User name:** `sa`
   * **Password:** `ServidorSQL2026`
   * **Encrypt:** `Optional` or `False` (Crucial for local Docker setups to bypass SSL certificate checks).

---

## 3. Why Use Bash? (And How It Works)
You asked why we have to do this via the command line (Bash/Fish) and how it works.

**How Docker Works:** Docker operates on a Client-Server architecture. The "Server" is the Docker daemon (`dockerd`), a background service managing containers. The "Client" is the command-line interface (`docker run`, `docker ps`). When you type a command in Bash, the CLI sends a request to the daemon to execute the task.

**Why Bash?** 
1. **Direct Control:** The command line is the most direct and resource-efficient way to interact with the Docker daemon. 
2. **Reproducibility:** A single text string (like our `docker run` command) can perfectly recreate a complex server environment anywhere.

**Are there alternatives?**
Yes. You don't *have* to use the terminal if you prefer graphical interfaces. You can use GUI tools like **Docker Desktop**, **Portainer**, or **Dockge**. These programs essentially click buttons that trigger the exact same API calls to the Docker daemon that your terminal commands do. However, learning the CLI is industry-standard for developers and system administrators.

---

## 4. The Issue We Faced & How We Solved It

### The Problem
When trying to run the Docker container, we received this error: 
`failed to bind host port 0.0.0.0:1433/tcp: address already in use`. 
This meant another program was already using port 1433, preventing Docker from mapping the port.

### The Investigation & Resolution
We went through a systematic troubleshooting process using Linux networking commands:

1. **`sudo docker ps -a`**
   * *Function:* Lists all Docker containers (running and stopped).
   * *Result:* We used this to see if a broken Docker container was holding the port. It wasn't the culprit.

2. **`sudo ss -tulpn | grep 1433`**
   * *Function:* `ss` dumps socket statistics (network connections). `-tulpn` shows TCP, UDP, listening ports, and the program names/PIDs. `grep 1433` filters the output to only show line(s) containing "1433".
   * *Result:* We discovered a process named `sqlservr` with PID `1096` was actively listening on port 1433. 

3. **`sudo fuser -k 1433/tcp`** (and `kill -9`)
   * *Function:* Attempts to forcefully kill whatever process is using a specific port.
   * *Result:* It killed the process momentarily, but it kept coming back. This indicated it was a system service configured to auto-restart.

4. **The Root Cause:**
   * You had a native installation of SQL Server running directly on your host operating system (CachyOS) via `systemd`. This native service was hogging port 1433.

5. **The Final Solution:**
   * **`sudo systemctl stop mssql-server`**: Stops the native SQL Server service.
   * **`sudo systemctl disable mssql-server`**: Prevents it from booting up automatically when you restart your computer.
   * **`sudo docker rm -f mssql`**: Forcefully removes the broken container from our failed attempts.
   * *Result:* Port 1433 was permanently freed, allowing the `docker run` command to successfully bind the port and start the container.