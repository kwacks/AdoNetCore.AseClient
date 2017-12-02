# AdoNetCore.AseClient - a .NET Core DB Provider for SAP ASE

Let's face it, accessing SAP (formerly Sybase) ASE from ADO.NET isn't great. The current .NET 4 version of the vendor's AseClient driver is a .NET Framework managed wrapper around SAP's unmanged [OLE DB provider](https://en.wikipedia.org/wiki/OLE_DB_provider) and is dependent upon [COM](https://en.wikipedia.org/wiki/Component_Object_Model). OLE DB and COM are Windows-only technologies and will never be available to .NET Core. 

Under the hood, ASE (and Microsoft Sql Server for that matter) use an application layer protocol called [Tabular Data Stream](https://en.wikipedia.org/wiki/Tabular_Data_Stream) to transfer data between the database server and the client application. ASE uses TDS 5.0.

This project provides a .NET Core native implementation of the TDS 5.0 protocol via an ADO.NET DB Provider, making SAP ASE accessible from .NET Core applications hosted on Windows, Linux, Docker and also serverless platforms like [AWS Lambda](https://aws.amazon.com/lambda/).

## Objectives
* Functional parity (eventually) with the `Sybase.AdoNet4.AseClient` provided by SAP. The following types will be supported:
    * AseClientFactory	- TODO
    * AseCommand - in progress
    * AseConnection - in progress
    * AseDataParameter - in progress
    * AseDataParameterCollection - in progress
    * AseDataReader - in progress
    * AseException - in progress
* Performance equivalent to or better than that of `Sybase.AdoNet4.AseClient` provided by SAP. This should be possible as we are eliminating the COM and OLE DB layers from this driver.
* Target all versions of .NET Core (1.0, 1.1, 2.0, and 2.1 when it is released)
* Should work with [Dapper](https://github.com/StackExchange/Dapper) at least as well as the .NET 4 client

## Note:
In theory, since we're implementing TDS 5.0, this client might work with other Sybase-produced databases, but the scope for now is just ASE.

## Suggested dev reference material
* [TDS 5.0 Functional Specification Version 3.8](http://www.sybase.com/content/1040983/Sybase-tds38-102306.pdf)
  * This spec is fairly complete, but it's got a few ??? entries -- if you can find a newer version of the spec to use, let its existence be known.
* `Sybase.AdoNet4.AseClient` wireshark packet captures
* `jTDS` if the above isn't informative enough (credit to them for figuring this out)

## Connection strings
[connectionstrings.com](https://www.connectionstrings.com/sybase-adaptive/) lists the following connection string properties for the ASE ADO.NET Data Provider. We aim to use identical connection string syntax to the SAP client, however our support for the various properties will be limited. Our support is as follows:
* `AlternateServers` - not supported.
* `ApplicationName` - supported.
* `BufferCacheSize` - not supported.
* `Charset` - supported.
* `ClientHostName` - supported.
* `ClientHostProc` - supported.
* `CodePageType` - not supported.
* `Connection Lifetime` - not supported.
* `ConnectionIdleTimeout` - not supported.
* `CumulativeRecordCount` - not supported.
* `Database` - supported.
* `Data Source` - supported.
* `DistributedTransactionProtocol` - not supported.
* `DSURL` - not supported.
* `EnableBulkLoad` - not supported.
* `EnableServerPacketSize` - not supported.
* `Encryption` - not supported.
* `EncryptPassword` - not supported.
* `Enlist` - not supported.
* `FetchArraySize` - not supported.
* `HASession` - not supported.
* `LoginTimeOut` - not supported.
* `Max Pool Size` - supported.
* `Min Pool Size` - supported.
* `PacketSize` - not supported.
* `Ping Server` - not supported.
* `Pooling` - supported.
* `Port` - supported.
* `Pwd` - supported.
* `RestrictMaximum PacketSize` - not supported.
* `Secondary Data Source` - not supported.
* `Secondary Server Port` - not supported.
* `TextSize` - not supported.
* `TightlyCoupledTransaction` - not supported.
* `TrustedFile` - not supported.
* `Uid` - supported.
* `UseAseDecimal` - not supported.
* `UseCursor` - not supported.

## Flows/design
Roughly the flows will be (names not set in stone):

### Open a connection
`AseConnection` -Connection Request-> `ConnectionPoolManager` -Request-> `ConnectionPool` *"existing connection is grabbed, or new connection is created"*
```C#
using(var connection = new AseConnection("Data Source=myASEserver;Port=5000;Database=myDataBase;Uid=myUsername;Pwd=myPassword;")) 
{
    // use the connection...
}
```
`AseConnection` <-InternalConnection- `ConnectionPoolManager` <-InternalConnection- `ConnectionPool`

### Send a command and receive any response data
`AseCommand` -ADO.net stuff-> `InternalConnection` -Tokens-> `MemoryStream` -bytes-> `PacketChunkerSocket` *"command gets processed"*

`AseCommand` <-ADO.net stuff- `InternalConnection` <-Tokens- `MemoryStream` <-bytes- `PacketChunkerSocket`

### Release the connection (dispose)
`AseConnection` -Connection-> `ConnectionPoolManager` -Connection-> `ConnectionPool` *"connection released"*

## Plan
In general, for reasons of unit-testing, please create and implement interfaces.

* Setup project structure / files
* Implement ADO.net interfaces
* Connection string parsing
* Structure internal connections / pool management
  * Be wary of `USE DATABASE` calls, these are expensive and not necessary if the connection is already using the desired database.
* Introduce tokens and types by implementing:
  * Login / capability negotiation
  * Simple sql command call (`create procedure ...`)
  * Stored procedure call (`TDS_DBRPC`)
  * Simple queries (`select 1 as x...` with different types)
  * More advanced queries and procedure calls (i.e. with more parameters/types)
