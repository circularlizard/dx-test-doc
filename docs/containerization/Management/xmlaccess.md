# XML Access

This topic provides information about the xmlaccess command that is used to export or import portlet configurations.

## XML Access

The `xmlaccess` command is used to export or import pages or portlet configurations from a target HCL DX 9.5 CF19 or later server using the input XMLAccess file.

**Required file**

XMLAccess file : This XML file must contain the configuration update or export operation for the web application.

**Command**

```
dxclient xmlaccess -xmlFile <path>
```

**Help command**

This command shows the help information for `xmlaccess` command usage:

```
dxclient xmlaccess -h
```

**Command options**

Use this attribute to specify the protocol with which to connect to the DX server \(wp\_profile\):

```
-dxProtocol <value>
```

Use this attribute to specify the hostname of the target DX server:

```
-hostname <value>
```

Use this attribute to specify the port on which to connect to the DX server \(`wp_profile`\):

```
-dxPort <value>
```

Use this attribute to specify the path to DX configuration endpoint \(e.g. /wps/config\):

```
-xmlConfigPath <value>
```

Use this attribute to specify the username to authenticate with the DX server \(`wp_profile`\):

```
-dxUsername <value>
```

Use this attribute to specify the password for the user in the `dxUsername` attribute:

```
-dxPassword <value>
```

Use this attribute to specify the local path to the XMLAccess file:

```
-xmlFile <Absolute or relative path to xmlaccess input file>
```

All of the above command options can also be configured in the config.json configuration file of the DXClient tool, available in the <working-directory\>/store directory of the DXClient installation.

**Note:** If you have installed DXClient using the node package file, then you can find the config.json file in the following path: dist/src/configuration.

Command options passed through the command line overrides values set in the config.json file.

Log files from command execution can be found in the logs directory of the DXClient installation.

**Example usage:**

```
dxclient xmlaccess -xmlFile <xml-file-with-path>
```

