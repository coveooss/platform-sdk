# Coveo C# Platform SDK
[![NuGet version (Coveo.Connectors.Utilities.PlatformSdk)](https://img.shields.io/nuget/v/Coveo.Connectors.Utilities.PlatformSdk.svg?style=flat-square)](https://www.nuget.org/packages/Coveo.Connectors.Utilities.PlatformSdk/)

The official Coveo C# SDK allows your software to communicate with the Coveo Cloud Platform public APIs and the Push API.

The package is .NET Standard 2.0 compatible and therefore works with .NET Core and .NET Framework

The code of the SDK is not open-source. This repository is meant to handle feedback, bug reporting, etc.

## Installation
Install the public NuGet package [Coveo.Connectors.Utilities.PlatformSdk](https://www.nuget.org/packages/Coveo.Connectors.Utilities.PlatformSdk/) in your C# project using your favorite IDE or NuGet command line.

The package is signed before being published to NuGet to avoid pulling a malicious package.
We provide the `PDBs` to have a better debugging experience.
The code documentation is generated and is available for most of the public members and methods.

## How to use the SDK
First, you need to instantiate the client to interact with the platform. Here is the minimum configuration you need to provide:

**Each section below will redirect you to the privileges needed for the requests to work. For more information about API key privileges, visit [Privilege Reference](https://docs.coveo.com/en/1707/cloud-v2-administrators/privilege-reference).**

```csharp
string apiKey = "Your API key with the required privileges";
string organizationId = "Your organization ID";
ICoveoPlatformConfig config = new CoveoPlatformConfig(apiKey, organizationId);
ICoveoPlatformClient client = new CoveoPlatformClient(config);
```

By default, the package targets the US production environment (platform.cloud.coveo.com). If you want to target the US HIPAA environment (platformhipaa.cloud.coveo.com) you can do it this way:

```csharp
string apiKey = "Your API key with the required privileges";
string organizationId = "Your organization ID";
ICoveoPlatformConfig config = new CoveoPlatformConfig(Constants.Endpoint.UsEast1.HIPAA_PUSH_API_URL,
    Constants.PlatformEndpoint.UsEast1.HIPAA_PLATFORM_API_URL,
    apiKey,
    organizationId);
ICoveoPlatformClient client = new CoveoPlatformClient(config);
```

## Using the SDK to push documents to a Push source
Methods to interact with the Push API are part of the `client.DocumentManager` object.
### Prerequisites
1. Ensure your API key has the required privilege to push documents inside a Push API source.
1. You can create an API key when creating a Push API source.
1. You can also create an API key manually. For more information about which privileges are required, visit [Privilege Reference](https://docs.coveo.com/en/1707/cloud-v2-administrators/privilege-reference#sources-domain).
1. [Create a Push API source.](https://docs.coveo.com/en/1546/cloud-v2-administrators/add-or-edit-a-push-source)
### Pushing a batch of documents
For overall performance, it is better to push your documents in batches. Use the single document method when batches are not required to meet both your performance and volume requirements. For more information, visit [Managing Batches of Items in a Push Source](https://docs.coveo.com/en/90/cloud-v2-developers/managing-batches-of-items-in-a-push-source).

```csharp
PushDocument firstDocumentToAdd = new PushDocument("http://www.coveo.com/page1") {
    ClickableUri = "http://www.coveo.com/page1",
    ModifiedDate = DateTime.UtcNow
};

PushDocument secondDocumentToAdd = new PushDocument("http://www.coveo.com/page2") {
    ClickableUri = "http://www.coveo.com/page2",
    ModifiedDate = DateTime.UtcNow
};

IList<PushDocument> documentsToAdd = new List<PushDocument> {
    firstDocumentToAdd,
    secondDocumentToAdd
};

client.DocumentManager.AddOrUpdateDocuments(sourceId, documentsToAdd, null);
```
### Pushing a single document
You should only use this method when you want to add or update a single document. Pushing several documents using this method may lead to the `429 - Too Many Requests` response from the Coveo platform and decrease the overall performance of your application.

```csharp
string sourceId = "your Push API source ID";
PushDocument document = new PushDocument("https://coveo.com") {
    ClickableUri = "https://www.coveo.com",
    ModifiedDate = DateTime.UtcNow
};

document.AddMetadata("title", "Coveo's home page.");

PushDocumentHelper.SetContent(document, "this is Coveo's website home page.");

client.DocumentManager.AddOrUpdateDocument(sourceId, document, null);
```

**Good to know:**
* The `SetContent` and `SetContentFromFile` put the value in the `data` field of the document. This is meant for small raw textual data. **Pro tip:** You should use the other method in PushDocumentHelper to put data on document for production system. [Using the data Property](https://docs.coveo.com/en/31/cloud-v2-developers/using-the-data-property).
* The `SetContent` method has an overload taking a Stream instead of a string. The stream must be convertible to textual data.
* The PushDocumentHelper class also has a  `SetContentFromFile` method taking a file path as an argument. **Be careful**, this method only works with text file. For binary file (e.g. PDF) use `SetBinaryContentFromFileAndCompress`.
* The third argument in `AddOrUpdateDocument` is the [ordering ID](https://docs.coveo.com/en/147/cloud-v2-developers/understanding-the-orderingid-parameter). If you don't provide a value, the SDK will create one using a timestamp to ensure the changes are performed in the order they were received.
* The call returns the generated ordering ID if you did not specify one. You can store it. It can be useful to delete a batch of documents.

### Pushing a document with large properties
If you use the call to push a single document (`AddOrUpdateDocument`) on a document whose size is too large, you will get an error. If the whole document is larger than 256MB, it will throw an `ArgumentException`. If the document is larger than 5MB, it will throw a `CoveoPlatformException` with status code `500` and `Internal server error` inside the `ErrorMessage` property. On the other hand, if you use the batch call to push one or more documents (`AddOrUpdateDocuments`), the SDK will automatically carry out the operations needed to handle large documents without error, up to 256 mb.
In cases where you want to push a document with large properties:
* You can add the document using the batch call `client.DocumentManager.AddOrUpdateDocuments`
* You can also check for the document size and then decide whether to use `client.DocumentManager.AddOrUpdateDocument` or `client.DocumentManager.AddOrUpdateDocuments`
```csharp
document.JsonObjectSize > Constants.COMPRESSED_DATA_MAX_SIZE_IN_BYTES
```

### Pushing a document with large binary data size
The Push API has a hard limit of 5MB (compressed) on the document's binary data. When the size exceeds that value, you typically need to request an upload URI to an S3 file container. You would then put your document's binary data in that container and refer to the container by ID when pushing the target document with the Push API. However, lucky you, the SDK handles this automatically. Thus, the SDK compresses the content, and if it exceeds 5MB (compressed), it will perform the needed logic. Note that the maximum size of a document is 256MB (compressed).

### Delete a single document
```csharp
string documentId = "https://coveo.com";
client.DocumentManager.DeleteDocument(sourceId, documentId, null);
```
**Good to know:**
* Favor deleting documents in batches to save API calls and to have a better application performance.

### Delete a batch of documents
```csharp
IList<string> documentsIdstoDelete = new List<string> {
    "https://coveo.com",
    "https://coveo.com/a page"
};
client.DocumentManager.DeleteDocuments(sourceId, documentsIdstoDelete, null);
```

### Delete a specific document and its children
You can delete a specific document and all of its children easily. For more information about deleting a document and its children, visit [Deleting an Item and Optionally, its Children in a Push Source](https://docs.coveo.com/en/171/cloud-v2-developers/deleting-an-item-and-optionally-its-children-in-a-push-source)

In this example, imagine you have added a document with an ID of `http://coveo.com/parent` and another document with an ID of `http://coveo.com/parent/child`. Clearly, it this example, the second document is the child of the former. To delete the parent and child documents:
```csharp
string parentDocumentId = "http://coveo.com/parent";
client.DocumentManager.DeleteDocument(sourceId, parentDocumentId, null, true);
```

### Delete documents older than a specific ordering ID
Remember the ordering ID? You can send a request to your Push API source to delete all documents that were pushed with an ordering ID less than the one you provide.
```csharp
ulong orderingId = 12345; // Every document in the source that has an ordering lower than 12345 will be deleted.
client.DocumentManager.DeleteDocumentsOlderThan(sourceId, orderingId, null);
```
**Good to know:**
* The third argument in `DeleteDocumentsOlderThan` is the processing delay. When passing `null`, it uses the default value of 15 minutes. For more information about processing delay, visit [QueueDelay](https://docs.coveo.com/en/131/cloud-v2-developers/deleting-old-items-in-a-push-source).

### Combine delete and add
You can combine a batch of documents to be deleted and a batch of documents to be added in the same call.
```csharp
PushDocument firstDocumentToAdd = new PushDocument("http://www.coveo.com/page/child") {
    ClickableUri = "http://www.coveo.com/page/child",
    ModifiedDate = DateTime.UtcNow
};

PushDocument secondDocumentToAdd = new PushDocument("http://www.coveo.com/page/child/child") {
    ClickableUri = "http://www.coveo.com/page/child/child",
    ModifiedDate = DateTime.UtcNow
};

IList<PushDocument> documentsToAdd = new List<PushDocument> {
    firstDocumentToAdd,
    secondDocumentToAdd
};

IList<string> documentsToDelete = new List<string> {
    "http://coveo.com/parent/child",
    "http://coveo.com/parent"
};

client.DocumentManager.AddOrUpdateDocuments(sourceId, documentsToAdd, documentsToDelete, null);
```

## Using the SDK to push documents to the Stream API

Methods to interact with the Stream API are part of the `client.DocumentManager` object. It’s a different mode of operation from the Push API, but very similar, with several more calls.

### Prerequisites:

* Before you proceed with this method, ensure you’ve understood how to [create a commerce catalog](https://docs.coveo.com/en/3139/coveo-for-commerce/create-a-coveo-commerce-catalog) and how to [index data with the Stream API](https://docs.coveo.com/en/2956/coveo-for-commerce/index-commerce-catalog-content-with-the-stream-api).
* For additional information and payload examples, see [How to Stream Your Catalog Data to Your Source](https://docs.coveo.com/en/lb4a0344/coveo-for-commerce/how-to-stream-your-catalog-data-to-your-source).
* Create the [catalog source](https://docs.coveo.com/en/3295/index-content/add-or-edit-a-catalog-source) first in the Coveo Administration Console and get an API key to provide to the `CoveoPlatformConfig`.
* To enable the use of the Stream API, create your `CoveoPlatformConfig` with the `useStreamApi` parameter set to `true`:

```
new CoveoPlatformConfig(Constants.Endpoint.UsEast1.PROD_PUSH_API_URL, Constants.PlatformEndpoint.UsEast1.PROD_PLATFORM_API_URL, apiKey, organizationId, true)
```

### How the Stream API works

The Stream API works in two ways:

* Stream mode (a full rebuild)
* Update mode (an incremental refresh)

### Call wrappers for the Stream API

Call wrappers for the Stream API are found in the `StreamApiDocumentServiceManager` class. As the class inherits from the `DocumentServiceManager` class, the only truly new public calls are:

* `OpenDocumentStream(string sourceId)` to open a stream.
* `GetNewChunkForStream(string sourceId)` to get a new stream chunk. Normally, this method won't need to be called because the SDK will automatically get a new chunk before each batch upload.
* `CloseDocumentStream(string sourceId)` to close an open stream.

Adding and deleting documents will call the base methods, but with some Stream API-specific details in the implementation.

### Opening a stream

> :warning: When you open and close a stream, all previous files not indexed in the current operation will be removed from the index. Updating individual documents should be done in **update mode**, without the need to open a stream and close it afterwards.

```
client.DocumentManager.OpenDocumentStream(sourceId)
```

This will save a `streamId` in the `client.DocumentManager` instance.

### Adding or updating documents

The below calls use the call to the `AddOrUpdateDocuments` method that is in the `DocumentServiceManager` class. The presence of a `streamId` in the `client.DocumentManager` instance will determine if the documents are uploaded in **stream mode** using the `/chunk` endpoint, or uploaded in **update mode** using the `/files` endpoint. Batching documents is strongly recommended and can be accomplished by calling `AddOrUpdateDocuments` for each batch of documents under 256 MB.

**Adding a batch of documents**

Create Push Documents from your catalog items and put them into a `List<PushDocument>`, similar to what is done when pushing a batch of documents with the Push API.
```
client.DocumentManager.AddOrUpdateDocuments(sourceId, documentsToAddOrUpdate, null)
```

**Adding a single document**
```
client.DocumentManager.AddOrUpdateDocument(sourceId, pushDocument, null)
```
**Good to know:**
* The Stream API has the same limits as the Push API regarding document size and number of calls. Adding several documents using `AddOrUpdateDocument` could result in a `429 - Too Many Requests` response from the Coveo platform.
* There shouldn’t be a need to manually call `GetNewChunkForStream`, as in **stream mode** each new call to `AddOrUpdateDocuments` will also call `GetNewChunkForStream` first.
* In **update mode**, the call to the `/update` endpoint is done automatically after uploading each batch.

### Deleting multiple documents
```
IList<string> documentsIdstoDelete = new List<string> {
    "https://coveo.com",
    "https://coveo.com/a page"
};
client.DocumentManager.DeleteDocuments(sourceId, documentsIdstoDelete, null)
```
### Deleting a single document
```
string documentId = "https://coveo.com";
client.DocumentManager.DeleteDocument(sourceId, documentId, null)
```
**Good to know:**
* Both of these calls actually call the `AddOrUpdateDocuments` method (if there is only one document, it will be put into a list first) and will use the **update mode**.
* If a stream is already open, it will be closed before `AddOrUpdateDocuments` is called.
* Each of these calls will be followed by a call to the `/update` endpoint.

### Closing a Stream
```
client.DocumentManager.CloseDocumentStream(sourceId)
```

## Adding permissions to your documents
You can add permissions to documents, so only allowed users or groups can view the document. To learn how to format your permissions, see [Push API Tutorial 2 - Managing Secured Content](https://docs.coveo.com/en/98/cloud-v2-developers/push-api-tutorial-2---managing-secured-content).

### Prerequisites
1. Ensure your [Push source is secured](https://docs.coveo.com/en/98/#step-1-configure-a-secured-push-source)
1. Ensure your API key has the privileges required to create a security identity provider. For more information about which privileges are required, visit [Privilege Reference](https://docs.coveo.com/en/1707/cloud-v2-administrators/privilege-reference#security-identities-domain).
1. Create an `Expanded` security provider that cascades to `Email Security Provider` and link it to your source. Here is an example using the SDK:
```csharp
string expandedProviderId = "The unique name you want";
client.SecurityProviderManager.AddOrUpdateExpandedProviderAssociatedToEmailProvider(expandedProviderId, new List<string> { sourceId }, false);
```
**Good to know:**
* The third argument of `AddOrUpdateExpandedProviderAssociatedToEmailProvider` determines whether the provider is case-sensitive or not. If false, `acme\jdoe` is the same as `acme\JDOE`.

### Add simple permissions to the document
In this example, we add a document with a simple permission model. I.e., we set the allowed and denied users directly on the document. For more information, visit [Simple Permission Model Definition Examples](https://docs.coveo.com/en/107/cloud-v2-developers/simple-permission-model-definition-examples)
```csharp
PushDocument document = new PushDocument("http://www.coveo.com/secured") {
    ClickableUri = "http://www.coveo.com/secured",
    ModifiedDate = DateTime.UtcNow
};

// Push the document with the security associated to it.
PermissionIdentity user = new PermissionIdentity(@"acme\johndoe", PermissionIdentityType.User);
PermissionIdentity group = new PermissionIdentity(@"acme\team", PermissionIdentityType.Group);

document.SetSimpleAllowedAndDeniedPermissions(new List<PermissionIdentity>{ user, group }, new List<PermissionIdentity>());

client.DocumentManager.AddOrUpdateDocument(sourceId, document, null);

// Push the permission mapping for a user from Active Directory to email.
PermissionIdentityBody userBody = new PermissionIdentityBody(user);
userBody.Mappings.Add(new PermissionIdentity("johndoe@acme.com", PermissionIdentityType.User));

// Push the permission mapping for a group from Active Directory to email.
PermissionIdentityBody groupBody = new PermissionIdentityBody(group);
groupBody.Mappings.Add(new PermissionIdentity("teammember1@team.acme.com", PermissionIdentityType.User));
groupBody.Mappings.Add(new PermissionIdentity("teammember2@team.acme.com", PermissionIdentityType.User));

client.PermissionManager.AddOrUpdateIdentity(expandedProviderId, null, userBody);
client.PermissionManager.AddOrUpdateIdentity(expandedProviderId, null, groupBody);
```

### Push a batch of identities
For overall performance, it is better to push your identities in batches.
```csharp
PermissionIdentity member1 = new PermissionIdentity(@"acme\member1", PermissionIdentityType.User);
PermissionIdentity member2 = new PermissionIdentity(@"acme\member2", PermissionIdentityType.User);

PermissionIdentityBody bodyMember1 = new PermissionIdentityBody(member1);
bodyMember1.Mappings.Add(new PermissionIdentity("member1@acme.com", PermissionIdentityType.User));

PermissionIdentityBody bodyMember2 = new PermissionIdentityBody(member2);
bodyMember2.Mappings.Add(new PermissionIdentity("member2@acme.com", PermissionIdentityType.User));

List<PermissionIdentityBody> mappingsMembersToAddOrUpdate = new List<PermissionIdentityBody> {
    bodyMember1,
    bodyMember2
};

List<PermissionIdentityBody> membersToAddOrUpdate = new List<PermissionIdentityBody> {
    new PermissionIdentityBody(new PermissionIdentity(@"acme\team2", PermissionIdentityType.Group)) {
        Members = new List<PermissionIdentity> {
            member1,
            member2
        }
    }
};

// Push user mappings and group members in one batched operation.
client.PermissionManager.AddOrUpdateIdentities(expandedProviderId, null, mappingsMembersToAddOrUpdate.Concat(membersToAddOrUpdate).ToList());
```

### Disable a single security identity
You can easily disable an identity. For more information, visit [Disabling a Single Security Identity](https://docs.coveo.com/en/84/cloud-v2-developers/disabling-a-single-security-identity).
```csharp
client.PermissionManager.DeleteIdentity(expandedProviderId, new PermissionIdentity(@"acme\johndoe", PermissionIdentityType.User));
```

### Disable a batch of identities
```csharp
IList<PermissionIdentity> identitiesToDelete = new List<PermissionIdentity> {
    new PermissionIdentity(@"acme\member1", PermissionIdentityType.User),
    new PermissionIdentity(@"acme\member2", PermissionIdentityType.User)
};

client.PermissionManager.DeleteIdentities(expandedProviderId, null, identitiesToDelete);
```

### Disable identities older than a specific ordering ID
Same as with documents, you can disable identities that have an ordering ID smaller than the one you provide. For more information, visit [Disabling Old Security Identities](https://docs.coveo.com/en/33/cloud-v2-developers/disabling-old-security-identities).
```csharp
client.PermissionManager.AddOrUpdateIdentity(expandedProviderId, 100, new PermissionIdentityBody(new PermissionIdentity(@"acme\tobedeleted3", PermissionIdentityType.User)));
client.PermissionManager.AddOrUpdateIdentity(expandedProviderId, 200, new PermissionIdentityBody(new PermissionIdentity(@"acme\tobedeleted4", PermissionIdentityType.User)));

// Wait a little bit, then disable all identities with an ordering ID less than 300.
client.PermissionManager.DeleteIdentitiesOlderThan(expandedProviderId, 300);
```
**Good to know:**
* As for `DeleteDocumentsOlderThan`, there is a processing delay. However, it is not configurable for this call. For more information about processing delay, visit [QueueDelay](https://docs.coveo.com/en/131/cloud-v2-developers/deleting-old-items-in-a-push-source).

### Add complex permissions to a document
The permission model of your system might be more complicated, thus, simple permissions might not be enough to secure your documents. Below is an example of a two-level permission model. One for the `Administrator` of the system and the other one for standard users. For more information, visit [Complex Permission Model Definition Example](https://docs.coveo.com/en/25/cloud-v2-developers/complex-permission-model-definition-example).
```csharp
PushDocument verySecureDocument = new PushDocument("http://www.coveo.com/verysecure") {
    ClickableUri = "http://www.coveo.com/verysecure",
    ModifiedDate = DateTime.UtcNow
};

DocumentPermissionSet adminSet = new DocumentPermissionSet {
    AllowedPermissions = new List<PermissionIdentity> {
        new PermissionIdentity(@"acme\admin", PermissionIdentityType.User)
    }
};

DocumentPermissionLevel adminLevel = new DocumentPermissionLevel {
    PermissionSets = new List<DocumentPermissionSet> { adminSet }
};

DocumentPermissionSet usersSet = new DocumentPermissionSet {
    AllowedPermissions = new List<PermissionIdentity> {
        new PermissionIdentity(@"acme\john", PermissionIdentityType.User)
    },
    DeniedPermissions = new List<PermissionIdentity> {
        new PermissionIdentity(@"acme\cannotaccess", PermissionIdentityType.User)
    }
};

DocumentPermissionLevel userLevel = new DocumentPermissionLevel {
    PermissionSets = new List<DocumentPermissionSet> { usersSet }
};

verySecureDocument.Permissions.Add(adminLevel);
verySecureDocument.Permissions.Add(userLevel);

client.DocumentManager.AddOrUpdateDocument(sourceId, verySecureDocument, null);
```

Then, you can push your identities as you did before.
```csharp
PermissionIdentity allowedMember = new PermissionIdentity(@"acme\john", PermissionIdentityType.User);
PermissionIdentity deniedMember = new PermissionIdentity(@"acme\cannotaccess", PermissionIdentityType.User);
PermissionIdentity admin = new PermissionIdentity(@"acme\admin", PermissionIdentityType.User);

PermissionIdentityBody bodyAllowedMember = new PermissionIdentityBody(allowedMember);
bodyAllowedMember.Mappings.Add(new PermissionIdentity("john@acme.com", PermissionIdentityType.User));

PermissionIdentityBody bodyDeniedMember = new PermissionIdentityBody(deniedMember);
bodyDeniedMember.Mappings.Add(new PermissionIdentity("cannotaccess@acme.com", PermissionIdentityType.User));

PermissionIdentityBody bodyAdmin = new PermissionIdentityBody(admin);
bodyAdmin.Mappings.Add(new PermissionIdentity("admin@acme.com", PermissionIdentityType.User));

List<PermissionIdentityBody> mappingsMembersToAddOrUpdate = new List<PermissionIdentityBody> {
    bodyAllowedMember,
    bodyDeniedMember,
    bodyAdmin
};

// Push user mappings and group members in one batch operation.
client.PermissionManager.AddOrUpdateIdentities(expandedProviderId, null, mappingsMembersToAddOrUpdate);
```

## Activate logging
The SDK uses `log4net` as its logging library. It can be useful to have logs in case of problems. In your log4net configuration, add a logger with named `Coveo`. We use namespaces as logger names. Using `Coveo` as the logger name will get you all the logs from the SDK.

**Good to know:**
* If you activate `Trace` level, you will get information about requests made by the SDK.

## Explore!
Feel free to explore the other `Managers` that the `ICoveoPlatformClient` provides to you. Don't hesitate to send us feedback, to report bugs if any, and to ask for a feature!
