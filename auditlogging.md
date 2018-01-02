---
redirect_to:
  - http://docs.search-guard.com/latest/audit-logging-compliance
---

<!---
Copryight 2016 floragunn GmbH
-->

# Audit Logging

Audit logging enables you to track access to your Elasticsearch cluster. Search Guard tracks the following types of events, on REST and transport levels:

* FAILED_LOGIN—the provided credentials of a request could not be validated, most likely because the user does not exist or the password is incorrect. 
* MISSING_PRIVILEGES—an attempt was made to access Elasticsearch, but the user does not have the required permissions.
* BAD_HEADERS—an attempt was made to spoof a request to Elasticsearch with Search Guard internal headers.
* SSL_EXCEPTION—an attempt was made to access Elasticsearch without a valid SSL/TLS certificate.
* SG\_INDEX\_ATTEMPT—an attempt was made to access the Search Guard internal user and privileges index without a valid admin TLS certificate. 
* AUTHENTICATED—represents a successful request to Elasticsearch. 

All events are logged asynchronously, so the audit log has only minimal impact on the performance of your cluster. You can tune the number of threads that Search Guard uses for audit logging.  See the section "Finetuning the thread pool" below.
  
For security reasons, audit logging has to be configured in `elasticsearch.yml`, not in `sg_config.yml`. Thus, changes to the audit log settings require a restart of all participating nodes in the cluster.

## Installation

Download the Audit Log enterprise module from Maven Central:

[Maven central](http://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22com.floragunn%22%20AND%20a%3A%22dlic-search-guard-module-auditlog%22)

and place it in the folder

* `<ES installation directory>/plugins/search-guard-2`

or

* `<ES installation directory>/plugins/search-guard-5`

if you are using Search Guard 5.

**Choose the module version matching your Elasticsearch version, and download the jar with dependencies.**

After that, restart all nodes to activate the module.
  
## Configuring audit logging

### Configuring the categories to be logged

Per default, the audit log module logs all events in all categories. If you want to log only certain events, you can disable categories individually in the `elasticsearch.yml` configuration file:

```
searchguard.audit.config.disabled_categories: [disabled categories]
```

For example:

```
searchguard.audit.config.disabled_categories: AUTHENTICATED, SG_INDEX_ATTEMPT
```

In this case, events in the categories `AUTHENTICATED` and `SG_INDEX_ATTEMPT` will not be logged.

### Configuring the log level

By default, Search Guard logs a reasonable amount of information for each audit log event, suitable for identifying attempted security breaches. This includes the audit category, user information, source IP, date, reason etc.

Search Guard also provides an extended log format, which includes the requested index (including composite index requests), and the query that was submitted. This extended logging can be enabled and disabled by setting:

```
searchguard.audit.enable_request_details: <boolean>
```

Since this extended logging comes with an performance overhead, the default setting is `false`.

#### Extended logging caveats

The extended logging can produce a considerable amount of log information. If you plan to use extended logging in production, please keep the following things in mind:

**Exclude the AUTHENTICATED category**

If the AUTHENTICATED category is enabled, Search Guard will log all requests, including valid, authenticated requests. This can lead to a huge amount of messages and **should be avoided in combination with extended logging**.

**Composite requests and field limits**

If a request contains subrequests, Search Guard adds audit information for each subrequest to the audit message separately. This includes the index name, the document type or the source. Each subrequest is identified by a consecutive number, appended to the field name of the logged data.

For example, if a request contains three subrequests, the audit message will contain the affected indices for each subrequest, like:

```
audit_trace_indices_sub_1: ...
audit_trace_indices_sub_2: ...
audit_trace_indices_sub_3: ...
```

If your composite request contains a huge number of subrequests, the produced audit messages will contain a huge number of fields as well. You will likely hit the field limit of Elasticsearch, which defaults to 1000 per index (see the corresponding issue on [GitHub](https://github.com/elastic/elasticsearch/pull/17357)).

If necessary, you can set a higher value for the field limit, even after the index has been created:

```
PUT auditlog/_settings
{
  "index.mapping.total_fields.limit": 10000
} 
```

However, before increasing the field limit, please think about if it is really necessary to log these kinds of messages at all.

**Use an external storage type**

Due to the amount of information stored, the audit log index can grow quite big. It's recommended to use an external storage for the audit messages, like `external_elasticsearch` or `webhook`, so you dont' put your production cluster in jeopardy.  

### Configuring excluded users (requires Audit Log v5 or above)

By default, Search Guard logs events from all users. In some cases you might want to exclude events created by certain users from being logged. For example, you might want to exclude the Kibana server user or the logstash user. You can define users to be excluded by setting the following configuration:

```
searchguard.audit.ignore_users:
  - kibanaserver
```

### Configuring the storage type

Search guard comes with three audit log storage types. This specifies where you want to store the tracked events. You can choose from:

* debug—outputs the events to stdout.
* internal_elasticsearch—writes the events in a separate audit index on the same cluster.
* external_elasticsearch—writes the events in a separate audit index on another ES cluster.
* webhook-writes the events to an arbitrary HTTP endpoint.

You configure the type of audit logging in the `elasticsearch.yml` file:

```
searchguard.audit.type: <debug|internal_elasticsearch|external_elasticsearch|webhook>
```

Note that it is not possible to specify more than one storage type at the moment.

You can also use your own, custom implementation of storage in case you have special requirements that are not covered by the built-in types. See the section "Custom storage types" below.

## Storage type 'debug'

There are no special configuration settings for this audit type.  Just add the audit type setting in `elasticsearch.yml`:

```
searchguard.audit.type: debug
```

This will output tracked events to stdout, and is mainly useful when debugging or testing.

## Storage type 'internal_elasticsearch'

In addition to specifying the type as `internal_elasticsearch`, you can set the index name and the document type:

```
searchguard.audit.type: internal_elasticsearch
searchguard.audit.config.index: <indexname>
searchguard.audit.config.type: <typename>
```

If not specified, Search Guard uses the default value `auditlog` for both index name and document type.

Since v5, you can use a date/time pattern in the index name as well, for example to set up a daily rolling index. For a reference on the date/time pattern format, please refer to the [Joda DateTimeFormat docs](http://www.joda.org/joda-time/apidocs/org/joda/time/format/DateTimeFormat.html).

Example:

```
searchguard.audit.config.index: "'auditlog-'YYYY.MM.dd"
```
 

## Storage type 'external_elasticsearch'

If you want to store the tracked events in a different Elasticsearch cluster than the cluster producing the events, you use `external_elasticsearch` as audit type, configure the Elasticsearch endpoints with hostname/IP and port and optionally the index name and document type:

```
searchguard.audit.type: internal_elasticsearch
searchguard.audit.config.http_endpoints: <endpoints>
searchguard.audit.config.index: <indexname>
searchguard.audit.config.type: <typename>
```

Since v5, you can use date/time pattern in the index name as well, as described for storage type `internal_elasticsearch`.

SearchGuard uses the Elasticsearch REST API to send the tracked events. So, for `searchguard.audit.config.http_endpoints`, use a comma-delimited list of hostname/IP and the REST port (default 9200). For example:

```
searchguard.audit.config.http_endpoints: 192.168.178.1:9200,192.168.178.2:9200
```

### Storing audit logs in a Search Guard secured cluster

If you use `external_elasticsearch` as audit type, and the cluster you want to store the audit logs in is also secured by Search Guard, you need to supply some additional configuration parameters.

The parameters depend on what authentication type you configured on the REST layer.

### TLS settings

Use the following settings to control SSL/TLS:

```
searchguard.audit.config.enable_ssl: <true|false>
```
Whether or not to use SSL/TLS. If you enabled SSL/TLS on the REST-layer of the receiving cluster, set this to true. The default is false.

```
searchguard.audit.config.verify_hostnames: <true|false>
```
Whether or not to verify the hostname of the SSL/TLS certificate of the receiving cluster. The default is true.

```
searchguard.audit.config.enable_ssl_client_auth: <true|false>
```
Whether or not to enable SSL/TLS client authentication. If you set this to true, the audit log module will send the nodes certificate from the keystore along with the request. The receiving cluster can use this certificate to verify the identity of the caller.

Note: The audit log module will use the key and truststore settings configured in the HTTP/REST layer SSL section of elasticsearch.yml. Please refer to the [Search Guard SSL](https://github.com/floragunncom/search-guard-ssl-docs/blob/master/configuration.md) configuration chapter for more information.

### Basic auth settings

If you enabled HTTP Basic auth on the receiving cluster, use these settings to specify username and password the the audit log module should use:

```
searchguard.audit.config.username: <username>
searchguard.audit.config.password: <password>
```

## Storage type 'webhook'

This storage type ships the audit log events to an arbitrary HTTP endpoint. Enable this storage type by adding the following to `elasticsearch.yml:`

```
searchguard.audit.type: webhook
```

In addition, you can configure the following keys:

```
searchguard.audit.config.webhook.url: <string>
```
The URL that the log events are shipped to. Can be an HTTP or HTTPS URL.

```
webhook.ssl.verify: <boolean>
```

If true, the TLS certificate provided by the endpoint (if any) will be verified. If set to false, no verification is performed. You can disable this check if you use self-signed certificates, for example.

```
webhook.format: <URL_PARAMETER_GET|URL_PARAMETER_POST|TEXT|JSON|SLACK>
```

The format in which the audit log message is logged:

**URL\_PARAMETER\_GET**

The audit log message is submitted to the configured webhook URL as HTTP GET. All logged information is appended to the URL as request parameters.

**URL\_PARAMETER\_POST**

The audit log message is submitted to the configured webhook URL as an HTTP POST. All logged information is appended to the URL as request parameters.

**TEXT**

The audit log message is submitted to the configured webhook URL as HTTP POST. The body of the HTTP POST request contains the audit log message in plain text format.

**JSON**

The audit log message is submitted to the configured webhook URL as an HTTP POST. The body of the HTTP POST request contains the audit log message in JSON format.

**SLACK**

The audit log message is submitted to the configured webhook URL as an HTTP POST. The body of the HTTP POST request contains the audit log message in JSON format suitable for consumption by Slack. The default implementation returns `"text": "<AuditMessage#toText>"`

### Customizing the audit log event format

If you need to provide the audit log event in a special format, suitable for consumption by your SIEM system, for example, you can subclass the `com.floragunn.searchguard.auditlog.impl.WebhookAuditLog` class and configure it as custom storage type (see below).

The relevant methods are:

```
protected String formatJson(final AuditMessage msg) {
  return ...
}
```

This method is called when the webhook format is set to JSON.

```
protected String formatText(final AuditMessage msg) {
  return ...
}
```

This method is called when the webhook format is set to TEXT.

```
protected String formatUrlParameters(final AuditMessage msg) {
  return ...
}
```

This method is called when using URL\_PARAMETER\_GET or URL\_PARAMETER\_POST as webhook format.

## Custom storage types

If you have special requirements regarding the storage, you can always implement your own storage type and let the audit log module use it instead of the built in types.

### Implementing a custom storage

Implementing a custom storage is very easy. You just write a class with **a default constructor** that extends `com.floragunn.searchguard.auditlog.impl.AbstractAuditLog`. You need to implement only two methods:

```
protected void save(final AuditMessage msg) {
}

public void close() throws IOException {
}
```

The `save` method is responsible for storing the event to whatever storage you require. The interface `AuditLog` also extends `java.io.Closeable`. If the node is shut down, this method is called, and you can use it to close any resources you have used. For example, the `HttpESAuditLog` uses it to close the connection to the remote ES cluster.

### Configuring a custom storage

In order for Search Guard to pick up your custom implementation, specify its fully qualified name as `searchguard.audit.type`:

```
searchguard.audit.type: com.example.MyCustomAuditLogStorage
```

Make sure that the class is accessible by Search Guard by putting the respective `jar` file in the `plugins/search-guard-2` or `plugins/search-guard-5` folder.


## Advanced settings: Finetuning the thread pool

All events are logged asynchronously, so the overall performance of our cluster will only be affected minimally. Search Guard internally uses a fixed thread pool to submit log events. You can define the number of threads in the pool by the following configuration key in `elasticsearch.yml`:

```
searchguard.audit.threadpool.size: <integer>
```

The default setting is `10`. Setting this value to `0` disables the thread pool completey, and the events are logged synchronously. 