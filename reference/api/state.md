# State management

Dapr offers a reliable state endpoint that allows developers to save and retrieve state via an API.
Dapr has a pluggable architecture and allows binding to a multitude of cloud/on-premises state stores.

Examples for state stores include ```Redis```, ```Azure CosmosDB```, ```AWS DynamoDB```, ```GCP Cloud Spanner```, ```Cassandra``` to name a few.

An Dapr State Store has the following structure:

```
apiVersion: dapr.io/v1alpha1
kind: Component
metadata:
  name: <NAME>
spec:
  type: state.<TYPE>
  metadata:
  - name:<KEY>
    value:<VALUE>
  - name: <KEY>
    value: <VALUE>
```

The ```metadata.name``` is the name of the state store.

the ```spec/metadata``` section is an open key value pair metadata that allows a binding to define connection properties.

## Key scheme
Dapr state stores are key/value stores. To ensure data compatibility, Dapr requires these data stores follow a fixed key scheme. For general states, the key format is:
```bash
<Dapr id>-<state key>
```
For Actor states, the key format is:
```bash
<Dapr id>-<Actor type>-<Actor id>-<state key>
```


## Save state

This endpoint lets you save an array of state objects.

### HTTP Request

`POST http://localhost:3500/v1.0/state`

#### Request Body
A JSON array of state objects. Each state object is comprised with the following fields:

Field | Description
---- | -----------
key | state key
value | state value, which can be any byte array
etag | (optional) state ETag
metadata | (optional) additional key-value pairs to be passed to the state store
options | (optional) state operation options, see [state operation options](#state-operation-options)

> **ETag format** Dapr runtime treats ETags as opaque strings. The exact ETag format is defined by the corresponding data store. 

### HTTP Response
#### Response Codes

Code | Description
---- | -----------
201  | State saved
400  | State store is missing or misconfigured
500  | Failed to save state

#### Response Body
None.

### Example
```shell
curl -X POST http://localhost:3500/v1.0/state \
  -H "Content-Type: application/json"
  -d '[
        {
          "key": "weapon",
          "value": "DeathStar"
        },
        {
          "key": "planet",
          "value": {
            "name": "Tatooine"
          }
        }
      ]'
```

## Get state

This endpoint lets you get the state for a specific key.

### HTTP Request

`GET http://localhost:3500/v1.0/state/<key>`

#### URL Parameters

Parameter | Description
--------- | -----------
key | the key of the desired state
consistency | (optional) read consistency mode, see [state operation options](#state-operation-options)

### HTTP Response

#### Response Codes

Code | Description
---- | -----------
200  | Get state successful
204  | Key is not found
400  | State store is missing or misconfigured
500  | Get state failed

#### Response Headers

Header | Description
--------- | -----------
ETag | ETag of returned value

#### Response Body
JSON-encoded value

### Example 

```shell
curl http://localhost:3500/v1.0/state/planet \
  -H "Content-Type: application/json"
```

> The above command returns the state:

```json
{
  "name": "Tatooine"
}
```
## Delete state

This endpoint lets you delete the state for a specific key.

### HTTP Request

`DELETE http://localhost:3500/v1.0/state/<key>`

#### URL Parameters

Parameter | Description
--------- | -----------
key | the key of the desired state
concurrency | (optional) either *first-write* or *last-write*, see [state operation options](#state-operation-options)
consistency | (optional) either *strong* or *eventual*, see [state operation options](#state-operation-options)
retryInterval | (optional) retry interval, in milliseconds, see [retry policy](#retry-policy)
retryPattern | (optional) retyr pattern, can be either *linear* or *exponential*, see [retry policy](#retry-policy)
retryThreshold | (optinal) number of retries, see [retry policy](#retry-policy)

#### Request Headers

Header | Description
--------- | -----------
If-Match | (Optional) ETag associated with the key to be deleted

### HTTP Response

#### Response Codes

Code | Description
---- | -----------
200  | Delete state successful
400  | State store is missing or misconfigured
500  | Delete state failed

#### Response Body
None.

### Example

```shell
curl -X "DELETE" http://localhost:3500/v1.0/state/planet -H "ETag: xxxxxxx"
```

## Expected state store behaviors

### Key scheme

A Dapr-compatible state store shall use the following key scheme:

* *\<Dapr id>-\<state key>* key format for general states
* *\<Dapr id>-\<Actor type>-\<Actor id>-\<state key>* key format for Actor states. 

### Concurrency
Dapr uses Optimized Concurrency Control (OCC) with ETags. Dapr imposes the following requirements on state stores: 
* An Dapr-compatible state store shall support optimistic concurrency control using ETags. When an ETag is associated with an *save* or *delete*  request, the store shall allow the update only if the attached ETag matches with the latest ETag in the database. 
* When ETag is mssing in the write requests, the state store shall handle the reuqests in a last-write-wins fashion. This is to allow optimizations for high-throughput write scenarios in which data contingency is low or has no negative effects. 
* A store shall **always** return ETags when returning states to callers. 

### Consistency
Dapr allows clients to attach a consistency hint to *get*, *set* and *delete* operation. Dapr support two consistency level: **strong** and **eventual**, which are defined as the follows:

#### Eventual Consistency

Dapr assumes data stores are eventually consistent by default. A state should:

* For read requests, the state store can return data from any of the replicas
* For write request, the state store should asynchronously replicate updates to configured quorum after acknowledging the update request. 

#### Strong Consistency
  
When a strong consistency hint is attached, a state store should:

* For read requests, the state store should return the most up-to-date data consistently acorss replicas.
* For write/delete requests, the state store should synchronisely replicate updated data to configured quorum before completing the write request.

### Retry Policy
Dapr allows clients to attach retry policies to *set* and *delete* operations. A retry policy is described by three fields:

Field | Description
---- | -----------
retryInterval | Initial delay between retries, in milliseconds. The interval remains constant for *linear* retry pattern. The interval is doubled after each retry for *exponential* retry pattern. So, for *exponential* pattern, the delay after attempt *n* will be interval*2^(n-1).
retryPattern | Retry pattern, can be either *linear* or *exponential*.
retryThreshold | Maximum number of retries.

### Example
The following is a sample *set* request with a complete operation option definition:

```shell
curl -X POST http://localhost:3500/v1.0/state \
  -H "Content-Type: application/json"
  -d '[
        {
          "key": "weapon",
          "value": "DeathStar",
          "etag": "xxxxx",
          "options": {
            "concurrency": "first-write",
            "consistency": "strong",
            "retryPolicy": {
              "interval": 100,      
              "threshold" : 3,
              "pattern": "exponential"
            }
          }
        }
      ]'
```
