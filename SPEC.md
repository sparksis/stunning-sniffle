## Spec Document: `sudo-secure-askpass`

`sudo-secure-askpass` is a CLI utility designed to allow users to securely authenticate with `sudo` on a remote machine using their local password manager, without exposing passwords to the remote system. This tool mitigates the security risks of both manually entering passwords and using `NOPASSWD` rules. It will be developed in Go to produce a single, statically-linked binary.

-----

### User Stories

  * **As a user,** I want to run `sudo` on a remote system and have my trusted local machine securely provide the password from my password manager.
  * **As a user,** I do not want to type or copy/paste complex passwords for `sudo`.
  * **As a user,** I want to be prompted for explicit approval on my local machine before a `sudo` command is executed on a remote system.

-----

### Goals & Non-Goals

#### Goals

  * **Secure Authentication:** Provide a secure way to use `sudo` on a remote host without exposing the plaintext password.
  * **User Experience:** Eliminate manual password entry for `sudo` on remote systems.
  * **Explicit Approval:** Require a local, interactive approval from the user before providing a password.
  * **Portability:** Deliver a single, statically-linked binary that is easy to install and use on various platforms.
  * **Integration:** Leverage existing tools like SSH and the `1Password` CLI.

#### Non-Goals

  * **Universal Password Manager Support:** The initial version will only support 1Password's `op` CLI.
  * **In-Process SSH:** The tool will not implement its own SSH client. It will rely on the system's `ssh` command.
  * **Automated `sudo`:** The tool is designed for interactive use and will not support fully automated, unattended `sudo` execution.

-----

### High-Level Architecture

The system consists of two primary components and a secure communication channel.

  * **Local Daemon (`sudo-secure-askpass-daemon`):** A persistent service running on the user's local machine. It manages a **Unix domain socket** and communicates with the `1Password` CLI for secure credential retrieval.
  * **Remote CLI (`sudo-secure-askpass`):** A small CLI utility that is deployed on the remote host. It is invoked by the user's shell and by `sudo` via the `SUDO_ASKPASS` variable.
  * **Secure Tunnel:** An SSH reverse tunnel, established by the Remote CLI, securely forwards traffic from a socket on the remote machine to the local daemon's socket.

The workflow is:

1.  The user's shell profile sources a script that defines a `sudo` wrapper function.
2.  This wrapper function launches the **Remote CLI**, which in turn starts an `ssh` session with a **reverse tunnel** to the local daemon's socket.
3.  The wrapper sets `SUDO_ASKPASS` to point to a temporary script that wraps the Remote CLI.
4.  When the user runs `sudo`, it executes the `SUDO_ASKPASS` script.
5.  The script sends a request (containing the command) over the secure tunnel to the local daemon.
6.  The local daemon receives the request, prompts the user via the `1Password` agent for approval.
7.  Upon approval, the daemon retrieves the password and sends it back through the tunnel.
8.  The `SUDO_ASKPASS` script provides the password to `sudo`, which executes the command.

-----

## Technical Implementation Plan

### Language & Tooling

We will use **Go (Golang)** for development. Go's ability to produce single, statically-linked binaries makes distribution straightforward, as there are no runtime dependencies beyond the binary itself. This fits our portability goal perfectly. The project will leverage standard Go libraries where possible.

### Component Breakdown

#### **1. Remote CLI (`sudo-secure-askpass`)**

This binary will be deployed on the remote host. Its primary responsibilities are:

  * **SSH Reverse Tunneling:** The tool will programmatically construct and execute an `ssh -R` command to create the secure tunnel. The command will be similar to `ssh -R /tmp/remote-sock:/local/path/to/daemon-sock user@localhost`. The remote socket path will be randomized to prevent conflicts.
  * **`SUDO_ASKPASS` Logic:** When run with the `SUDO_ASKPASS` flag, it will act as an IPC client. It will read the command from standard input (provided by `sudo`), package it into a JSON request, send it over the socket, and wait for the password response. It will then print the password to standard output, as required by `sudo`.
  * **User Interface:** The tool will provide clear, concise output to the user during the process, indicating that a local prompt is active and waiting for approval.

#### **2. Local Daemon (`sudo-secure-askpass-daemon`)**

This binary will run as a persistent service on the user's local machine. Its responsibilities include:

  * **Unix Socket Server:** It will listen on a dedicated Unix domain socket with strict file permissions, accessible only by the current user.
  * **Request Handling:** Upon receiving a connection from the SSH tunnel, it will parse the incoming JSON request.
  * **1Password Integration:** It will use the `os/exec` package to call the `op` CLI with the appropriate arguments. The `op` CLI will handle the secure local prompt (GUI or terminal) and return the password.
  * **Response Handling:** It will send the retrieved password back to the remote CLI through the socket.
  * **Daemon Management:** It will handle signals for graceful shutdown and logging.

### Communication Protocol

A simple JSON-based protocol will be used for communication over the Unix socket.

**Request from Remote CLI to Local Daemon:**

```json
{
  "command": "sudo apt update",
  "remote_host": "web-server-01"
}
```

**Response from Local Daemon to Remote CLI:**

```json
{
  "password": "your-complex-password"
}
```

This protocol is simple, human-readable, and extensible. We can add fields later, such as `user` or `request_id`, without breaking existing functionality.

### Build & Distribution

The project will use Go's native cross-compilation to build binaries for macOS, Linux (x86-64), and potentially Windows Subsystem for Linux (WSL). We will use a `Makefile` or a simple build script to automate the process. All dependencies will be statically linked to ensure the output is a single, executable file.

### Security Considerations

  * **No Password on Remote Host:** The password is never stored or processed in plaintext on the remote machine. It is only passed through the `SUDO_ASKPASS` script's stdin to `sudo`'s stdin.
  * **Secure Tunnel:** The SSH reverse tunnel provides end-to-end encryption for the communication channel.
  * **Unix Domain Sockets:** Using a socket with restricted file permissions (`chmod 600`) ensures that only the intended processes can read from or write to the communication channel.

-----

### Milestones

  * **Phase 1 (MVP):**
      * Build the `sudo-secure-askpass-daemon` in Go with a basic Unix socket listener.
      * Build the `sudo-secure-askpass` CLI to send a hard-coded request over a manually created reverse tunnel.
      * Integrate with the `op` CLI to retrieve a test password.
  * **Phase 2 (Functionality):**
      * Implement the automated SSH reverse tunnel creation within the CLI.
      * Pass the real `sudo` command and host info to the daemon.
      * Complete the end-to-end workflow and test on both macOS and Linux.
  * **Phase 3 (Polishing):**
      * Add robust error handling and user-friendly output.
      * Create a simple installation script.
      * Document the project for a public release.
