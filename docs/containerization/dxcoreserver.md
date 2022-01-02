# DX Core server

This topic provides information about restarting the DX Core server by using the DXClient tool.

## Restart DX Core server

The restart-dx-core command is used to restart the DX Core server.

-   **Command description**

    This command invokes the restart-dx-core tool inside the DXClient and runs the DX Core restart action.

    ```
    dxclient restart-dx-core
    ```

-   **Help command**

    This command shows the help information for `restart-dx-core` command usage:

    ```
    dxclient restart-dx-core -h
    ```

-   **Command options**

    Use this attribute to specify the username that is required for authenticating with the DX Core

    ```
    -dxUsername <value> 
    ```

    Use this attribute to specify the password that is required for authenticating with the DX Core

    ```
    -dxPassword <value>
    ```

    Use this attribute to specify the config wizard home \(route change only in case of Open Shift Kubernetes Enviornment, otherwise same as hostname\) that is required for authenticating to the cw\_profile

    ```
    -dxConnectHostname <value>
    ```

    Use this attribute to specify the port number of the cw\_profile\(for Kubernetes Environment dxConnectPort is 443\)

    ```
    -dxConnectPort <value>
    ```

    Use this attribute to specify the username that is required for authenticating to the cw\_profile

    ```
    -dxConnectUsername <value>
    ```

    Use this attribute to specify the password that is required for authenticating to the cw\_profile

    ```
    -dxConnectPassword <value>
    ```

    Specify either the `dxProfileName` or `dxProfilePath` of the DX core server:

    -   Use this attribute to specify the profile name of the DX core server \(for example: `wp_profile`\)

        ```
        -dxProfileName <Profile name of the DX core server>
        ```

    **OR**

    -   Use this attribute to specify the profile path of the DX server \(for example: `/opt/HCL/wp_profile`\)

        ```
        -dxProfilePath <Path of the DX core server profile> 
        ```

    All the above command options can also be configured inside the config.json configuration file of the DXClient tool, available in the <working-directory\>/store directory of the DXClient installation.

    **Note:** If you have installed DXClient using the node package file, then you can find the config.json file in the following path: dist/src/configuration.

    The values that are passed through the command line override the default values.

-   **Example Usage:**

    ```
    dxclient restart-dx-core -dxUsername <dxUsername> -dxPassword <dxPassword> -dxConnectHostname <hostname> -dxConnectPort <dxConnectPort> -dxConnectUsername <dxConnectUsername> -dxConnectPassword <dxConnectPassword> -dxProfileName <Profile name of the DX core server>
    ```

-   **Limitation**

    In Kubernetes based deployments, if there are more than one pod for the core container, the restart command of DXClient only restarts the pod it is connected to and not all running pods. For a full restart of all pods, leverage using the Kubernetes interfaces like `kubectl`.


