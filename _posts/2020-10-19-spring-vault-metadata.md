---
layout: post
title: "open source contribution overview: adding metadata operations for Vault's KV engine to spring-vault"
author: "Zakaria"
comments: true
description: spring-vault metadata operations
---

[spring-vault](https://github.com/spring-projects/spring-vault) is a handy project that provides abstractions for interacting with [Hashicop Vault](https://www.vaultproject.io/) server. It can be used in spring or non-spring based projects that requires vault access, since it provides most of the common operations to perform the different vault operations through the rest API. While working recently with spring-vault, I noticed that the version 2 of vault's `KV` secret engine does not have operations for dealing with metadata in spring-vault and decided, out of a real world project need, to submit a PR, hoping that other devs will find this handy as well. 

# KV version 1 and version 2 differences 

To give a bit of context, let's do a quick comparison between version 1 and 2 of the key-value secret engine. Vault offers two versions of its `KV` secret engine:

 * version 1: is a simple secrets store that keeps only the latest value for a certain key. There no information about history and previous stored values for a key.
 * version 2: adds the possiblity to store several versions of a value for a single key. It keeps the previous versions of a key as well as other important metadata like modification/deletion date. 

Working with version 2 offers the advantage of versioning a key, but also introduces complexity. For example, if one would like to delete a key, he should be aware that the `DELETE` request only deletes the latest value (marks it as deleted). If one would like to completely remove a key, the metadata needs to be removed. 

For example:

if you run `vault kv put secret/test test=test` and then `vault kv get -format=json secret/test`, the response looks like:

__in version 1__:

```
{
  "request_id": "c113982d-af4d-459a-bbbd-f8ff17abb44c",
  "lease_id": "",
  "lease_duration": 2764800,
  "renewable": false,
  "data": {
    "test": "test"
  },
  "warnings": null
}
```

__in version 2__:

```
{
  "request_id": "9b7e1beb-e42b-1460-c82b-4cc8f5d843fa",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "data": {
      "test": "test"
    },
    "metadata": {
      "created_time": "2020-10-19T12:11:32.541367481Z",
      "deletion_time": "",
      "destroyed": false,
      "version": 1
    }
  },
  "warnings": null
}
```

notice the new `metadata` object added in version 2. If we want to delete our `test` key:

`vault kv delete secret/test` 

if we try to read the secret again: 

__in version 1__:

```
No value found at secret/test
```

which implies that the key has been effectively deleted

__in version 2__:

```
{
  "request_id": "e873230c-a588-1795-ee12-e3852668e286",
  "lease_id": "",
  "lease_duration": 0,
  "renewable": false,
  "data": {
    "data": null,
    "metadata": {
      "created_time": "2020-10-19T12:29:06.484006823Z",
      "deletion_time": "2020-10-19T12:29:25.419704601Z",
      "destroyed": false,
      "version": 1
    }
  },
  "warnings": null
}
```

We can notice that the data is still there, except that the key has been been marked as deleted ("deletion_time" has been updated.)

to be able to completely delete the key, we need to execute another call to delete the metadata: `vault kv metadata delete secret/test` 

and now we get the same response as version 1: `No value found at secret/data/test` 

In `spring-vault`, all the operations to interact with the `KV` engine (both version 1 and version 2) are provided through the [VaultKeyValueOperations](https://github.com/spring-projects/spring-vault/blob/master/spring-vault-core/src/main/java/org/springframework/vault/core/VaultKeyValueOperations.java) interface. The missing part for `KV` version 2 was the metadata operations. Therefore one has to directly call the API through a http client, as a workaround. The objective of the contribution was the addition of operations for metadata for version 2. (checkout the issue for more details: [https://github.com/spring-projects/spring-vault/issues/432](https://github.com/spring-projects/spring-vault/issues/432))


# Introducing VaultKeyValueMetadataOperations:

In the patch submitted, a new set of operations to deal with version 2 metadata was introduced, you can find all the details in the pull request: [https://github.com/spring-projects/spring-vault/pull/561/files](https://github.com/spring-projects/spring-vault/pull/561/files)

 {% highlight java  linenos %}
public interface VaultKeyValueMetadataOperations {

  /**
   * permanently deletes the key metadata and all version data for the specified key. All version history will be removed.
   * @param path the secret path, must not be null or empty
   */
  void delete(String path);

  /**
   * retrieves the metadata and versions for the secret at the specified path.
   * @param path the secret path, must not be null or empty
   * @return {@link VaultMetadataResponse}
   */
  VaultMetadataResponse get(String path);

  /**
   * Updates the secret metadata, or creates new metadata if not present.
   *
   * @param path the secret path, must not be null or empty
   * @param body {@link VaultMetadataRequest}
   */
  void put(String path, VaultMetadataRequest body);
}

{% endhighlight %}

# VaultKeyValueMetadataOperations in action:

First let's configure our template: 

 {% highlight java  linenos %}
  @Autowired
  VaultTemplate vaultTemplate;

  VaultVersionedKeyValueTemplate vaultVersionedKeyValueTemplate;

  @BeforeEach
  void setup() {
    vaultVersionedKeyValueTemplate = new VaultVersionedKeyValueTemplate(vaultTemplate, "secret");
  }
{% endhighlight %}

After adding a key to the `secret/test` path, it's now possible to delete it completely using just spring-vault the `VaultKeyValueMetadataOperations` :

 {% highlight java  linenos %}
 @Test
  void deleteMetadata() {
    Map<String, String> data = new HashMap<>();
    data.put("test", "test");
    vaultVersionedKeyValueTemplate.put("test", data);
    VaultMetadataResponse response = vaultVersionedKeyValueTemplate.opsForKeyValueMetadata().get("test");
    Assert.assertEquals(response.getVersions().size(), 1);
    vaultVersionedKeyValueTemplate.opsForKeyValueMetadata().delete("test");
    response = vaultVersionedKeyValueTemplate.opsForKeyValueMetadata().get("test");
    Assert.assertNull(response);
  }
{% endhighlight %}

# Available starting from 2.3 version:

The PR has been already merged, but the `VaultKeyValueMetadataOperations` will only be available in the 2.3 release of `spring-vault`. Until then, it's possible to use snapshot version by adding the snapshots repository: 

 {% highlight xml  linenos %}
<dependency>
<groupId>org.springframework.vault</groupId>
<artifactId>spring-vault-core</artifactId>
<version>2.3.0-SNAPSHOT</version>
</dependency>

<repositories>
  <repository>
    <id>spring-libs-snapshot</id>
    <name>Spring Snapshot Repository</name>
    <url>https://repo.spring.io/libs-snapshot</url>
  </repository>
</repositories>
{% endhighlight %}

