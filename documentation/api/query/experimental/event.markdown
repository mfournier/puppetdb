---
title: "PuppetDB 1.3 » API » Experimental » Querying Events"
layout: default
canonical: "/puppetdb/latest/api/query/experimental/event.html"
---

[curl]: ../curl.html#using-curl-from-localhost-non-sslhttp
[report]: ./report.html
[operator]: ../v2/operators.html

## Routes

### `GET /experimental/events`

This will return all resource events matching the given query.  (Resource events
are generated from Puppet reports.)  There must be an `Accept` header containing
`application/json`.

#### Parameters

* `query`: Required. A JSON array of query predicates, in prefix form (the standard
 `["<OPERATOR>", "<FIELD>", "<VALUE>"]` format), conforming to the format described
 below.

The `query` parameter is described by the following grammar:

    query: [ {bool} {query}+ ] | [ "not" {query} ] | [ {match} {field} {value} ] | [ {inequality} {field} {value} ]
    field:          FIELD (conforming to [Fields](#fields) specification below)
    value:          string
    bool:           "or" | "and"
    match:          "=" | "~"
    inequality:     ">" | ">=" | "<" | "<="

For example, for all events in the report with id
'38ff2aef3ffb7800fe85b322280ade2b867c8d27', the JSON query structure would be:

    ["=", "report", "38ff2aef3ffb7800fe85b322280ade2b867c8d27"]

To retrieve all of the events within a given time period:

    ["and", ["<", "timestamp", "2011-01-01T12:01:00-03:00"],
            [">", "timestamp", "2011-01-01T12:00:00-03:00"]]

To retrieve all of the 'failed' events for nodes named 'foo.*' and resources of
type 'Service':

    ["and", ["=", "status", "failed"],
            ["~", "certname", "^foo\\."],
            ["=", "resource-type", "Service"]]

For more information on the available values for `FIELD`, see the [fields](#fields) section below.

* `limit`: Optional.  If specified, this should be an integer value that will be used to override the `event-query-limit` [configuration setting](../../../configure.html#event_query_limit)

##### Operators

See [the Operators page](./operators.html) for the full list of available operators.
Note that inequality operators (`<`, `>`, `<=`, `>=`) are only supported against
the `timestamp` FIELD.

##### Fields

`FIELD` may be any of the following.  Unless otherwise noted, all fields support
both equality and regular expression match operators, but do not support inequality
operators.

`certname`
: the name of the node that the event occurred on.

`report`
: the id of the report that the event occurred in; these ids can be acquired
  via event queries or via the [`/reports`][report] query endpoint.

`status`
: the status of the event; legal values are `success`, `failed`, `noop`, and `skipped`.

`timestamp`
: the timestamp (from the puppet agent) at which the event occurred.  This field
  supports the inequality operators.  All values should be specified as ISO-8601
  compatible date/time strings.

`resource-type`
: the type of resource that the event occurred on; e.g., `File`, `Package`, etc.

`resource-title`
: the title of the resource that the event occurred on

`property`:
: the property/parameter of the resource that the event occurred on; e.g., for a
  `Package` resource, this field might have a value of `ensure`.  NOTE: this field
  may contain `NULL` values; see notes below.

`new-value`
: the new value that Puppet was attempting to set for the specified resource
  property.  NOTE: this field may contain `NULL` values; see notes below.

`old-value`
: the previous value of the resource property, which Puppet was attempting to
  change.  NOTE: this field may contain `NULL` values; see notes below.

`message`
: a description (supplied by the resource provider) of what happened during the
  event.  NOTE: this field may contain `NULL` values; see notes below.

##### Notes on fields that allow `NULL` values

In the case of a `skipped` resource event, some of the fields of an event may
not have values.  We handle this case in a slightly special way when these
fields are used in equality (`=`) or inequality (`!=`) queries; specifically,
an equality query will always return `false` for an event with no value for
the field, and an inequality query will always return `true`.

#### Response format

 The response is a JSON array of events that matched the input parameters.
 The events are sorted by their timestamps, in descending order:

    [
      {
        "certname": "foo.localdomain",
        "old-value": "absent",
        "property": "ensure",
        "timestamp": "2012-10-30T19:01:05.000Z",
        "resource-type": "File",
        "resource-title": "/tmp/reportingfoo",
        "new-value": "file",
        "message": "defined content as '{md5}49f68a5c8493ec2c0bf489821c21fc3b'",
        "report": "38ff2aef3ffb7800fe85b322280ade2b867c8d27",
        "status": "success"
      },
      {
        "certname": "foo.localdomain",
        "old-value": "absent",
        "property": "message",
        "timestamp": "2012-10-30T19:01:05.000Z",
        "resource-type": "Notify",
        "resource-title": "notify, yo",
        "new-value": "notify, yo",
        "message": "defined 'message' as 'notify, yo'",
        "report": "38ff2aef3ffb7800fe85b322280ade2b867c8d27",
        "status": "success"
      }
    ]


#### Example

[You can use `curl`][curl] to query information about events like so:

    curl -G -H "Accept: application/json" 'http://localhost:8080/experimental/events' --data-urlencode 'query=["=", "report", "38ff2aef3ffb7800fe85b322280ade2b867c8d27"]' --data-urlencode 'limit=1000'
