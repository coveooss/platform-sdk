# Coveo C# Platform SDK
[![NuGet version (Coveo.Connectors.Utilities.PlatformSdk)](https://img.shields.io/nuget/v/Coveo.Connectors.Utilities.PlatformSdk.svg?style=flat-square)](https://www.nuget.org/packages/Coveo.Connectors.Utilities.PlatformSdk/)

The official Coveo C# SDK allows your software to communicate with the Coveo Cloud Platform public APIs and the Push API.

The package is .NET Standard 2.0 compatible and therefore works with .NET Core and .NET Framework

The code of the SDK is not open-source. This repository is meant to provide feedback, bug reporting, etc.

## Installation
Install the public NuGet package [Coveo.Connectors.Utilities.PlatformSdk](https://www.nuget.org/packages/Coveo.Connectors.Utilities.PlatformSdk/) in your C# project using your favorite IDE or NuGet command line.

The package is signed before being published to NuGet to avoid pulling a malicious package.
We provide the `PDBs` to have a better debugging experience.
Documentation is available for most of public members and methods.

## How to use the SDK
First, you need to instantiate the client to interact with the platform. Here is the minimum configuration you need to provide:


```csharp
string apiKey = "Your api key with required rights";
string organizationId = "You organization ID";
ICoveoPlatformConfig config = new CoveoPlatformConfig(apiKey, organizationId);
ICoveoPlatformClient client = new CoveoPlatformClient(config);
```

By default, the package target the US production environment (platform.cloud.coveo.com). If you want to target the US HIPAA environment (platformhipaa.cloud.coveo.com) you can do it that way:

```csharp
string apiKey = "Your api key with required rights";
string organizationId = "You organization ID";
ICoveoPlatformConfig config = new CoveoPlatformConfig(Constants.Endpoint.UsEast1.HIPAA_PUSH_API_URL,
    Constants.PlatformEndpoint.UsEast1.HIPAA_PLATFORM_API_URL,
    apiKey,
    organizationId);
ICoveoPlatformClient client = new CoveoPlatformClient(config);
```

## Using the SDK to push documents to a Push API source
Methods to interact with the Push API are part of `client.DocumentManager` object.

### Pushing a batch of documents
For overall performance, it is better to push your documents in batch. Use the single document method when batches are not required to meet both your performance and volume requirement. For more information, visit [Managing Batches of Items in a Push Source](https://docs.coveo.com/en/90/cloud-v2-developers/managing-batches-of-items-in-a-push-source).

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
You should only use this method when you want to add or update a single document. Pushing several documents using this method may lead to the `429 - Too Many Requests` response from the Coveo platform and decrease the overall performance time of your application.

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
* The `SetContent` method has an overload taking a Stream instead of a string.
* The PushDocumentHelper class has also the method `SetContentFromFile` taking a file path.
* The third argument in `AddOrUpdateDocument` is the ordering ID. If you don't provide a value, the SDK will create one using a timestamp to ensure the changes are performed in the order they were received. For more information about ordering ID, visit [Understanding the orderingId Parameter](https://docs.coveo.com/en/147/cloud-v2-developers/understanding-the-orderingid-parameter).
* The call returns the generated ordering ID if you did not specify one. You can store it. It can be useful to delete a batch of documents.

### Pushing a document with big properties
In case the size of the document you just created is too big, the SDK does not handle it if you use the call to index a single document. Take note that if you always use the batched call the SDK will handle it automatically.
In that case you have 2 options:
* You can add the document using the batched call `client.DocumentManager.AddOrUpdateDocuments`
* You can also check for the document size and then decide if you use `client.DocumentManager.AddOrUpdateDocument` or `client.DocumentManager.AddOrUpdateDocuments`
```csharp
document.JsonObjectSize > Constants.COMPRESSED_DATA_MAX_SIZE_IN_BYTES
```

### Pushing a document with a large binary data
The Push API has a hard limit of 5MB (compressed) for the document's binary data. When the data exceeds that value we need to request an upload URI to an S3 bucket to put the document's binary data in it and refer to that ID when pushing the document in the Push API. However, lucky you, the SDK handles this automatically. Thus, the SDK compresses the content, and if it exceeds 5MB (compressed), it will do the needed logic. Take note that the maximum size of a document is 256MB (compressed). 

### Delete a single document
```csharp
string documentId = "https://coveo.com";
client.DocumentManager.DeleteDocument(sourceId, documentId, null);
```
**Good to know:**
* Prefer deleting documents in batches to save API calls and to have a better application performance.

### Delete a batch of documents
```csharp
IList<string> documentsIdstoDelete = new List<string> {
    "https://coveo.com",
    "https://coveo.com/a page"
};
client.DocumentManager.DeleteDocuments(sourceId, documentsIdstoDelete, null);
```

### Delete a specific document and its children
You can delete a specific document and all its children easily. For more information about deleting a document and its children, visit [Deleting an Item and Optionally, its Children in a Push Source](https://docs.coveo.com/en/171/cloud-v2-developers/deleting-an-item-and-optionally-its-children-in-a-push-source)

In this example, imagine you have added a document with an ID of `http://coveo.com/parent` and another document with an ID of `http://coveo.com/parent/child`. Clearly, it this example, the second document is the child of the former. To delete a document and all its children, it is pretty easy:
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

## Adding securities to your documents
You can add securities to documents, so only allowed users or groups can view the document. To learn how to format your permissions, see [Push API Tutorial 2 - Managing Secured Content](https://docs.coveo.com/en/98/cloud-v2-developers/push-api-tutorial-2---managing-secured-content).

### Prerequisites
1. Create or update your source to be secured.
1. Ensure your API key has access to create security provider.
1. Create an `Expanded` security provider that cascades to `Email Security Provider` and link it to your source. Here is an example using the SDK:
```csharp
string expandedProviderId = "The unique name you want";
client.SecurityProviderManager.AddOrUpdateExpandedProviderAssociatedToEmailProvider(expandedProviderId, new List<string> { sourceId }, false);
```
**Good to know:**
* The third argument of `AddOrUpdateExpandedProviderAssociatedToEmailProvider` is whether the provider is case-sensitive or not. If false, `acme\jdoe` is the same as `acme\JDOE`.

### Add simple permission to the document
In this example, we add a document with a simple permissions model. It means that we set the allowed and denied users directly on the document. For more information, visit [Simple Permission Model Definition Examples](https://docs.coveo.com/en/107/cloud-v2-developers/simple-permission-model-definition-examples)
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

### Push identities in batch
If possible, you should push the identity in batch.
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

### Disable a single identity
You can easily disable an identity. For more information, visit [Disabling a Single Security Identity](https://docs.coveo.com/en/84/cloud-v2-developers/disabling-a-single-security-identity).
```csharp
client.PermissionManager.DeleteIdentity(expandedProviderId, new PermissionIdentity(@"acme\johndoe", PermissionIdentityType.User));
```

### Disable identities in batch
```csharp
IList<PermissionIdentity> identitiesToDelete = new List<PermissionIdentity> {
    new PermissionIdentity(@"acme\member1", PermissionIdentityType.User),
    new PermissionIdentity(@"acme\member2", PermissionIdentityType.User)
};

client.PermissionManager.DeleteIdentities(expandedProviderId, null, identitiesToDelete);
```

### Disable identities older than a specific ordering ID
Same as the documents, you can disable identities that have an ordering ID smaller than the one you provide. For more information, visit [Disabling Old Security Identities](https://docs.coveo.com/en/33/cloud-v2-developers/disabling-old-security-identities).
```csharp
client.PermissionManager.AddOrUpdateIdentity(expandedProviderId, 100, new PermissionIdentityBody(new PermissionIdentity(@"acme\tobedeleted3", PermissionIdentityType.User)));
client.PermissionManager.AddOrUpdateIdentity(expandedProviderId, 200, new PermissionIdentityBody(new PermissionIdentity(@"acme\tobedeleted4", PermissionIdentityType.User)));

// Wait a little bit, then disable all identities with an ordering ID less than 300.
client.PermissionManager.DeleteIdentitiesOlderThan(expandedProviderId, 300);
```

**Good to know:**
* As for `DeleteDocumentsOlderThan`, there is a processing delay. However, it is not configurable for this call.

### Add complex permission to a document
The permission model of your system might be more complicated, thus, simple permission might not be enough to secure your documents. Below is an example of having two levels. One for the `Administrator` of the system and the other one for standard users. For more information, visit [Complex Permission Model Definition Example](https://docs.coveo.com/en/25/cloud-v2-developers/complex-permission-model-definition-example).
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

Then, you can push your identities as we did before.
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

// Push user mappings and group members in one batched operation.
client.PermissionManager.AddOrUpdateIdentities(expandedProviderId, null, mappingsMembersToAddOrUpdate);
```

## Activate logging
The SDK uses `log4net` as its logging library. It can be useful to have some logs in case of a problem. In your log4net configuration, add a logger with `Coveo` as the name. We use namespaces as logger name. Using `Coveo` as the logger name will get you all the logs from the SDK. 

**Good to know:**
* If you activate `Trace` level, you will get information about requests made by the SDK.

## Explore!
Feel free to explore the other `Managers` that the `ICoveoPlatformClient` provides to you. Don't hesitate to provides us feedback, report bugs if any, and ask for a feature!
