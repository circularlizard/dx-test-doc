# Update deployment to a later version

This section shows how to update your HCL DX 9.5 Container Update CF197 and later deployment to a newer DX 9.5 Container Update release version.

To proceed, administrators should have prepared the container platform cluster, together with the HCL DX 9.5 container deployment custom-values.yaml using the following guidance, [Planning your container deployment using Helm](helm_planning_deployment.md), and then install your deployment using the instructions in [Install and uninstall commands for HCL DX 9.5 CF196 and later container deployments to Kubernetes and Red Hat OpenShift platforms using Helm](helm_install_commands.md).

**Important:**

-   As of HCL DX 9.5 Container Update CF197, you can use this process to update a DX 9.5 deployment from Container Update CF196 on the Google Kubernetes Engine \(GKE\) platform.
-   Support to update DX 9.5 197 container deployments using Helm to CF198 and later DX 9.5 container versions is provided for Red Hat OpenShift, Amazon EKS, Azure AKS, as well as Google GKE platforms beginning with Container Update CF198.

Follow the guidance in this section to update the HCL DX 9.5 container release version CF197 and later deployment, to Kubernetes or Red Hat OpenShift that was installed using Helm.

These instructions assume that you have made all configuration changes using the recommended Helm upgrade route described in [Updating the DX 9.5 Deployment Configuration](helm_operations.md). This ensures that your custom-values.yaml file is an updated description of the configuration of your environment. If that is not the case, you must update your custom-values.yaml file first with all configuration changes.

1.  **Populate your repository with the new images**

    Download the new HCL DX 9.5 container update images you need to upgrade and ensure that they are available in the image repository specified in your custom-values.yaml file. See the [Docker image list](https://help.hcltechsw.com/digital-experience/9.5/containerization/docker.html) for the latest HCL DX 9.5 container update images available.

2.  **Download the Helm charts for the version to be installed**

    Download the Helm charts corresponding to the HCL DX 9.5 container versions you want to install. You must always use the Helm charts that correspond to the container versions you are installing or to which you are upgrading.

3.  **Update the image tags**

    Update the image tags in your custom-values.yaml file to match those for the new images in your repository. See [Planning your container deployment using Helm](helm_planning_deployment.md) for more information.

4.  **Run the upgrade command**

    After making the changes to the custom-values.yaml file, use the following command to upgrade your HCL DX 9.5 deployment to CF197 and later release version:

    ```
    # Helm upgrade command
                            helm upgrade -n your-namespace -f path/to/your/custom-values.yaml your-release-name path/to/hcl-dx-deployment-vX.X.X_XXXXXXXX-XXXX.tar.gz
                        
    ```

    In this example:

    -   `your-namespace` is the namespace in which your HCL Digital Experience 9.5 Container Update deployment is installed and `your-release-name` is the Helm release name you used when installing.
    -   The `-f path/to/your/custom-values.yaml` parameter must point to the custom-values.yaml you updated.
    -   `path/to/hcl-dx-deployment-vX.X.X_XXXXXXXX-XXXX.tar.gz` is the HCL Digital Experience 9.5 Container Update Helm Chart that you extracted in the preparation steps.

