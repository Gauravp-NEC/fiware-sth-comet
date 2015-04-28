#<a id="section0"></a> IoT-STH

* [Introduction] (#section1)
    * [Consuming raw data] (#section1.1)
    * [Consuming aggregated time series information] (#section1.2)
    * [Updating aggregated time series information] (#section1.3)
* [Dependencies](#section2)
* [Installation](#section3)
* [Running the STH server](#section4)
* [Inserting data (random single events and its aggregated data) into the database](#section5)
* [STH component complete test coverage](#section6)
* [Contact](#section7)

##<a id="section1"></a> Introduction
The STH component is a FIWARE component in charge of providing aggregated time series information about the evolution in
time of entity attribute values registered using the
<a href="http://catalogue.fiware.org/enablers/publishsubscribe-context-broker-orion-context-broker" target="_blank">Orion Context Broker</a>,
an implementation of the publish/subscribe context management system exposing NGSI9 and
<a href="http://technical.openmobilealliance.org/Technical/technical-information/release-program/current-releases/ngsi-v1-0">NGSI10</a> interfaces.

The aggregated time series information is stored in a MongoDB instance. This information can be generated by 2 main
means:

1. The STH component can directly subscribe to the Context Broker to receive notifications when the entity attribute
values change, calculating the aggregated time series information and storing it in the MongoDB instance.
This option is called the minimalist option.
2. A new sink will be enabled in the <a href="https://github.com/telefonicaid/fiware-cygnus" target="_blank">Cygnus</a>
component to calculate and to update the aggregated time series information
as the entity attribute values change over time. Using Cygnus adds a set of capabilities not available in the minimalist
option such as advanced filtering regarding the attributes to consider in the time series, advanced flow and congestion
management, amongst others. This option is provided as a
<a href="https://github.com/telefonicaid/fiware-cygnus/blob/master/flume/doc/devel/add_new_sink.md" target="_blank">Cygnus sink</a>
just like the <a href="https://github.com/telefonicaid/fiware-cygnus/tree/master/flume/src/main/java/es/tid/fiware/fiwareconnectors/cygnus/sinks" target="_blank">other data stores</a> already supported by Cygnus.
This option is the formal one.

Since both mechanisms (the formal one using Cygnus and the minimalist one using the STH component directly) update the same database,
it is the responsibility of the people or software in charge of creating the needed subscriptions to avoid updating the
time series database twice (i.e. to avoid enabling both mechanisms at the same time). This would happen if both mechanisms
are enabled for the same attribute of the same entity.

Regarding the aggregated time series information provided by the STH component, there are 4 main concepts which are
important to know about:

* <b>Range</b>: The period of time about which the aggregated time series information is provided. Possible valid
ranges values are: year, month, day, hour, minute.
* <b>Resolution</b> or <b>aggregation period</b>: The time period by which the aggregated time series information is grouped.
Possible valid resolution values are: month, day, hour, minute and second. For the time being, we only consider the
following range-resolution pairs: year-month, month-day, day-hour, hour-minute and minute-second.
* <b>Origin</b>: For certain range-resolution pair, it is the origin of time for which the aggregated time series
information applies. For example, for a pair hour-minute, a valid origin value could be: ```2015-03-01T13:00:00.000Z```,
meaning the 13th hour of March, the 3rd, 2015. The origin is stored using UTC time to avoid locale issues.
* <b>Offset</b>: For certain range-resolution pair, it is the offset from the origin for which the aggregated time series
information applies. For example, for a pair hour-minute and an origin ```2015-03-01T13:00:00.000Z```, an offset of 10
refers to the 10th minute of the concrete hour pointed by the origin. In this example, there would be a maximum of 60
offsets from 0 to 59 corresponding to each one of the 60 minutes within the concrete hour.
* <b>Samples</b>: For a quadruple range-resolution-origin-offset, it is the number of samples, values, events or notifications available.

###<a id="section1.1"></a> Consuming raw data

The STH component exposes an HTTP REST API to let external clients query the raw events (aka. raw data) from which the
aggregated time series information is generated. A typical URL querying for this information using a GET request is the following:

<pre>http://localhost:8666/STH/v1/contextEntities/type/&lt;entityType&gt;/id/&lt;entityId&gt;/attributes/&lt;attrName&gt;?hLimit=3&hOffset=0&dateFrom=2014-02-14T00:00:00.000Z&dateTo=2014-02-14T23:59:59.999Z</pre>

The entries between "<" and ">" in the URL path depend on the concrete case (type of data, entity and attribute) being queried.

The requests can make use the following query parameters:

* <b>lastN</b>: Only the requested last entries should be returned. It is a mandatory parameter if no hLimit and hOffset are provided.
* <b>hLimit</b>: In case of pagination, the number of entries per page. It is a mandatory parameter if no lastN is provided.
* <b>hOffset</b>: In case of pagination, the offset to apply to the requested search of raw data. It is a mandatory parameter if no lastN is provided.

An example response provided by the STH component to a request such as the previous one could be the following:

<pre>
{
    "contextResponses": [
        {
            "contextElement": {
                "attributes": [
                    {
                        "name": "attrName",
                        "values": [
                            {
                                {
                                    "recvTime": "2014-02-14T13:43:33.306Z",
                                    "attrValue": "21.28"
                                },
                                {
                                   "recvTime": "2014-02-14T13:43:34.636Z",
                                   "attrValue": "23.42"
                                },
                                {
                                    "recvTime": "2014-02-14T13:43:35.424Z",
                                    "attrValue": "22.12"
                                }
                            }
                        ]
                    }
                ],
                "id": "entityId",
                "isPattern": false
            },
            "statusCode": {
                "code": "200",
                "reasonPhrase": "OK"
            }
        }
    ]
}
</pre>

Notice that a paginated response has been requested with a limit of 3 entries and an offset of 0 entries (first page).

[Top](#section0)

###<a id="section1.2"></a> Consuming aggregated time series information

The STH component exposes an HTTP REST API to let external clients query this aggregated time series information. A
typical URL querying for this information using a GET request is the following:

<pre>http://localhost:8666/STH/v1/contextEntities/type/&lt;entityType&gt;/id/&lt;entityId&gt;/attributes/&lt;attrName&gt;?aggrMethod=sum&aggrPeriod=second&dateFrom=2015-02-22T00:00:00.000Z&dateTo=2015-02-22T23:00:00.000Z</pre>

The entries between "<" and ">" in the URL path depend on the concrete case (type of data, entity and attribute) being queried.

The requests can make use the following query parameters:

* <b>aggrMethod</b>: The aggregation method. The STH component supports the following aggregation methods: max (maximum
value), min (minimum value), sum (sum of all the samples) and sum2 (sum of the square value of all the samples). Combining
the information provided by these aggregated methods with the number of samples, it is possible to calculate probabilistic
values such as the average value, the variance as well as the standard deviation. It is a mandatory parameter.
* <b>aggrPeriod</b>: Aggregation period or resolution. For the time being, a fixed resolution determines the range as
well as the origin time format and the possible offsets. It is a mandatory parameter.
* <b>dateFrom</b>: The origin of time from which the aggregated time series information is desired. It is an optional parameter.
* <b>dateTo</b>: The end of time until which the aggregated time series information is desired. It is an optional parameter.

An example response provided by the STH component to a request such as the previous one could be the following:

<pre>
{
    "contextResponses": [
        {
            "contextElement": {
                "attributes": [
                    {
                        "name": "attrName",
                        "values": [
                            {
                                "_id": {
                                    "origin": "2015-02-18T02:46:00.000Z",
                                    "range": "minute",
                                    "resolution": "second"
                                },
                                "points": [
                                    {
                                        "offset": 13,
                                        "samples": 1,
                                        "sum": 34.59
                                    }
                                ]
                            }
                        ]
                    }
                ],
                "id": "entityId",
                "isPattern": false
            },
            "statusCode": {
                "code": "200",
                "reasonPhrase": "OK"
            }
        }
    ]
}
</pre>

In this example response, aggregated time series information for a range of minutes and a resolution of seconds is returned.
This information has as its origin the 46nd minute, of the 2nd hour of February, the 18th, 2015. And includes data for the
13th second, for which there is a sample and the sum (and value of that sample) is 34.59.

[Top](#section0)

###<a id="section1.3"></a> Updating aggregated time series information

As already mentioned, there are 2 main ways to update the aggregated time series information associated to attributes.
The so-called minimalist option and the formal one.

Regarding the formal option (based on using the Cygnus component for the updating), please refer to the documentation available at the
<a href="https://github.com/telefonicaid/fiware-cygnus" target="_blank">Cygnus component repository</a>, and more concretely at the following links:

* <a href="https://github.com/telefonicaid/fiware-cygnus/tree/master/flume" href="_blank">Cygnus connector documentation</a>
* <a href="https://github.com/telefonicaid/fiware-cygnus/tree/master/flume#orion-subscription" href="_blank">Orion subscription</a>

The another option to update the aggregated time series information consists on directly subscribing the STH component
to the Orion Context Broker to receive the corresponding notifications and generate and update the aggregated data.

In the minimalist option, the STH component calculates aggregated data grouped at certain resolutions whenever it receives
a notification from the Orion Context Broker. To this regard and as a way to subscribe the STH component to the Orion Context Broker
so it receives the attribute values of interest, the following curl command can be used:

<pre>
curl orion.contextBroker.host:1026/v1/subscribeContext -s -S --header 'Content-Type: application/json' --header 'Accept: application/json' --header 'Fiware-Service: theService' --header 'Fiware-ServicePath: theServicePath' -d @- &lt;&lt;EOF
{
    "entities": [
        {
            "type": "Room",
            "isPattern": "false",
            "id": "Room1-gtv"
        }
    ],
    "attributes": [
        "temperature"
    ],
    "reference": "http://&lt;sth.host&gt;:&lt;sth.port&gt;/notify",
    "duration": "P1M",
    "notifyConditions": [
        {
            "type": "ONTIMEINTERVAL",
            "condValues": [
                "PT1S"
            ]
        }
    ],
}
EOF
</pre>

In this request, a subscription to be notified the value of the temperature attribute of the Room1 entity every second
is made to an instance of the Orion Context Broker listening at orion.contextBroker.host:1026.
The notifications will be sent to the endpoint made available by the STH component at http://<sth.host>:<sth.port>/notify

If the list of "attributes" is empty, this is interpreted by the Orion Context Broker as "all the attributes of the selected entities".

Of course, the concrete curl command to be used depends on each case but can easily be infered from the previous example.

Remember that subscription expire and must be re-enabled. More concretely, the "duration" property sets the duration of the subscription.
One month in the proposed example.

On the other hand, for the time being the STH component only is able to manage notifications in JSON format and consequently
it is very important to set the "Accept" header to "application/json".

Last but not least, the "throttling" makes it possible to control the frequency of the notifications. In this sense and for
this concrete example, the Orion Context Broker will send notifications separated 1 second in time the least. This is, the
time between notifications will be at least 1 second. Depending on the resolution of the aggregated data you are interested
in, the "throttling" should be fine-tuned accordingly.

[Top](#section0)

##<a id="section2"></a> Dependencies
The STH component is a Node.js application which depends on certain Node.js modules as stated in the ```project.json``` file.

Apart from these Node.js modules, the STH component also needs a running MongoDB instance where the aggregated time series
information is stored for its proper functioning. Since the STH component uses MongoDB update operators (see
<a href="http://docs.mongodb.org/v2.6/reference/operator/update/" target="_blank">http://docs.mongodb.org/v2.6/reference/operator/update/</a>)
such as the ```$max``` and the ```$min``` update operators which were introduced in version 2.6, there is a dependency
of the STH component with this concrete version of the MongoDB instance where the aggregated data will be stored.
Consequently, a MongoDB instance version &gt;= 2.6 is needed to store the aggregated time series information.

[Top](#section0)

##<a id="section3"></a> Installation
1. Clone the repository:
<pre> git clone https://github.com/telefonicaid/IoT-STH.git </pre>
2. Get into the directory where the STH repository has been cloned:
<pre> cd IoT-STH/ </pre>
3. Install the Node.js modules and dependencies:
<pre> npm install </pre>
The STH component server is ready to be started.

[Top](#section0)

##<a id="section4"></a>Running the STH server
1. To run the STH server, just execute:
<pre> npm start </pre>

The script accepts the following parameters as environment variables:

- STH_HOST: The host where the STH server will be started. Optional. Default value: "localhost".
- STH_PORT: The port where the STH server will be listening. Optional. Default value: 8666.
- LOG_LEVEL: The logging level of the messages. Messages with a level equal or superior to this will be logged. Optional. Default value: "info".
- LOG_TO_CONSOLE: A flag indicating if the logs should be sent to the console. Optional. Default value: true.
- LOG_TO_FILE: A flag indicating if the logs should be sent to a file. Optional. Default value: true.
- LOG_FILE_MAX_SIZE_IN_BYTES: Maximum size in bytes of the log files. If the maximum size is reached, a new log file is created incrementing
a counter used as the suffix for the log file name. Optional. Default value: undefined.
- LOG_DIR: The path to a directory where the log file will be searched for or created if it does not exist. Optional. Default value: "./log".
- LOG_FILE_NAME: The name of the file where the logs will be stored. Optional. Default value: "sth_app.log".
- DB_PREFIX: The prefix to be added to the service for the creation of the databases. More information below. Optional. Default value: "sth".
- SERVICE: The service to be used if not sent by the Orion Context Broker in the notifications. Optional. Default value: "orion".
- COLLECTION_PREFIX: The prefix to be added to the collections in the databases. More information below. Optional. Default value: "sth".
- SERVICE_PATH: The service path to be used if not sent by the Orion Context Broker in the notifications. Optional. Default value: "/".
- POOL_SIZE: The default MongoDB pool size of database connections. Optional. Default value: "5".
- DATA_MODEL: The data model to use. Currently 3 possible values are supported: collection-per-service-path (which creates a MongoDB collection
 per service patch to store the data), collection-per-entity (which creates a MongoDB collection per service path and entity to store the data)
 and collection-per-attribute (which creates a collection per service path, entity and attribute to store the data). More information about these
 values below. Optional. Default value: "collection-per-attribute".
- DB_USERNAME: The username to use for the database connection. Optional. Default value: "".
- DB_PASSWORD: The password to use for the database connection. Optional. Default value: "".
- DB_URI: The URI to use for the database connection. This does not include the 'mongo://' protocol part (see a couple of examples below).
Optional. Default value: "localhost:27017".
- DB_NAME: The name of the database to use. Optional. Default value: "test".
- FILTER_OUT_EMPTY: A flag indicating if the empty results should be removed from the response. Optional. Default value: "true".

For example, to start the STH server listening on port 7777, connecting to a MongoDB instance listening on mymongo.com:27777 and
without filtering out the empty results, use:

<pre> STH_PORT=7777 DB_URI=mymongo.com:27777 FILTER_OUT_EMPTY=false npm start</pre>

On the other hand, in case of connecting to a MongoDB replica set composed of 3 machines with IPs addresses 1.1.1.1, 1.1.1.2, 1.1.1.3
listening on ports 27771, 27772 and 27773, respectively, use:

<pre> DB_URI=1.1.1.1:27771,1.1.1.2:27772,1.1.1.3:27773 npm start</pre>

Of special interest is the DATA_MODEL environment variable. Currently, the STH component supports 3 possible data distribution
models as a way to let us evaluate which of them provides the best performance. After running a set of performance tests currently
under implementation, we will opt for one of the options.

The STH component creates a new database for each <a href="https://forge.fiware.org/plugins/mediawiki/wiki/fiware/index.php/Publish/Subscribe_Broker_-_Orion_Context_Broker_-_User_and_Programmers_Guide#Multi_service_tenancy" target="_blank">service</a>.
The name of these databases will be the concatenation of the DB_PREFIX environment variable and the service, using an underscore ("_") as the separator.

Using these databases, the behavior of the STH component according to each one of the values the DATA_MODEL environment variable may have is the following:

- "collection-per-service": The STH component creates 2 collections per <a href="https://forge.fiware.org/plugins/mediawiki/wiki/fiware/index.php/Publish/Subscribe_Broker_-_Orion_Context_Broker_-_User_and_Programmers_Guide#Entity_service_paths" target="_blank">service path</a>
for each one of the databases, storing in these collection all the raw and aggregated data separately.
- "collection-per-entity": The STH component creates 2 collections per <a href="https://forge.fiware.org/plugins/mediawiki/wiki/fiware/index.php/Publish/Subscribe_Broker_-_Orion_Context_Broker_-_User_and_Programmers_Guide#Entity_service_paths" target="_blank">service path</a>
and entity duple for each one of the databases, storing in these collection the corresponding raw and aggregated data separately.
- "collection-per-attribute": The STH component creates 2 collections per <a href="https://forge.fiware.org/plugins/mediawiki/wiki/fiware/index.php/Publish/Subscribe_Broker_-_Orion_Context_Broker_-_User_and_Programmers_Guide#Entity_service_paths" target="_blank">service path</a>,
entity and attribute triple for each one of the databases, storing in these collection the corresponding raw and aggregated data separately.
As as side note, just mention that the attribute type is not included in the name of the created collections since it is not provided
when querying the STH using the convenience operation provided. This aspect is 100% aligned with the Orion Context Broker where
the attribute type does not have any special semantic or effect currently.

[Top](#section0)

##<a id="section5"></a> Inserting data (random single events and its aggregated data) into the database
The STH component source code includes a set of tests to validate the correct functioning of the component. Amongst these
tests, there is a suite to validate the insertion of aggregated time series information into the MongoDB instance.

### Preconditions
A running instance of a MongoDB database.

### Running the tests
1. To run the tests, just execute:
<pre> make test-database </pre>

The script accepts the following parameters as environment variables:

- SAMPLES: The number of random events which will be generated and inserted into the database. Optional. Default value: "5".
- ENTITY_ID: The id of the entity for which the random event will be generated. Optional. Default value: "entityId".
- ENTITY_TYPE: The type of the entity for which the random event will be generated. Optional. Default value: "entityType".
- ATTRIBUTE_NAME: The id of the attribute for which the random event will be generated. Optional. Default value: "attrName".
- ATTRIBUTE_TYPE: The type of the attribute for which the random event will be generated. Optional. Default value: "attrType".
- START_DATE: The date from which the random events will be generated. Optional. Default value: the beginning of the previous
year to avoid collisions with the testing of the Orion Context Broker notifications which use the current time.
For example if in 2015, the start date is set to "2015-01-01T00:00:00", UTC time. Be very careful if setting the start date,
since these collisions may arise.
- END_DATE: The date before which the random events will be generated. Optional. Default value: the end of the previous
year to avoid collisions with the testing of the Orion Context Broker notifications which use the current time.
For example if in 2015, the end date is set to "2014-12-31T23:59:59", UTC time. Be very careful if setting the start date,
since these collisions may arise.
- MIN_VALUE: The minimum value associated to the random events. Optional. Default value: "0".
- MAX_VALUE: The maximum value associated to the random events. Optional. Default value: "100".
- DB_USERNAME: The username to use for the database connection. Optional. Default value: "".
- DB_PASSWORD: The password to use for the database connection. Optional. Default value: "".
- DB_URI: The URI to use for the database connection. This does not include the 'mongo://' protocol part. Optional. Default value: "localhost:27017".
- DB_NAME: The name of the database to use. Optional. Default value: "test".
- CLEAN: A flag indicating if the generated collections should be removed after the tests. Optional. Default value: "true".

For example, to insert 100 samples on a certain date without cleaning up the database after running the tests, use:
<pre>SAMPLES=100 START_DATE=2015-02-14T00:00:00 END_DATE=2015-02-14T23:59:59 CLEAN=false make test-database</pre>

In case of executing the tests with the CLEAN option set to false, the contents of the database can be inspected using the MongoDB
(```mongo```) shell.

[Top](#section0)

##<a id="section6"></a> STH component complete test coverage
The STH component source code includes a set of tests to validate the correct functioning of the whole set of capabilities
exposed by the component. This set includes:

- Tests to check the connection to the database
- Tests to check the correct starting of the STH component
- Tests to check the STH component correctly deals with all the possible requests it may receive (including invalid URL paths (routes)
as well as all the combinations of possible query parameters) 
- Tests to check the correct aggregate time series information querying after inserting random events (attribute values)
into the database
- Tests to check the correct aggregate time series information generation when receiving (simulated) notifications by a
(fake) Orion Content Broker

### Preconditions
A running instance of a MongoDB database.

### Running the tests
1. To run the tests, just execute:
<pre> make test </pre>

The script accepts the following parameters as environment variables:

- SAMPLES: The number of random events which will be generated and inserted into the database. Optional. Default value: "5".
- ENTITY_ID: The id of the entity for which the random event will be generated. Optional. Default value: "entityId".
- ENTITY_TYPE: The type of the entity for which the random event will be generated. Optional. Default value: "entityType".
- ATTRIBUTE_NAME: The id of the attribute for which the random event will be generated. Optional. Default value: "attrName"
- ATTRIBUTE_TYPE: The type of the attribute for which the random event will be generated. Optional. Default value: "attrType".
- START_DATE: The date from which the random events will be generated. Optional. Default value: the beginning of the previous
year to avoid collisions with the testing of the Orion Context Broker notifications which use the current time.
For example if in 2015, the start date is set to "2015-01-01T00:00:00", UTC time. Be very careful if setting the start date,
since these collisions may arise.
- END_DATE: The date before which the random events will be generated. Optional. Default value: the end of the previous
year to avoid collisions with the testing of the Orion Context Broker notifications which use the current time.
For example if in 2015, the end date is set to "2014-12-31T23:59:59", UTC time. Be very careful if setting the start date,
since these collisions may arise.
- MIN_VALUE: The minimum value associated to the random events. Optional. Default value: "0".
- MAX_VALUE: The maximum value associated to the random events. Optional. Default value: "100".
- DB_USERNAME: The username to use for the database connection. Optional. Default value: "".
- DB_PASSWORD: The password to use for the database connection. Optional. Default value: "".
- DB_URI: The URI to use for the database connection. This does not include the 'mongo://' protocol part. Optional. Default value: "localhost:27017".
- DB_NAME: The name of the database to use. Optional. Default value: "test".
- CLEAN: A flag indicating if the generated collections should be removed after the tests. Optional. Default value: "true".

For example, to run the tests using 100 samples, certain start and end data without cleaning up the database after running
the tests, use:
<pre>SAMPLES=100 START_DATE=2014-02-14T00:00:00 END_DATE=2014-02-14T23:59:59 CLEAN=false make test</pre>

In case of executing the tests with the CLEAN option set to false, the contents of the database can be inspected using the MongoDB
(```mongo```) shell.

[Top](#section0)

##<a id="section6"></a>Contact
* Germán Toro del Valle (<a href="mailto:german.torodelvalle@telefonica.com">german.torodelvalle@telefonica.com</a>, <a href="http://www.twitter.com/gtorodelvalle" target="_blank">@gtorodelvalle</a>)
* Francisco Romero Bueno (<a href="mailto:francisco.romerobueno@telefonica.com">francisco.romerobueno@telefonica.com</a>)
* Iván Arias León (<a href="mailto:ivan.ariasleon@telefonica.com">ivan.ariasleon@telefonica.com</a>)

[Top](#section0)
