---
layout: default
title: Streaming Bulk
parent: Document APIs
nav_order: 20
redirect_from:
 - /opensearch/rest-api/document-apis/bulk/streaming/
---

# Bulk
**Introduced 2.17.0**
{: .label .label-purple }

The streaming bulk operation lets you add, update, or delete multiple documents in by streaming the request and getting the results as streaming response. In comparison to traditional [bulk]({{site.url}}{{site.baseurl}}/api-reference/document-apis/bulk/) APIs, streaming ingestion eliminates the need to guess the batch size (which is affected by the cluster operational state at any given moment of time) and naturally applies the back pressure between many clients and the cluster. The streaming works over HTTP/2 or HTTP 1.1 (using chunked transfer encoding), depending on the capabilities of the clients and the cluster.

The streaming support not provided by default HTTP transport. Instead, the [transport-reactor-netty4]({{site.url}}{{site.baseurl}}/install-and-configure/configuring-opensearch/network-settings/#selecting-the-transport) HTTP transport plugin has to be installed and used as the default HTTP transport. Both the transport and streaming bulk APIs are experimental.
{: .note}

## Example

```json
POST _bulk/stream -H "Transfer-Encoding: chunked" -H "Content-Type: application/json"
{ "delete": { "_index": "movies", "_id": "tt2229499" } }
{ "index": { "_index": "movies", "_id": "tt1979320" } }
{ "title": "Rush", "year": 2013 }
{ "create": { "_index": "movies", "_id": "tt1392214" } }
{ "title": "Prisoners", "year": 2013 }
{ "update": { "_index": "movies", "_id": "tt0816711" } }
{ "doc" : { "title": "World War Z" } }

```
{% include copy-curl.html %}


## Path and HTTP methods

```
POST _bulk/stream
POST <index>/_bulk/stream
```

Specifying the index in the path means you don't need to include it in the [request body chunks]({{site.url}}{{site.baseurl}}/api-reference/document-apis/bulk/#request-body).

OpenSearch also accepts PUT requests to the `_bulk/steram` path, but we highly recommend using POST. The accepted usage of PUT---adding or replacing a single resource at a given path---doesn't make sense for streaming bulk requests.
{: .note }


## URL parameters

All streaming bulk URL parameters are optional.

Parameter | Type | Description
:--- | :--- | :---
pipeline | String | The pipeline ID for preprocessing documents.
refresh | Enum | Whether to refresh the affected shards after performing the indexing operations. Default is `false`. `true` makes the changes show up in search results immediately, but hurts cluster performance. `wait_for` waits for a refresh. Requests take longer to return, but cluster performance doesn't suffer.
require_alias | Boolean | Set to `true` to require that all actions target an index alias rather than an index. Default is `false`.
routing | String | Routes the request to the specified shard.
timeout | Time | How long to wait for the request to return. Default `1m`.
type | String | (Deprecated) The default document type for documents that don't specify a type. Default is `_doc`. We highly recommend ignoring this parameter and using a type of `_doc` for all indexes.
wait_for_active_shards | String | Specifies the number of active shards that must be available before OpenSearch processes the bulk request. Default is 1 (only the primary shard). Set to `all` or a positive integer. Values greater than 1 require replicas. For example, if you specify a value of 3, the index must have two replicas distributed across two additional nodes for the request to succeed.
batch_interval | Time | Specifies how long bulk operations should be accumulated into batch before sending over to data nodes.
batch_size | Time | Specifies how many bulk operations should be accumulated into batch before sending over to data nodes. Default `1`.
{% comment %}_source | List | asdf
_source_excludes | list | asdf
_source_includes | list | asdf{% endcomment %}

## Request body

The streaming bulk API request body is fully compatible with bulk API [request body]({{site.url}}{{site.baseurl}}/api-reference/document-apis/bulk/#request-body), whereas each bulk operation (create / index / update / delete) is sent as a separate chunk.  

## Example response

Depending on the batch settings, each streamed response chunk may report the results of one or many (batch) bulk operations, for example for the request from above with no batching (default), the following streaming response will be received:

```json
{"took": 11, "errors": false, "items": [ { "index": {"_index": "movies", "_id": "tt1979320", "_version": 1, "result": "created", "_shards": { "total": 2 "successful": 1, "failed": 0 }, "_seq_no": 1, "_primary_term": 1, "status": 201 } } ] }
{"took": 2, "errors": true, "items": [ { "create": { "_index": "movies", "_id": "tt1392214", "status": 409, "error": { "type": "version_conflict_engine_exception", "reason": "[tt1392214]: version conflict, document already exists (current version [1])", "index": "movies", "shard": "0", "index_uuid": "yhizhusbSWmP0G7OJnmcLg" } } } ] }
{"took": 4, "errors": true, "items": [ { "update": { "_index": "movies", "_id": "tt0816711", "status": 404, "error": { "type": "document_missing_exception", "reason": "[_doc][tt0816711]: document missing", "index": "movies", "shard": "0", "index_uuid": "yhizhusbSWmP0G7OJnmcLg" } } } ] }
```