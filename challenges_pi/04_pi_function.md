# Communicate from the Azure IoT Hub to the Azure Machine Learing service

We created an Azure Function for you. It is triggered every time the Iot hub receives a message from your Pi and forwards the needed content to the Azure Machine Learning service. In return it also receives the rain prediction to the sensor temperature and humidity data.

We will stay on your local machine to implement this.

## Deploy Machine Learning model

The automated Machine Learning model should have trained by now. In order to be able to use the trained Machine Learning model, we first need to deploy it. Therefore, please follow the next steps:

(Btw, If you have trained and deployed the Machine Learning model using the ML Designer, you can skip this step.)

1. Navigate back to the _Azure Machine Learning Studio_ (via the portal move to your AML service and from there to the studio).
1. Navigate to _Jobs_ and select your experiment _predictRain_.

   ![Showing where AutoML can be found in the azure machine learning studio](/images/04experiments.png) <br>

1. Under _Display name_ you should see the details of your run. Select the only run there is.
   ![Showing where AutoML can be found in the azure machine learning studio](/images/04model.png) <br>
1. Select the _Models_ tab. <br>
   ![Showing where AutoML can be found in the azure machine learning studio](/images/04modeltap.png) <br>
1. There select the **VotingEnsemble** - it should be the best ML Algorithm for the given data. (_Note_: The first algorithm in the list is the best one.)
1. Select **Deploy** so we can consume this ML model as an endpoint. Choose the **Deploy to web service** option:
   ![Showing where AutoML can be found in the azure machine learning studio](/images/01automlws.png) <br>
   There give it a name e.g. **mlendpoint** and for _Compute type_ select **Azure Container Instance**. Switch **Enable authentication** to on. After that hit **Deploy**.
   ![Showing where AutoML can be found in the azure machine learning studio](/images/04deploy1.png) <br>
   This will take a bit so let's move on to the next task.


## Create an Azure Function and an Azure Storage Account locally
Open a terminal on your local computer again and make sure your prefix is still stored in it.
1. We will start by creating our general-purpose storage account.
    ```shell
    az storage account create --name $prefix'awjstorage' --location westeurope --resource-group $prefix'iotpirg' --sku Standard_LRS
    ```
1. We need to create an Azure function at this point. We are going to keep using Python.
    ```shell
    az functionapp create --resource-group $prefix'iotpirg' --consumption-plan-location westeurope --runtime python --runtime-version 3.9 --functions-version 3 --name $prefix'iotfunction' --os-type linux --storage-account $prefix'awjstorage'
    ```

## Prepare the function locally

1.  Now we start off locally. To work with Azure Funcitons locally we need to install the _Azure Functions Core Tools_. Follow [these instructions](https://docs.microsoft.com/en-us/azure/azure-functions/functions-run-local?tabs=v3%2Cwindows%2Ccsharp%2Cportal%2Cbash%2Ckeda#v2) to do so.
    Restart your terminal and enter this to make sure everything works:
    ```shell
    func --version
    ```
1.  Make sure you are up to date on the _AzureIoTHack_ git repo. While in the repo check:

    ```shell
    git pull
    ```

    We want to activate a virtual environment named .venv. We have already created all necessary parts, for you to create the Azure function. Therfore, change into the directory, where the Azure function files are stored.

    ```shell
    cd raspberrypi_function
    ```

    For Azure Functions we need to use Python Version 3.7, 3.8 or 3.9. Have a look whether you have the correct version installed. If not please do so.

    ```shell
    python --version
    ```

    If you have one or more versions installed you can set the version of the virtual environment you will create next by adding `-3.7`, `-3.8` or `-3.9` to the command.

    Using PowerShell: py -3.9 -m venv .venv (In case you have several versions of Python installed. Run as Administrator)

    ```shell
    py -m venv .venv
    ```

    ```shell
    .venv/scripts/activate
    ```

    Using bash:

    ```bash
    python -m venv .venv
    ```

    ```bash
    source .venv/bin/activate
    ```

    ```bash
    sudo apt-get install python3-venv
    ```

1.  While we implemented the function for you, it still needs connection to your resources. We are starting by adding the _AzureWebJobsStorage_ and the IoT hub _ConnectionString_ values in the 'local.settings.json'. To do so open the function in the IDE of your choice. E.g. with VS Code:
    ```shell
    cd raspberrypi_function
    code .
    ```
    Get the needed values:
    ```shell
    # for AzureWebJobsStorage
    az storage account show-connection-string --name $prefix'awjstorage' --resource-group $prefix'iotpirg' --output tsv
    ```
    Paste the output 'DefaultEndpointProtocol=https;A...' as value for _AzureWebJobsStorage_ in the _local.settings.json_.
    ```shell
    # for ConnectionString
    az iot hub connection-string show -n $prefix'iotpihub' --default-eventhub --output tsv
    ```

    Paste the output 'Endpoint=sb://...' as value for *ConnectionString* in the *local.settings.json*.

    ```shell
    # for DeviceConnectionString
    az iot hub connection-string show -n $prefix'iotpihub' --output tsv
    ```
    Paste the output 'HostName=...' as value for _DeviceConnectionString_ in the _local.settings.json_.

1.  We will also enter our Machine Learning model endpoint and its key.
    The az CLI extension for this is currently still experimental so we need to navigate back to the _Azure Machine Learning studio_.
    Under _Endpoints_ select the endpoint you previously deployed.
    ![Where to find the Endpoint](/images/01automlendoint.png) <br>
    On the _Consume_ tab of your endpoint you will find a **REST endpoint**. Paste the value 'http://...' as value for _AzureMLurl_ in the _local.settings.json_.
    Under _Authentication_ copy the **Primary key** and paste the value 'yqpie4...' as value for _AzureMLkey_ in the _local.settings.json_.
    ![Showing where AutoML can be found in the azure machine learning studio](/images/04basics.png) <br>
    As you are already here go to the _Test_ tab and test your endpoint.
    Your function is now ready to run. If you are using VS Code hit **F5** to start the function. If not start the function from the _raspberrypi_function_ folder by entering:


Now, ensure you have installed all required modules and packages shown in file _requirements.txt_

```shell
pip install -r requirements.txt
``` 

Your function is now ready to run. If you are using VS Code hit **F5** to start the function. If not start the function from the _raspberrypi_function_ folder by entering:

```shell
func start
```

## Run the updated app on your Pi
Connect to your Pi again. Open a terminal and run the 'temphumidrain.py' script.
Now we can run the application. Make sure you are in the 'raspberry_app' folder and run the following:
 ```bash
python3 temphumidrain.py 
```
Temperature and humidity data are still send to the Azure IoT Hub and displayed on the Sense Hat's LEDs.

Now our Pi also listens to the Azure IoT hub, which forwards the result of your ML model to your Pi. There it displays the result as a sun, if the prediction from temperature and humidity data is no rain, and an umbrella if the result is rain.

## Deploy the Azure function

1. Now we are going to run our function in Azure. Since this command takes the Python version of the current environment make sure to run it from within the .venv environment.
   ```shell
   func azure functionapp publish $prefix'iotfunction'
   ```
1. There is one thing missing. Our Connection String and the connection to the Azure Storage account currently reside in the `local.settings.json` file of the function project. This file will not be uploaded to Azure (see `.funcignore` for the files that will not be uploaded). We can set the needed keys in the Azure portal. So first navigate to the portal.
1. There find your Azure Function and from there the function `iothubtrigger` you just uploaded under `Function`.
   ![](/images/04iothubtrigger.png)
1. Here you can also see an overview of how often the function was triggered. For now we need to enter the keys that we did have in the `local.settings.json`. Navigate to `Function Keys`, select `+ New function key` and kopy all the key-value-pairs from your `local.settings.json`. TThe result should look like this with the addition of the DeviceConnectionString:
   ![](/images/04functionkeys.png)

This is how your final object should look like:
![Showing the menue in the Azure portal with the + create button being on the very left](/images/architecture.png)

Go to the [next steps](./06_pi_cicd.md)


