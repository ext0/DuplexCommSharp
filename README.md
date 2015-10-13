# DuplexCommSharp
Reverse client-server duplex communication via polling + WCF in C#

<h3><b>What does this do?</b></h3>
<ul>
<li>Allows for reverse client-server interaction</li>
<li>Creates safe and concurrent solution for handling pushed requests to multiple clients</li>
</ul>

<h3><b>How to use?</b></h3>

<h4><b>Server side setup</b></h4>
Modify the public enum ClientOperations (list of operations that will be availible to be pushed to the client) on the server-side.
For this example, we will consider the requirements for making a program that will serve as a remote file explorer (from server->client).

For this example, the ClientOperations would look like this.
```
	public enum ClientOperations {
		readFile,
		writeFile,
		getDirectoryEntries,
		getParentDirectory
	}
```
Now, you will need to set up your listeners. Listeners are functions that will be called and passed parameters (almost always of Types Client and Object) when certain events arise. The basic listener is a getListener, which is called when a client initially connects to the server and awaits operations.

If you wanted your console to print the name of a client whenever they connect, you could achieve this with the following code.

```
	public static void main(String[]args) {
		ClientConnectionHandler service = ClientConnectionHandler.createService(8080) //port to listen on
		service.Open();
		OperationQueue.addGetListener((client)=>{
			Console.WriteLine(client.username + " has connected!");
		});
		Console.ReadKey();
	}
```

Now the client has established a connection to the server and will await operations. The OperationQueue class feeds instructions to clients on a queue based system, so you have to push commands and wait for them to be executed. If there are no commands in the queue, the client will be sent a NOP command, meaning No Operation. Pushing commands to the OperationQueue is very simple, all you have to do is call the addToQueue function. This function takes two different sets of parameters depending on whether or not you want to send data to the client (which would be required for an operation that needs data such as writeFile, but NOT necessary for a static operation such as getUsername). The following example will illustrate how to add the two types of operations to the OperationQueue using the code example shown previously. 

```
	public static void main(String[]args) {
		ClientConnectionHandler service = ClientConnectionHandler.createService(8080) //port to listen on
		service.Open();
		OperationQueue.addGetListener((client)=>{
			Console.WriteLine(client.username + " has connected!");
			OperationQueue.addToQueue(
				client, 
				ClientOperations.getUsername
			);
			String pathToRead = "...";
			OperationQueue.addToQueue(
				client, 
				ClientOperations.readFile, 
				new List<byte[]> { Encoding.ASCII.GetBytes(pathToRead) },
				pathToRead + "callback"
		});
		Console.ReadKey();
	}
```

Now the request will be sent to the client on the next poll cycle (handled client-side), and will be sent back (if data needs to be returned) to the ClientDatastore to be received by pre-configured listeners. Example below.

```
	public static void main(String[]args) {
		ClientConnectionHandler service = ClientConnectionHandler.createService(8080) //port to listen on
		service.Open();
		OperationQueue.addGetListener((client)=>{
			Console.WriteLine(client.username + " has connected!");
			OperationQueue.addToQueue(
				client, 
				ClientOperations.getUsername
			);
			String pathToRead = "...";
			OperationQueue.addToQueue(
				client, 
				ClientOperations.readFile, 
				new List<byte[]> { Encoding.ASCII.GetBytes(pathToRead) },
				pathToRead + "callback"
			);
			ClientDatastore.addListener(client, pathToRead + "callback", (client, obj)=>{
				File.WriteAllBytes("outputFile.bin",(byte[])obj);
				Console.WriteLine("Wrote file received from "+client.username+" to outputFile.bin");
			});
		});
		Console.ReadKey();
	}
```

The whole system works around the listener + datastore system, where the cycle is as follows.
<ul>
<li>Create listener in ClientDatastore</li>
<li>Queue operation in OperationQueue</li>
<li>Operation completes, return data stored in ClientDatastore</li>
<li>Listener called and return data is used.</li>
</ul>
So, for the code shown before, the cycle would be as follows.
<ul>
<li>Create listener in ClientDatastore, set to callback on key (path + "callback")</li>
<li>Queue readFile operation for client in operationQueue</li>
<li>Client side operation finishes reading file, bytes sent back and stored in the entry (path + "callback")</li>
<li>Listener is alerted of data entry and listener is called with the data returned</li>
<li>Listener function writes bytes to outputFile.bin</li>
This system can be manipulated in many ways to allow for complex duplex communications.
</ul>
<h4><b>Client side setup</b></h4>

Now that the server-side is completely set up, the client-side system needs a way to process and respond to data being sent to it. This has two, Client Creation and Polling.

Client Creation is how you will define seperate clients. This mainly serves to give basic information (username, IP, etc.) about the client simply. Each client needs to be unique, however, so there must be some way to differentiate between every client to make sure that no client objects are interpreted as "equal" when they are infact two seperate clients. This will lead to data being sent to the incorrect client and mismatched operation queues.

For this example, we can use a simple Client class definition, which contains an entry for MAC Address (to ensure uniqueness between clients), hostname, and active username. This will ensure that every client connecting from seperate machines will have unique sessions dedicated to them.

```
	public Client buildClient()
        {
            Client client = new Client();
            String addr = (
                from nic in NetworkInterface.GetAllNetworkInterfaces()
                where nic.OperationalStatus == OperationalStatus.Up
                select nic.GetPhysicalAddress().ToString()
            ).FirstOrDefault();
            client.hostname = Environment.MachineName;
            client.username = Environment.UserName;
            return client;
        }
```

Now that the Client Creation stage is finished, a polling system must be setup. Using WCF, the polling process is actually quite simple. The DataTransfer object used below is what is packaged server-side and sent to the client. This object consists of a ClientOperation, a hasData flag, a dataName, and a data object (List<byte[]>). This allows the server to send data to the client for processing. The dataName is simply where the data object (if present, which is checkable using the hasData flag) will be stored when it is sent back to the server in the ClientDatastore. This value is important for making sure the server-side listeners are triggered when the data is sent back. Since data stored in the data object MUST be a byte[], the readFile path sent from the server earlier encoding the path string as bytes. These bytes are stored in the first entry of the obj.data list, and are decoded using the Encoding.ASCII.GetString() function.

```
        using (ServiceClient proxy = new ServiceClient())
        {
        	while (true)
        	{
        		Thread.Sleep(50); //polling frequency
        		DataTransfer ret = proxy.getWork(client);
        		ClientOperations operation = ret.operation;
                if (operation == ClientOperations.readFile)
                {
                    proxy.sendWork(client, ret.dataName, File.ReadAllBytes(Encoding.ASCII.GetString(obj.data[0]));
                }
            }
        }
```

Adding extra functions is as easy as adding entries to the ClientOperations enum and implementing the client-side work in the polling loop.

Example of full remote file manager using this system.

(https://i.imgur.com/WgjYhK7.png)

