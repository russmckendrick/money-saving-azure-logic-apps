# Money Saving Azure Logic Apps

This code accompanies the blog post [Saving money with Azure Logic Apps](https://www.russ.cloud/2024/04/01/saving-money-with-azure-logic-apps/) and is provided as is.

## Deployment & Permissions

To deploy these Logic Apps, you need to create a User Assigned Managed Identity and grant it the necessary permissions to interact with your resources.

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/russmckendrick/money-saving-azure-logic-apps.git
    cd money-saving-azure-logic-apps
    ```

2.  **Set Environment Variables:**
    ```bash
    export RESOURCE_GROUP_NAME="rg-logicapps-automation"
    export REGION="uksouth"
    export SUBSCRIPTION_ID=$(az account show --query id --output tsv)
    export MANAGED_ID_NAME="mi-logicapps-automation"
    export LOGIC_APP_NAME="la-fabricCapacityPause" # Change as needed
    ```

3.  **Create Resource Group & Managed Identity:**
    ```bash
    az group create --name $RESOURCE_GROUP_NAME --location $REGION
    az identity create --resource-group $RESOURCE_GROUP_NAME --name $MANAGED_ID_NAME
    ```

4.  **Assign Roles (Permissions):**
    The Managed Identity needs **Reader** access to the subscription to list resources, plus specific permissions for the resources it manages.

    *   **Common (Reader on Subscription):**
        ```bash
        az role assignment create \
          --assignee-principal-type "ServicePrincipal" \
          --assignee-object "$(az identity show --resource-group $RESOURCE_GROUP_NAME --name $MANAGED_ID_NAME --query principalId --output tsv)" \
          --role "Reader" \
          --scope "/subscriptions/$SUBSCRIPTION_ID"
        ```

    *   **For Virtual Machines (Virtual Machine Contributor):**
        ```bash
        az role assignment create \
          --assignee-principal-type "ServicePrincipal" \
          --assignee-object "$(az identity show --resource-group $RESOURCE_GROUP_NAME --name $MANAGED_ID_NAME --query principalId --output tsv)" \
          --role "Virtual Machine Contributor" \
          --scope "/subscriptions/$SUBSCRIPTION_ID"
        ```

    *   **For Fabric Capacities (Contributor):**
        *Note: "Fabric Capacity Contributor" or similar specific roles may be preferred if available, but "Contributor" on the resource group is often used for broad automation context.*
        ```bash
        az role assignment create \
          --assignee-principal-type "ServicePrincipal" \
          --assignee-object "$(az identity show --resource-group $RESOURCE_GROUP_NAME --name $MANAGED_ID_NAME --query principalId --output tsv)" \
          --role "Contributor" \
          --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/YOUR_FABRIC_RESOURCE_GROUP"
        ```

5.  **Deploy Logic App:**
    ```bash
    az logic workflow create \
      --resource-group $RESOURCE_GROUP_NAME \
      --location $REGION \
      --name $LOGIC_APP_NAME \
      --mi-user-assigned "$(az identity show --resource-group $RESOURCE_GROUP_NAME --name $MANAGED_ID_NAME --query id --output tsv)" \
      --state "Disabled" \
      --definition "fabricCapacityPause.json"
    ```

6.  **Update Parameters:**
    ```bash
    # Update Managed Identity ID
    az logic workflow update \
      --resource-group $RESOURCE_GROUP_NAME \
      --name $LOGIC_APP_NAME \
      --set "definition.parameters.managedId.defaultValue=$(az identity show --resource-group $RESOURCE_GROUP_NAME --name $MANAGED_ID_NAME --query id --output tsv)"

    # Update Subscription ID
    az logic workflow update \
      --resource-group $RESOURCE_GROUP_NAME \
      --name $LOGIC_APP_NAME \
      --set "definition.parameters.subscriptionId.defaultValue=$SUBSCRIPTION_ID"
    ```