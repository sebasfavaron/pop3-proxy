# POP3 Proxy Server with Content Filtering

This project implements a POP3 proxy server that filters email content based on configurable rules. It consists of three main components:

1.  **POP3 Proxy Server (`proxyPop3nio`)**: This server acts as an intermediary between a POP3 client (e.g., an email application) and a POP3 origin server. It relays POP3 traffic and applies content filtering rules to emails before they reach the client. The core logic for this component can be found in `server/src/main.c`.

2.  **SPCP Management Client (`spcpClient`)**: This command-line client is used to manage and monitor the POP3 proxy server. It communicates with the proxy using a custom SPCP (Simple Proxy Configuration Protocol). Functionalities include viewing server metrics, adding or removing filtered media types, and modifying the replacement message for filtered content. The source code is located in `client/src/client.c`.

3.  **Stripmime Utility (`stripmime`)**: This utility is responsible for the actual content filtering. It parses the MIME structure of email messages and removes or replaces parts of the message that match the configured filtering criteria. The proxy server utilizes `stripmime` as an external transformer for email bodies. The implementation details are in `stripmime/stripmime.c`.

## Interaction

The `proxyPop3nio` server processes email traffic, utilizing `stripmime` for content filtering. Simultaneously, the `spcpClient` can connect to the proxy's management interface to monitor and configure its operation. More detailed operational flow is described in the "How it Works" section.

## POP3 Proxy Features

The POP3 proxy server (`proxyPop3nio`) forms the core of this project. Its primary responsibilities and features include:

*   **POP3 Protocol Handling**: It faithfully implements the POP3 protocol, acting as a server to the email client and as a client to the origin POP3 server. This allows it to transparently relay email communications.
*   **Content Filtering Engine**: The proxy incorporates a flexible content filtering mechanism.
    *   **MIME-Type Based Filtering**: Filtering decisions are made based on the MIME types of email attachments or message parts.
    *   **External Transformer**: The actual transformation of the email body is delegated to an external command. While any compatible command can be used, the `stripmime` utility provided in this project is the default transformer.
    *   **User-Defined Filtering Rules**: Users can configure which specific media types are targeted for filtering. They can also specify a custom message that will replace the content of any filtered parts.
*   **Command-Line Configuration**: Many operational parameters of the proxy can be configured at startup via command-line arguments. This includes network settings, filtering options, and the external transformation command. Detailed options are listed further in this document.

## SPCP Management Client

The `spcpClient` is a command-line utility designed for configuring and monitoring the POP3 proxy server. It facilitates administrative tasks by interacting with the proxy's dedicated management interface.

*   **Communication Protocol**: The client uses the SPCP (Simple Proxy Configuration Protocol), a custom text-based protocol, to send commands and receive responses from the proxy server.
*   **Key Functionalities**:
    *   **User Authentication**: Secure access to management functions is ensured through a login mechanism.
    *   **Metrics Retrieval**: Administrators can query the proxy for various operational metrics, including:
        *   Current number of concurrent connections.
        *   Total number of bytes transferred since the proxy started.
        *   Number of historical connections.
    *   **Transformation Configuration**:
        *   View and modify the external command used for email content transformation (e.g., `stripmime`).
        *   View and modify the buffer size allocated for the transformation process.
*   **Command-Line Configuration**: The network address and port for connecting to the proxy's SPCP server can be specified using command-line arguments. Further details on these options are available later in this document.

## Stripmime Utility

The `stripmime` utility is a command-line tool specifically designed for parsing and filtering MIME-encoded email messages. It operates as a filter, reading the raw email content from standard input and writing the processed (and potentially modified) email to standard output.

*   **Role in the Proxy**: `stripmime` serves as the default external transformer program for the POP3 proxy server. When the proxy identifies an email that requires content filtering, it invokes `stripmime` to perform the actual modification.
*   **Configuration via Environment Variables**: The behavior of `stripmime` is controlled through environment variables. This design allows the proxy server (or any other invoking process) to dynamically configure the filtering rules:
    *   `FILTER_MEDIAS`: This variable takes a comma-separated list of MIME types that should be censored. For example, to filter JPEG images and ZIP archives, this variable would be set to `image/jpeg,application/zip`.
    *   `FILTER_MSG`: This variable defines the message that will replace the content of any MIME part whose type matches one of the types listed in `FILTER_MEDIAS`.
*   **Invocation by the Proxy**: The POP3 proxy server sets the `FILTER_MEDIAS` and `FILTER_MSG` environment variables based on its own configuration. It then executes the `stripmime` command, piping the email data through `stripmime`'s standard input and reading the filtered email from its standard output.

## How it Works

This section provides a high-level overview of the email and management data flow when the POP3 proxy system is active.

### Email Flow

1.  **Client Connection**: An email client application (e.g., Thunderbird, Outlook) is configured to connect to the `proxyPop3nio` server's address and port instead of the actual POP3 origin server.
2.  **Proxy Relay**: The `proxyPop3nio` server receives POP3 commands from the client. It relays these commands to the designated origin POP3 server and forwards the origin server's responses (including email data) back to the client.
3.  **Content Filtering**: For incoming emails (during `RETR` or `TOP` commands), before the email data is sent to the client, the proxy server pipes the raw email through an external transformation command.
    *   By default, this command is `stripmime`.
    *   The proxy sets the `FILTER_MEDIAS` (list of MIME types to filter) and `FILTER_MSG` (replacement message) environment variables for the `stripmime` process based on its current configuration.
    *   `stripmime` parses the email's MIME structure. If it encounters any parts with a MIME type present in the `FILTER_MEDIAS` list, it replaces that part's content with the string from `FILTER_MSG`.
4.  **Delivery to Client**: The email data, now potentially modified by the transformation command, is sent to the email client.

### SPCP Management Channel

Alongside handling POP3 traffic, the `proxyPop3nio` server also listens for management connections on a separate port (configurable via the `-o` option). This communication uses the SPCP (Simple Proxy Configuration Protocol), which is a custom, text-based protocol layered over SCTP.

The `spcpClient` utility connects to this management port, allowing an administrator to:

*   **Authenticate**: Provide credentials to access management functionalities.
*   **Monitor Metrics**: Retrieve real-time statistics from the proxy, such as the number of active connections, total bytes transferred, and historical connection counts.
*   **Manage Proxy Settings**: View and modify certain operational parameters of the proxy. For example, the client can change the external transformation command and the buffer size used for the transformation process. (Note: Direct modification of `FILTER_MEDIAS` or `FILTER_MSG` via the current SPCP client is not explicitly supported; these are typically set at proxy startup or managed by other means if dynamic changes are needed beyond what the `spcpClient` offers).

This dual-channel architecture separates the email processing path from the management and monitoring path, allowing for robust and independent operation.

## Dependencies

To build and run this project, the following software components are required:

*   **C Compiler**: A standard C compiler (such as GCC or Clang) is necessary to compile the C source code for the proxy, client, and stripmime utility.
*   **Make**: The `make` utility is used to manage the build process. The project includes Makefiles that automate the compilation of the different components.
*   **Standard C Libraries**: Core C libraries (e.g., `libc`) are essential for compilation and execution. These are typically included in any standard C development environment.
*   **SCTP Library**: The SPCP management protocol between the proxy server and the client operates over SCTP. Therefore, development libraries for SCTP are required.
    *   On Debian-based Linux distributions (like Ubuntu), this can usually be installed via a package named `libsctp-dev`.
    *   For other systems, the package name for SCTP development libraries might vary.

Ensure these dependencies are installed on your system before attempting to build the project.

## Making the Executables

The project is divided into three main components, each with its own executable. Follow the steps below to compile them. Each component is built using `make` from within its respective source directory.

### POP3 Proxy Server (`proxyPop3nio`)

1.  Navigate to the server's source directory:
    ```bash
    cd server/src/
    ```
2.  Compile the proxy server:
    ```bash
    make
    ```
    The compiled executable, `proxyPop3nio`, will be located in the `server/src/` directory.

### SPCP Management Client (`spcpClient`)

1.  Navigate to the client's source directory:
    ```bash
    cd client/src/
    ```
2.  Compile the SPCP client:
    ```bash
    make
    ```
    The compiled executable, `spcpClient`, will be located in the `client/src/` directory.

### Stripmime Utility (`stripmime`)

1.  Navigate to the stripmime utility's directory:
    ```bash
    cd stripmime/
    ```
2.  Compile the utility:
    ```bash
    make
    ```
    The compiled executable, `stripmime`, will be located in the `stripmime/` directory.

## Running the Executables and Their Options

Once compiled, the executables can be run from their respective directories. Below are the instructions and available command-line options for each component.

### Running the POP3 Proxy Server (`proxyPop3nio`)

The POP3 proxy server is run from its source directory.

**Command:**
```bash
cd server/src/
./proxyPop [OPTIONS] <origin-server-address>
```

The `<origin-server-address>` is the hostname or IP address of the POP3 server your clients will connect to through this proxy.

**Options:**

All options are in POSIX style (e.g., `-o value` or `-o "value with spaces"`).

*   `-e <filter-error-file>`: Specifies the path to a file where errors related to content filtering will be logged. Defaults to `/dev/null` (errors are discarded).
*   `-h`: Prints the help dialogue, showing all available options, and then exits.
*   `-l <pop3-address>`: Specifies the local IP address on which the proxy will listen for incoming POP3 client connections. By default, it listens on all available network interfaces (`0.0.0.0`).
*   `-L <config-address>`: Specifies the local IP address on which the SPCP management service will listen. Defaults to `127.0.0.1` (loopback, only accessible from the same machine).
*   `-m <message>`: Specifies the default message used to replace filtered content. This can be overridden by the SPCP client.
*   `-M <censored-media-types>`: A comma-separated list of MIME types to be censored by default (e.g., `image/gif,application/x-shockwave-flash`). This list can be modified at runtime via the SPCP client.
*   `-o <management-port>`: Specifies the TCP port where the SPCP management service will listen. Defaults to `9090`.
*   `-p <local-port>`: Specifies the TCP port where the proxy will listen for incoming POP3 client connections. Defaults to `1110`.
*   `-P <origin-port>`: Specifies the TCP port of the origin POP3 server that the proxy will connect to. Defaults to `110`.
*   `-t <cmd>`: Specifies the external command to execute for email transformations. This command is expected to read from standard input and write to standard output. The `stripmime` utility is an example of such a command.
*   `-v`: Prints the proxy version information and then exits.

### Running the SPCP Management Client (`spcpClient`)

The SPCP client is run from its source directory. It connects to the POP3 proxy's management interface.

**Command:**
```bash
cd client/src/
./spcpClient [OPTIONS]
```

**Options:**

*   `-L <proxy-config-address>`: Specifies the IP address where the POP3 proxy's SPCP management service is listening. This should match the address set by the proxy's `-L` option. Defaults to `127.0.0.1`.
*   `-o <proxy-management-port>`: Specifies the TCP port where the POP3 proxy's SPCP management service is listening. This should match the port set by the proxy's `-o` option. Defaults to `9090`.

### Running the Stripmime Utility (`stripmime`)

The `stripmime` utility is typically invoked by the POP3 proxy server, but it can also be run directly from its directory for testing or manual filtering.

**Command:**
```bash
cd stripmime/
./stripmime <optional-input-file>
```
If `<optional-input-file>` is provided, `stripmime` will read from that file. Otherwise, it reads from standard input. The output is always sent to standard output.

**Configuration (Environment Variables):**

The behavior of `stripmime` is controlled by the following environment variables:

*   `FILTER_MEDIAS`: A comma-separated list of MIME types to be filtered (e.g., `image/jpeg,application/pdf`).
*   `FILTER_MSG`: The message that will replace the content of any filtered MIME parts.

Example usage (manual filtering):
```bash
cd stripmime/
export FILTER_MEDIAS="image/jpeg,image/png"
export FILTER_MSG="[Image content removed by stripmime]"
cat original_email.eml | ./stripmime > filtered_email.eml
```
