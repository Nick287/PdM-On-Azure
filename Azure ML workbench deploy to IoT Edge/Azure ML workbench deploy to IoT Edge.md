# How to deploy Azure ML Workbench to IoT edge.

## Prerequisites

In this session We assume you have basic knowledge and has setup below services on Azure

- Setup IoT Edge & IoT Hub Azure service https://docs.microsoft.com/en-us/azure/iot-edge/quickstart-linux
- Installed Azure IoT Edge runtime in your development environment
If haven’t installed & setup, follow this instruction to prepare your development machine: https://docs.microsoft.com/en-us/azure/iot-edge/tutorial-csharp-module
- A Azure Container Registry or A Docker Hub account

## Step 1：Setup an IoT Edge Device and push docker image to Edge

In this demo we create an Linux Ubuntu with Docker VM first and then follow the iot Edge quick start document setup this VM as IoT Edge environment. after that let's push Azure ML docker image to this edge device  (Note: Please name Azure ML image as irisapp).

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

In the Container Create Option field，enter ->
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
please summit your module to deploy, after that you can check the module running normaly.

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

## Step 3： Develop and deploy a C# IoT Edge module to your 

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

## 技术细节

每个发送到共享中心的事件消息都需要被格式化为JSON。并且每个消息体中都需要一个全局的唯一标识符（GUID），也就是说这个唯一的标识符对应一位游客。另外模拟入口的时间戳也必须被记录。JSON的格式如下：

``` javascript
{
    "ticketId": {{uniqueIdentifier}},
    "entryTime": {{currentTime}}
}
```

## 成功标准（挑战步骤）
- 步骤 1 - **在您的 Azure 门户中创建一个 IoT Hub**
  - 您的团队需要将服务的部署环境定位到美国西部。
  - 团队统一配置并指定到一个消息中心。虽然您团队中的每一位成员都可以创建一个中心，用于体验中心创建和配置过程，但是最终当这个挑战结束时所有的模拟器都应该将消息发送到团队的统一的IoT Hub。
  - 注意阅读参考资料文档中如何选择IoT Hub 中的”定价和缩放级别“以适应吞吐量的缩放需求。
  - 请参阅 [使用 .NET 将设备连接到 IoT 中心](https://docs.microsoft.com/zh-cn/azure/iot-hub/iot-hub-csharp-csharp-getstarted) 前半部分内容，创建一个IoT Hub

- 步骤 2 - **您的每一位团队成员都需要向IoT Hub发送消息**
  - 您的团队成员需要创建一个客户端程序将JSON信息发送到云端IoT Hub
  - 请参阅 [使用 .NET 将设备连接到 IoT 中心](https://docs.microsoft.com/zh-cn/azure/iot-hub/iot-hub-csharp-csharp-getstarted) 后半部分内容，创建客户端程序将数据发送到IoT Hub
  - 如果您是使用其他语言的开发者请在文档中切换您熟悉的开发语言。例如：Java，Node.js, Python

- 步骤 3 - **使用Azure IoT Hub Device Explorer检查数据上传结果**
  - 您的团队需要实时监控客户端上传到IoT Hub的信息。
  - 您所有的团队成员终端模拟设备需要同时运行
  - [使用说明](https://github.com/Azure/azure-iot-sdk-csharp/tree/master/tools/DeviceExplorer) 
  - [下载 msi版本 可运行程序](https://github.com/Azure/azure-iot-sdks/releases)

- 最终，结合1-3步您和您的队友同时需要启动一个或多个客户端程序向IoT Hub发送数据，并通过Azure IoT Hub Device Explorer查看数据上传结果。


## 必要条件

- IDE 环境或代码编辑器
- 团队的Azure订阅账号


## 其他参考资料 - 帮助您更了解 IoT Hub 更多内容

  - [Azure IoT 中心与 Azure 事件中心的比较](https://docs.microsoft.com/zh-cn/azure/iot-hub/iot-hub-compare-event-hubs). 注：更多信息请参见支持信息部分。
  - [Azure 和物联网](https://docs.microsoft.com/zh-cn/azure/iot-hub/iot-hub-what-is-azure-iot) 在 Azure 中创建IoT解决方案。
  - [Azure IoT 中心服务概述](https://docs.microsoft.com/zh-cn/azure/iot-hub/iot-hub-what-is-iot-hub) 如何开始使用 [using IoT Hub](https://docs.microsoft.com/zh-cn/azure/iot-hub/iot-hub-get-started-simulated). 在资源列表部分您可以找到各种监视和诊断工具。


## 支持信息

- 阅读更多关于 [在 Azure 中选择实时消息引入技术](https://docs.microsoft.com/zh-cn/azure/architecture/data-guide/technology-choices/real-time-ingestion)

- 虽然IoT Hub 和 Event Hub 的结点是兼容的 [partitioning / 分区](https://docs.microsoft.com/zh-cn/azure/event-hubs/event-hubs-features#partitions)，并且这里我们不限定单一的成功标准，在这里多了解一些最佳实践也是对您的团队非常有帮助的。请阅读 - [Event Hubs - 最佳实践](https://docs.microsoft.com/azure/event-hubs/event-hubs-faq#best-practices)。如果有疑问请和我们的同事一起讨论。

- [IoT 中心配额和限制](https://docs.microsoft.com/zh-cn/azure/iot-hub/iot-hub-devguide-quotas-throttling)

- 并不是所有的IoT Hub 都可以进行扩展. 请您阅读 [IoT 中心 定价](https://azure.microsoft.com/pricing/details/iot-hub/).


#

# 挑战 2

## 用户背景

在成功的完成试点项目以后，Contoso 的IT人员开始将智能旋转门部署到公园入口。然而，随着云霄飞车在运行过程中多次出现小问题随之成为了人们关注的焦点。据报道，出现了各种异常现象，如照片设备数据故障和列车本身故障。游乐场的管理层认为过山车的口碑对公园至关重要，所以现要求组织所有技术资源都必须收集云霄飞车的数据随后进行分析问题来源。

云霄飞车上已经部署过一些传感器设备可以收集到收集大量数据其中包括，包括速度、加速度、乘客数量以及每次运行的其他关键事件数据。因为目前生成的数据保存在列车本地, Contoso 公司想使用从 IoT Hub 收集上来的数据进行数据的管理分析。

因为 Contoso 不确定他们今后要分析的数据有哪些，所以他们要求目前存储云霄飞车所产生的所有数据。另外该公司也意识到，能够实时的处理数据也是非常关键的，所以他们还希望部署一个流处理引擎来处理接收到的数据并使用可视化工具展示出数据。

## 技术细节

- 您的团队需要模拟五辆云霄飞车同时运行时所产生的数据流并且将数据上传到IoT Hub。

- 所上传的数据格式需要符合JSON格式标准 - 包括三个传感器 GPS，加速器，事件捕捉器的信息。

- GPS data

``` javascript
{
    "rideId": "2327F177-8079-43E1-BA50-569455E2FADD",
    "trainId": "6FA812B3-FAE8-4875-8D80-62B037CD3528",
    "correlationId": "9C67DC77-F1BA-4BFA-8F38-0F76F3D5EC58",
    "lat": "44.8547",
    "long": "-93.2428",
    "alt": "246.509",
    "speed": "4.37",
    "vertAccuracy": "4",
    "horizAccuracy": "10",
    "deviceTime": "2017-12-06T20:23:43.9790000Z"
}
```

- Accelerometer
  
``` javascript
{
    "rideId": "61397CA0-89ED-4F8C-8997-86F32AEEBD2E",
    "trainId": "05D8569B-69F9-40C8-B862-E197F9F0331E",
    "correlationId": "BB72B77A-687D-4809-92B0-407EA3633B3C",
    "accelX": "0.701859",
    "accelY": "2.19725",
    "accelZ": "1.26033",
    "deviceTime": "2018-01-17T19:06:08.0490000Z"
}
```

- Events  
注意: 这里有三种类型的触发事件：RideStart（飞车启动），RideEnd（飞车终止），PhotoTriggered（行驶过程中拍照）；
  
``` javascript
{
    "rideId": "61397CA0-89ED-4F8C-8997-86F32AEEBD2E",
    "trainId": "05D8569B-69F9-40C8-B862-E197F9F0331E",
    "correlationId": "BB72B77A-687D-4809-92B0-407EA3633B3C",
    "passengerCount": "30",
    "eventType": "PhotoTriggered",
    "deviceTime": "2018-01-17T19:06:08.7490000Z"
}

{
    "rideId": "61397CA0-89ED-4F8C-8997-86F32AEEBD2E",
    "trainId": "05D8569B-69F9-40C8-B862-E197F9F0331E",
    "correlationId": "BB72B77A-687D-4809-92B0-407EA3633B3C",
    "passengerCount": "30",
    "eventType": "RideStart",
    "deviceTime": "2018-01-17T19:06:08.7490000Z"
}

{
    "rideId": "61397CA0-89ED-4F8C-8997-86F32AEEBD2E",
    "trainId": "05D8569B-69F9-40C8-B862-E197F9F0331E",
    "correlationId": "BB72B77A-687D-4809-92B0-407EA3633B3C",
    "passengerCount": "30",
    "eventType": "RideEnd",
    "deviceTime": "2018-01-17T19:06:08.7490000Z"
}
```

- 当数据到达IoT Hub以后需要您将数据导出到Azure存储账户中去。
- 另外您需要使用Power BI对数据进行可视化展示。

## 成功标准（挑战步骤）

- **步骤 1 模拟云霄飞车数据并且数据上传到 IoT Hub**
  - 修改挑战1中-步骤2 设备上传代码中的JSON格式，并且随机生成2辆云霄飞车三个传感器的数据上传到IoT Hub。
  - 注意：JSON文件属性中，每辆云霄飞车有唯一标识，为trainID. 云霄飞车每行驶一次即一圈，该次被赋予行驶ID，为rideID.
  - 时间：deviceTime为云霄飞车每次行驶的起始时间, 一定注意时间的格式与当前格式保持一致。
  - 要求：每隔5秒钟上传一次云霄飞车数据，数据需要包含三个传感器的各项数据。假设每辆云霄飞车每次发车间隔时间为10秒钟

- **步骤 2 Stream Processing / 流分析**
  - 当您的数据上传到IoT Hub的时，实际会在数据IoT Hub中缓存7天，在对应大量数据流入的场景中您可以使用Azure Stream Analytics进行实时数据 分析/导出/抽取 操作，所以在做数据可视化展示之前您需要先将数据从IoT Hub中提取出来。
  - 请参考 [开始使用 Azure 流分析处理来自 IoT 设备的数据](https://docs.microsoft.com/zh-cn/azure/stream-analytics/stream-analytics-get-started-with-azure-stream-analytics-to-process-data-from-iot-devices)
  - 相关案例参考 [使用流分析构建 IoT 解决方案](https://docs.microsoft.com/zh-cn/azure/stream-analytics/stream-analytics-build-an-iot-solution-using-stream-analytics)

- **步骤 3 Data Archival / 数据归档**
  - 您的团队需要将数据保存到Azure订阅中，并为其创建一个存储帐户中的容器中。
  - 您的团队应提供一个数据归档解决方案，并提供一个带有数据的Azure Blob存储容器。
  - Azure Blob存储容器中的数据应该按照年、月、日的颗粒度度进行划分存储。
  - 请参考 [流分析输出：存储、分析选项](https://docs.microsoft.com/zh-cn/azure/stream-analytics/stream-analytics-define-outputs) - 通过流分析存储数据
  - 请参考 [将包含传感器数据的 IoT 中心消息保存到 Azure Blob 存储](https://docs.microsoft.com/zh-cn/azure/iot-hub/iot-hub-store-data-in-azure-table-storage) - 通过设置IoT Hub 节点存储

- **步骤 4 生成报告**
  - 您的团队需要借助 ***Stream Processing / 流分析服务*** 中输出的数据并结合Power BI展示数据内容应该包括以下表格:
    - 实时显示每辆云霄飞车的乘客人数
    - 统计云霄飞车的拍照次数，从而查看拍照收益  
  - 您的可视化报告必须符合以下条件：
    - 处理输入数据源是动态数据源，而不是blob存储中提供的静态测试数据。
    - 对于每一个云霄飞车，用5分钟为单位进行实时显示。
    - 可视化报告必须基于消息内的时间戳计算，而不是消息到IoT Hub的时间。   
  - 您可以参考以下示例
    - [使用 Power BI 可视化 Azure IoT 中心的实时传感器数据](https://docs.microsoft.com/zh-cn/azure/iot-hub/iot-hub-live-data-visualization-in-power-bi)
    - [流分析和 Power BI：针对流数据的实时分析仪表板](https://docs.microsoft.com/zh-cn/azure/stream-analytics/stream-analytics-power-bi-dashboard)

## 提示信息

**注意：对于挑战2 其实不仅限于 Azure Stream Analytics 一种解决方案 如果时间充裕您也可以阅读以下资料尝试更多技术方案**

- 更多的云霄飞车 = 更多的数据 = 需要更多 IoT Hub 承载力.

- 有一些文档在HDInsight中写的关于Spark，这些文档可以和Azure中的Databricks一起使用(有时甚至不需要修改)。如果在Databricks文档中找不到什么东西，那么请在HDInsight上找找类似的内容。

- 对于Spark Structured streaming，数据从IoT Hub传输进来，存储在“body” column中，并且IoT Hub / Event Hub 默认会将数据采用二进制格式进行传输。您的团队应该将二进制数据转换成字符串格式，以便能够使用它。

- 流处理引擎可能需要一点时间才能连接到流并开始使用数据。如果时间超过5分钟还没有数据呈现那可能是哪里出现了问题。

- 不要忘记消息有课能delay需要处理。

- Spark结构化流支持在数据上编写SQL查询，比如“Spark”。sql(' SELECT * FROM myTable ')'

- Apache Hive样式的SQL语言也是可用的，虽然不是所有的函数都支持。

- 当使用Spark Structured streaming时，只有在使用Java或Scala时，才能完成这些挑战。另外在Python中完成这一挑战也是可能的，只是需要一个额外的处理步骤。

## 参考资料

- **流处理 / Stream Processing  & 数据存储 - Spark / Data Storage - Spark**
  - [A Gentle Introduction to Apache Spark on Databricks](https://docs.azuredatabricks.net/spark/latest/gentle-introduction/gentle-intro.html#azure-gentle-introduction-to-apache-spark-azure)
  - [Spark Structured Streaming Programming Guide](http://spark.apache.org/docs/latest/structured-streaming-programming-guide.html)
  - [Working with Complex Data Formats with Structured Streaming in Apache Spark 2.1](https://databricks.com/blog/2017/02/23/working-complex-data-formats-structured-streaming-apache-spark-2-1.html) - Good information on dealing with converting from JSON to a tabular format.
  - [Apache Spark Structured Streaming on HDInsight to process events from Event Hubs](https://docs.microsoft.com/azure/hdinsight/spark/apache-spark-eventhub-structured-streaming#run-spark-shell-on-your-hdinsight-cluster)
  - [Access Azure Blob Storage from Azure Databricks](https://docs.databricks.com/spark/latest/data-sources/azure/azure-storage.html)
  - [Using Spark on Azure Databricks to consume data from EventHubs](https://lenadroid.github.io/posts/connecting-spark-and-eventhubs.html)

- **流处理 / Stream Processing & 数据存储与流分析 / Data Storage - Azure Stream Analytics**
  - [什么是流分析？](https://docs.microsoft.com/zh-cn/azure/stream-analytics/stream-analytics-introduction)
  - [数据连接：了解从事件到流分析的数据流输入](https://docs.microsoft.com/zh-cn/azure/stream-analytics/stream-analytics-define-inputs)
  - [流分析输出：存储、分析选项](https://docs.microsoft.com/zh-cn/azure/stream-analytics/stream-analytics-define-outputs)


## 支持信息

- 如果Spark是您团队选择的流处理环境，则需要一个事件集线器连接器连接到数据流。该连接器的Maven坐标为:
  - Azure Databricks: [com.microsoft.azure:azure-eventhubs-databricks_2.11:3.4.0](http://search.maven.org/#artifactdetails%7Ccom.microsoft.azure%7Cazure-eventhubs-databricks_2.11%7C3.4.0%7Cjar)
  - HDInsight: [com.microsoft.azure:azure-eventhubs-spark_2.11:2.1.6](http://search.maven.org/#artifactdetails%7Ccom.microsoft.azure%7Cazure-eventhubs-spark_2.11%7C2.1.6%7Cjar)

- 理解如何 [监视 Azure IoT 中心的运行状况并快速诊断问题](https://docs.microsoft.com/zh-cn/azure/iot-hub/iot-hub-monitor-resource-health).

- 理解 Azure 中的 - [HDInsight](https://docs.microsoft.com/azure/hdinsight/spark/apache-spark-overview) 和 [Azure Databricks](https://docs.azuredatabricks.net)

- 常见问题 [PySpark SQL Functions](http://spark.apache.org/docs/latest/api/python/pyspark.sql.html) & [Scala Spark SQL Column Functions](http://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.Column)

- Python 介绍 [Beginner's Guide to Python](https://wiki.python.org/moin/BeginnersGuide)

- Scala 介绍 [Tour of Scala](http://docs.scala-lang.org/tour/tour-of-scala.html)

- 理解如何加载外部程序集在 [HDInsight](https://docs.microsoft.com/azure/hdinsight/spark/apache-spark-jupyter-notebook-use-external-packages) 和 [Azure Databricks](https://docs.azuredatabricks.net/user-guide/libraries.html)

- [Why Would I Ever Need to Partition My Big ‘Raw’ Data?](https://www.red-gate.com/simple-talk/cloud/cloud-data/ever-need-partition-big-raw-data/)

- [Hadoop Azure Support: Azure Blob Storage](https://hadoop.apache.org/docs/current/hadoop-azure/index.html).
