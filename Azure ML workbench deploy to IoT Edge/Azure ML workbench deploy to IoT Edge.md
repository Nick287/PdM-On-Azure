# How to deploy Azure ML Workbench to IoT edge.

## Prerequisites

In this session We assume you have basic knowledge and has setup below services on Azure

- Setup IoT Edge & IoT Hub Azure service https://docs.microsoft.com/en-us/azure/iot-edge/quickstart-linux
- Installed Azure IoT Edge runtime in your development environment
If haven’t installed & setup, follow this instruction to prepare your development machine: https://docs.microsoft.com/en-us/azure/iot-edge/tutorial-csharp-module
- A Azure Container Registry or A Docker Hub account

## Step 1：Setup an IoT Edge Device and push docker image to Edge

In this demo we create an Linux Ubuntu with Docker VM first and then follow the iot Edge quick start document setup this VM as IoT Edge environment. after that let's push Azure ML docker image to this edge device  (Note: Please name Azure ML image as irisapp).

### Add registry credentials to Edge runtime
Add the credentials for your registry to the Edge runtime on the computer where you are running your Edge device. These credentials give the runtime access to pull the container.

For Linux, run the following command:
``` cmd/sh
sudo iotedgectl login --address <your container registry address> --username <username> --password <password> 
```

In the Azure portal, navigate to your IoT hub. ->
Go to IoT Edge (preview) and select your IoT Edge device. ->
Select Set Modules. ->
Select Add IoT Edge Module. ->
In the Name field, enter irisapp. ->
In the Image URI field, enter your docker registry address. ->

In the Container Create Option field，enter ->
```javascript
{
  "HostConfig": {
    "PortBindings": {
      "8883/tcp": [
        {
          "HostPort": "3883"
        }
      ],
      "5001/tcp": [
        {
          "HostPort": "3001"
        }
      ],
      "8888/tcp": [
        {
          "HostPort": "3888"
        }
      ]
    }
  }
}
```
please summit your module to deploy, after that you can check the module running normaly.

## step 2: Through the Http query test Azure ML web service endpoint

In this session I will use Postman to test web service endpoint, before the test please conform your VM（Edge device）there is no firewall blocking the port.

Please you POST request follow this required
- Web service endpoint: http://< IoT Edge IP>:3001/Score
- Headers Key "Content-Type", Value "application/json"
- Body
```javascript
{"input_df": [{"petal width": 0.25, "sepal length": 3.0, "sepal width": 3.6, "petal length": 1.3}]}
```
- return value is "\"Iris-setosa\""

## Step 3： Develop and deploy a C# IoT Edge module to your 

### Prerequisites

- The Azure IoT Edge device that you created in the quickstart or first tutorial.
- The primary key connection string for the IoT Edge device.
- [Visual Studio Code.](https://code.visualstudio.com/)
- [Azure IoT Edge extension for Visual Studio Code.](https://marketplace.visualstudio.com/items?itemName=vsciot-vscode.azure-iot-edge)
- [C# for Visual Studio Code (powered by OmniSharp) extension.](https://marketplace.visualstudio.com/items?itemName=ms-vscode.csharp)
- [Docker](https://docs.docker.com/install/) on the same computer that has Visual Studio Code. The Community Edition (CE) is sufficient for this tutorial.
- [.NET Core 2.0 SDK.](https://www.microsoft.com/net/learn/get-started/macos#windowscmd)


### Create a container registry

- You can use any Docker-compatible registry for this tutorial. Two popular Docker registry services available in the cloud are Azure Container Registry and Docker Hub. This tutorial uses Azure Container Registry.
- In the Azure portal, select Create a resource > Containers > Azure Container Registry.
- Give your registry a name, choose a subscription, choose a resource group, and set the SKU to Basic.
- Select Create.
- Once your container registry is created, navigate to it and select Access keys.
T- oggle Admin user to Enable.
- Copy the values for Login server, Username, and Password. You'll use these values later in the tutorial when you publish the Docker image to your registry, and when you add the registry credentials to the Edge runtime.

### Create an IoT Edge module project
The following steps show you how to create an IoT Edge module based on .NET core 2.0 using Visual Studio Code and the Azure IoT Edge extension.

- In Visual Studio Code, select View > Integrated Terminal to open the VS Code integrated terminal.
- In the integrated terminal, enter the following command to install (or update) the AzureIoTEdgeModule template in dotnet:
``` C#
dotnet new -i Microsoft.Azure.IoT.Edge.Module
```
- Create a project for the new module. The following command creates the project folder, sendMessage, with your container repository. The second parameter should be in the form of < your container registry name>.azurecr.io if you are using Azure container registry. Enter the following command in the current working folder:
``` C#
dotnet new aziotedgemodule -n sendMessage -r <your container registry address>/sendmessage
```
- Select File > Open Folder.
- Browse to the sendMessage folder and click Select Folder to open the project in VS Code.
- In VS Code explorer, click Program.cs to open it and replace these code to this file.

``` C#
namespace sendMessage
{
    using System;
    using System.IO;
    using System.Runtime.InteropServices;
    using System.Runtime.Loader;
    using System.Security.Cryptography.X509Certificates;
    using System.Text;
    using System.Threading;
    using System.Threading.Tasks;
    using Microsoft.Azure.Devices.Client;
    using Microsoft.Azure.Devices.Client.Transport.Mqtt;

    class Program
    {
        static int counter;

        static string message = "{\"input_df\": [{\"sepal length\": 3.0, \"sepal width\": 3.6, \"petal width\": 1.3, \"petal length\":0.25}]}";

        static System.Timers.Timer timer = new System.Timers.Timer(5000);

        static void Main(string[] args)
        {
            // The Edge runtime gives us the connection string we need -- it is injected as an environment variable
            string connectionString = Environment.GetEnvironmentVariable("EdgeHubConnectionString");

            // Cert verification is not yet fully functional when using Windows OS for the container
            bool bypassCertVerification = RuntimeInformation.IsOSPlatform(OSPlatform.Windows);
            if (!bypassCertVerification) InstallCert();
            Init(connectionString, bypassCertVerification).Wait();

            // Wait until the app unloads or is cancelled
            var cts = new CancellationTokenSource();
            AssemblyLoadContext.Default.Unloading += (ctx) => cts.Cancel();
            Console.CancelKeyPress += (sender, cpe) => cts.Cancel();
            WhenCancelled(cts.Token).Wait();
        }

        /// <summary>
        /// Handles cleanup operations when app is cancelled or unloads
        /// </summary>
        public static Task WhenCancelled(CancellationToken cancellationToken)
        {
            var tcs = new TaskCompletionSource<bool>();
            cancellationToken.Register(s => ((TaskCompletionSource<bool>)s).SetResult(true), tcs);
            return tcs.Task;
        }

        /// <summary>
        /// Add certificate in local cert store for use by client for secure connection to IoT Edge runtime
        /// </summary>
        static void InstallCert()
        {
            string certPath = Environment.GetEnvironmentVariable("EdgeModuleCACertificateFile");
            if (string.IsNullOrWhiteSpace(certPath))
            {
                // We cannot proceed further without a proper cert file
                Console.WriteLine($"Missing path to certificate collection file: {certPath}");
                throw new InvalidOperationException("Missing path to certificate file.");
            }
            else if (!File.Exists(certPath))
            {
                // We cannot proceed further without a proper cert file
                Console.WriteLine($"Missing path to certificate collection file: {certPath}");
                throw new InvalidOperationException("Missing certificate file.");
            }
            X509Store store = new X509Store(StoreName.Root, StoreLocation.CurrentUser);
            store.Open(OpenFlags.ReadWrite);
            store.Add(new X509Certificate2(X509Certificate2.CreateFromCertFile(certPath)));
            Console.WriteLine("Added Cert: " + certPath);
            store.Close();
        }


        /// <summary>
        /// Initializes the DeviceClient and sets up the callback to receive
        /// messages containing temperature information
        /// </summary>
        static async Task Init(string connectionString, bool bypassCertVerification = false)
        {
            Console.WriteLine("Connection String {0}", connectionString);

            MqttTransportSettings mqttSetting = new MqttTransportSettings(TransportType.Mqtt_Tcp_Only);
            // During dev you might want to bypass the cert verification. It is highly recommended to verify certs systematically in production
            if (bypassCertVerification)
            {
                mqttSetting.RemoteCertificateValidationCallback = (sender, certificate, chain, sslPolicyErrors) => true;
            }
            ITransportSettings[] settings = { mqttSetting };

            // Open a connection to the Edge runtime
            DeviceClient ioTHubModuleClient = DeviceClient.CreateFromConnectionString(connectionString, settings);
            await ioTHubModuleClient.OpenAsync();
            Console.WriteLine("IoT Hub module client initialized.");

            // Register callback to be called when a message is received by the module
            await ioTHubModuleClient.SetInputMessageHandlerAsync("input1", PipeMessage, ioTHubModuleClient);

            timer.Elapsed += new System.Timers.ElapsedEventHandler((s, e) => OnTimedEvent(s, e, ioTHubModuleClient));
            timer.Start();

        }

        static async void OnTimedEvent(object sender, System.Timers.ElapsedEventArgs e, DeviceClient deviceClient)
        {
            int counterValue = Interlocked.Increment(ref counter);
            Console.WriteLine("----------------------------------------------------");
            Console.WriteLine(counter + " times senting message");
            //Console.WriteLine(message);

            var sendMessage = new Message(System.Text.Encoding.Default.GetBytes(message));

            //sendMessage.Properties.Add("MessageType", "Alert");

            byte[] messageBytes = sendMessage.GetBytes();

            string messageString = Encoding.UTF8.GetString(messageBytes);

            Console.WriteLine("message is : " + messageString);

            await deviceClient.SendEventAsync("sendmessageOutput",sendMessage);

            Console.WriteLine("message sented");

            Console.WriteLine("----------------------------------------------------");
        }

        /// <summary>
        /// This method is called whenever the module is sent a message from the EdgeHub. 
        /// It just pipe the messages without any change.
        /// It prints all the incoming messages.
        /// </summary>
        static async Task<MessageResponse> PipeMessage(Message message, object userContext)
        {
            int counterValue = Interlocked.Increment(ref counter);

            var deviceClient = userContext as DeviceClient;
            if (deviceClient == null)
            {
                throw new InvalidOperationException("UserContext doesn't contain " + "expected values");
            }

            byte[] messageBytes = message.GetBytes();
            string messageString = Encoding.UTF8.GetString(messageBytes);
            Console.WriteLine($"Received message: {counterValue}, Body: [{messageString}]");

            if (!string.IsNullOrEmpty(messageString))
            {
                var pipeMessage = new Message(messageBytes);
                foreach (var prop in message.Properties)
                {
                    pipeMessage.Properties.Add(prop.Key, prop.Value);
                }
                await deviceClient.SendEventAsync("output1", pipeMessage);
                Console.WriteLine("Received message sent");
            }
            return MessageResponse.Completed;
        }
    }
}
```
Save this file.

### Create a Docker image and publish it to your registry

- Sign in to Docker by entering the following command in the VS Code integrated terminal:
``` csh/sh
docker login -u <ACR username> -p <ACR password> <ACR login server>
```
To find the user name, password and login server to use in this command, go to the Azure portal. From All resources, click the tile for your Azure container registry to open its properties, then click Access keys. Copy the values in the Username, password, and Login server fields.

- In VS Code explorer, Right-click the module.json file and click Build and Push IoT Edge module Docker image. In the pop-up dropdown box at the top of the VS Code window, select your container platform, either amd64 for Linux container or windows-amd64 for Windows container. VS Code then builds your code, containerize the sendmessage.dll and push it to the container registry you specified.

- You can get the full container image address with tag in the VS Code integrated terminal. For more infomation about the build and push definition, you can refer to the module.json file.

### Add registry credentials to Edge runtime
Add the credentials for your registry to the Edge runtime on the computer where you are running your Edge device. These credentials give the runtime access to pull the container.

- For Windows, run the following command:

``` cmd/sh
iotedgectl login --address <your container registry address> --username <username> --password <password> 
```
- For Linux, run the following command:

``` cmd/sh
sudo iotedgectl login --address <your container registry address> --username <username> --password <password> 
```


### Run the solution
- In the Azure portal, navigate to your IoT hub.
- Go to IoT Edge (preview) and select your IoT Edge device.
- Select Set Modules.
- Select Add IoT Edge Module.
  - In the Name field, enter sendmessage.
  - In the Image URI field, enter your image address; for example < your container registry address>/sendmessage:0.0.1-amd64. The full image address can be found from the previous section.
- Click Save and then Click Next.

In the Specify Routes step, copy the JSON below into the text box. Modules publish all messages to the Edge runtime. Declarative rules in the runtime define where the messages flow. In this tutorial, you need two routes. The first route transports messages from the sendmessage model to the MLmodel module via the "amlInput" endpoint, which is the endpoint configured with the irisapp model. The second route transports messages from the filter module to IoT Hub. In this route, upstream is a special destination that tells Edge Hub to send messages to IoT Hub.

``` javascript
{
  "routes": {
    "sendmessageToIoirisapp": "FROM /messages/modules/sendmessage/outputs/sendmessageOutput INTO BrokeredEndpoint(\"/modules/irisapp/inputs/amlInput\")",
    "irisappToIoTHub": "FROM /messages/modules/irisapp/outputs/amlOutput INTO $upstream"
  }
}
```
- Click Next.

- In the Review Template step, click Submit.
- Return to the IoT Edge device details page and click Refresh. You should see the new sendmessage running along with the irisapp module and the IoT Edge runtime.

### View generated data
To monitor device to cloud messages sent from your IoT Edge device to your IoT hub:

- Configure the Azure IoT Toolkit extension with connection string for your IoT hub:
  - Open the VS Code explorer by selecting View > Explorer.
  - In the explorer, click IOT HUB DEVICES and then click .... Click Set IoT Hub Connection String and enter the connection string for the IoT hub that your IoT Edge device connects to in the pop-up window.
  - To find the connection string, click the tile for your IoT hub in the Azure portal and then click Shared access policies. In Shared access policies, click the iothubowner policy and copy the IoT Hub connection string in the iothubowner window.
- To monitor data arriving at the IoT hub, select View > Command Palette and search for the IoT: Start monitoring D2C message menu command.
  - The message body value is \Iris-setosa\
- To stop monitoring data, use the IoT: Stop monitoring D2C message menu command.
