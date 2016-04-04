# Picton

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](http://jericho.mit-license.org/)
[![Build status](https://ci.appveyor.com/api/projects/status/2tl8wuancvf3awap?svg=true)](https://ci.appveyor.com/project/Jericho/picton)
[![Coverage Status](https://coveralls.io/repos/github/Jericho/Picton/badge.svg?branch=master)](https://coveralls.io/github/Jericho/Picton?branch=master)

## About

Picton is a C# library client containing a high performance worker role designed to process messages from an Azure storage queue as efficiently as possible.

I created Picton because I needed a way to process messages from Azure storge queues in the most efficient way. I searched for a long time, but I could never find a solution that met all my requirements.

In March 2016 I attended three webinars by [Daniel Marbach](https://github.com/danielmarbach) on "Async/Await & Task Parallel Library" ([part 1](https://github.com/danielmarbach/02-25-2016-AsyncWebinar), [part 2](https://github.com/danielmarbach/03-03-2016-AsyncWebinar) and [part 3](https://github.com/danielmarbach/03-10-2016-AsyncWebinar)).
Part 2 was particularly interresting to me because Daniel presented a generic message processor (also known as a message "pump") that met most (but not all) of my requirements. Specifically, Daniel's message pump meets the following criteria:

- Concurrent message handling. This means that multiple messages can be processed at the same time. This is critical, especially if your message processing logic is I/O bound.
- Limiting concurrency. This means tht we can decide the maximum number of messages that can be processed at the same time.
- Cancelling and graceful shutdown. This means that our message pump can be notified that we want to stop processing additional messages and also we can decide what to do with messages that are being processed at the moment when we decide to stop.

The sample code that Daniel shared during his webinars was very generic and not specific to Azure so I made the following enhancements:
- The message pump can run as a WorkerRole (it derives from RoleEntryPoint)
- The messages are fetched from an Azure storage queue. We can specify which queue and it can even be a queue in the storage emulator.
- Backoff. When no message is found in the queue, it's important to scale back and query the queue less often. As soon as new messages are available in the queue, we want to scale back up in order to process these messages as quickly as possible. I made two improvements to implemented this logic: 
  - first, introduce a pause when no message is found in the queue in order to reduce the number of queries to the storage queue. By default we pause for one second, but it's configurable. You could even eliminate the pause altogether but I do not recommend it. 
  - second, reduce the number of concurrent message processing tasks. This is the most important improvement introduced in the Picton library (in my humble opinion!). Daniel's sample code uses ``SemaphoreSlim`` to act as a "gatekeeper" and to limit the number of tasks that be be executed concurently. However, the number of "slots" permitted by the semaphore must be predetermined and is fixed. Picton eliminates this restriction and allows this number to be dynamically increased and decreased based on the presence or abscence of messages in the queue.


## Nuget

Picton is available as a Nuget package.

[![NuGet Version](http://img.shields.io/nuget/v/Picton.svg)](https://www.nuget.org/packages/Picton/)
[![AppVeyor](https://img.shields.io/appveyor/ci/Jericho/picton.svg)](https://ci.appveyor.com/project/Jericho/picton)

## Release Notes

+ **1.0**
	- Initial release


## Installation

The easiest way to include Picton in your C# project is by grabing the nuget package:

```
PM> Install-Package Picton
```

Once you have the Picton library properly referenced in your project, add a new class like tis example:

```csharp
using Microsoft.WindowsAzure.Storage;
using Microsoft.WindowsAzure.Storage.Queue;
using Microsoft.WindowsAzure.Storage.RetryPolicies;
using Picton.Azure.WorkerRoles;

namespace MyNamespace
{
	public class MyQueueWorker : AsyncQueueWorker
		: base("MyWorker", 1, 25, TimeSpan.FromMilliseconds(500), 5)
	{
		public override CloudQueue GetQueue()
		{
			var storageAccount = CloudStorageAccount.DevelopmentStorageAccount;
			var cloudQueueClient = storageAccount.CreateCloudQueueClient();
			cloudQueueClient.DefaultRequestOptions.RetryPolicy = new NoRetry();
			var cloudQueue = cloudQueueClient.GetQueueReference("myqueue");
			cloudQueue.CreateIfNotExists();
			return cloudQueue;
		}
		
		public override void OnMessage(CloudQueueMessage message, CancellationToken cancellationToken = default(CancellationToken))
		{
			Debug.WriteLine(msg.AsString);
		}
		
		public override void OnError(CloudQueueMessage message, Exception exception, bool isPoison)
		{
			Trace.TraceInformation("'{0}'.OnError: {1}", this.WorkerName, exception);
			
			if (isPoison)
			{
				// Copy message to a poison queue otherwise it will be lost forever
			}
		}

	}
}
```
