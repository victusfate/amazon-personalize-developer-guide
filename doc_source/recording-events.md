# Recording Events<a name="recording-events"></a>

Amazon Personalize can make recommendations based purely on historical imported data as demonstrated in the [Getting Started](getting-started.md) guides\. Amazon Personalize can also make recommendations purely on real time event data, or a combination of both, using the Amazon Personalize event ingestion SDK\.

Unlike historical data, after a campaign is created, new recorded event data is automatically used when getting recommendations from the campaign\.

**Note**  
The minimum data requirements to train a model are:  
1000 records of combined interaction data \(after filtering by `eventType` and `eventValueThreshold`, if provided\)
25 unique users with at least 2 interactions each

The event ingestion SDKs include a JavaScript library for recording events from web client applications\. The SDKs also include a library for recording events in server code\.

To record events, you need the following:
+ A dataset group that includes an `Interactions` dataset, which can be empty\.
+ An event tracker\.
+ A call to the [PutEvents](API_UBS_PutEvents.md) operation\.

 Amazon Personalize predefined recipes are compatible with the `PutEvents` API\. This includes the required `sessionId` parameter which makes these recipes aware of sessions\.

**Topics**
+ [Creating a Dataset Group](#event-dataset-group)
+ [Getting a Tracking ID](#event-get-tracker)
+ [Event\-Interactions Dataset](#event-dataset)
+ [PutEvents Operation](#event-record-api)
+ [Event Metrics](#event-metrics)
+ [Events and Solutions](#event-solutions)

## Creating a Dataset Group<a name="event-dataset-group"></a>

If you went through the [Getting Started](getting-started.md) guide, you can use the same dataset group that you created\. For a Python example that creates a dataset and a dataset group, see [Importing Your Data](data-prep-importing.md)\.

## Getting a Tracking ID<a name="event-get-tracker"></a>

A tracking ID associates an event with a dataset group and authorizes you to send data to Amazon Personalize\. You generate a tracking ID by calling the [CreateEventTracker](API_CreateEventTracker.md) API and supplying the dataset group ARN\.

**Note**  
Only one event tracker can be associated with a dataset group\. You will get an error if you call `CreateEventTracker` using the same dataset group as an existing event tracker\.

------
#### [ Python ]

```
import boto3

personalize = boto3.client('personalize')

response = personalize.create_event_tracker(
    name='MovieClickTracker',
    datasetGroupArn='arn:aws:personalize:us-west-2:acct-id:dataset-group/MovieClickGroup'
)
print(response['eventTrackerArn'])
print(response['trackingId'])
```

------
#### [ AWS CLI ]

```
aws personalize create-event-tracker \
    --name MovieClickTracker \
    --dataset-group-arn arn:aws:personalize:us-west-2:acct-id:dataset-group/MovieClickGroup
```

------

The event tracker ARN and tracking ID are displayed, for example:

```
{
    "eventTrackerArn": "arn:aws:personalize:us-west-2:acct-id:event-tracker/MovieClickTracker",
    "trackingId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

## Event\-Interactions Dataset<a name="event-dataset"></a>

When Amazon Personalize creates an event tracker, it also creates an event\-interactions dataset in the dataset group associated with the event tracker\. The event\-interactions dataset stores the event data from the [PutEvents](API_UBS_PutEvents.md) call\. The contents of the dataset are not available to the user\.

To view the properties of the dataset, call the [ListDatasets](API_ListDatasets.md) API, supplying the dataset group ARN\. For additional information about the dataset, use the dataset ARN for the EVENT\_INTERACTIONS dataset to call the [DescribeDataset](API_DescribeDataset.md) API\. The following is an example response from `ListDatasets`:

```
{
    "datasets": [
        {
            "name": "ratings-dsgroup/EVENT_INTERACTIONS",
            "datasetArn": "arn:aws:personalize:us-west-2:acct-id:dataset/MovieClickGroup/EVENT_INTERACTIONS",
            "datasetType": "EVENT_INTERACTIONS",
            "status": "ACTIVE",
            "creationDateTime": 1554304597.806,
            "lastUpdatedDateTime": 1554304597.806
        },
        {
            "name": "ratings-dataset",
            "datasetArn": "arn:aws:personalize:us-west-2:acct-id:dataset/MovieClickGroup/INTERACTIONS",
            "datasetType": "INTERACTIONS",
            "status": "ACTIVE",
            "creationDateTime": 1554299406.53,
            "lastUpdatedDateTime": 1554299406.53
        }
    ],
    "nextToken": "..."
}
```

## PutEvents Operation<a name="event-record-api"></a>

To record events, you call the [PutEvents](API_UBS_PutEvents.md) operation\. The following example shows a `PutEvents` call that passes one event that contains the minimum required information\. The corresponding Interactions schema is shown, along with an example row from the Interactions dataset\.

The session ID is custom to your application\. The event list is an array of [Event](API_UBS_Event.md) objects\. The `properties` key is a string map \(key\-value pairs\) of event\-specific data\. In this case, just the item ID is specified\.

The `userId`, `itemId`, and `sentAt` parameters map to the USER\_ID, ITEM\_ID, and TIMESTAMP fields of a corresponding historical `Interactions` dataset\. For more information, see [Datasets and Schemas](how-it-works-dataset-schema.md)\.

**Note**  
You can also use AWS Amplify to send event data to Amazon Personalize\. For more information, see [Amplify \- Analytics](https://aws-amplify.github.io/docs/js/analytics)\.

```
Interactions schema: USER_ID, ITEM_ID, TIMESTAMP
Interactions dataset: user123, item-xyz, 1543631760
```

------
#### [ Python ]

```
import boto3

personalize_events = boto3.client(service_name='personalize-events')

personalize_events.put_events(
    trackingId = 'tracking_id',
    userId= 'USER_ID',
    sessionId = 'session_id',
    eventList = [{
        'sentAt': TIMESTAMP,
        'eventType': 'EVENT_TYPE',
        'properties': "{\"itemId\": \"ITEM_ID\"}"
        }]
)
```

------
#### [ AWS CLI ]

```
aws personalize-events put-events \
    --tracking-id tracking_id \
    --user-id USER_ID \
    --session-id session_id \
    --event-list '[{
        "sentAt": "TIMESTAMP",
        "eventType": "EVENT_TYPE",
        "properties": "{\"itemId\": \"ITEM_ID\"}"
      }]'
```

------

After this example, you would proceed to train a model using only the required properties\. The training would not use the `eventValue` property because it wasn't included in the event\.

The next example shows how to submit data that would train on the event value\. It also demonstrates the passing of multiple events of different types \('like' and 'rating'\)\. In this case, you must specify the event type to train on in the [CreateSolution](API_CreateSolution.md) operation \(see **Events and Solutions** below\)\. Also shown is the recording of an extra property, `numRatings`, that is used as metadata by certain recipes\.

```
Interactions schema: USER_ID, ITEM_ID, TIMESTAMP, EVENT_TYPE, EVENT_VALUE, NUM_RATINGS
Interactions dataset: user123, movie_xyz, 1543531139, rating, 5, 12
                      user321, choc-ghana, 1543531760, like, true
                      user111, choc-fake, 1543557118, like, false
```

------
#### [ Python ]

```
import boto3
import json

personalize_events = boto3.client(service_name='personalize-events')

personalize_events.put_events(
    trackingId = 'tracking_id',
    userId= 'user555',
    sessionId = 'session1',
    eventList = [{
        'eventId': 'event1',
        'sentAt': '1553631760',
        'eventType': 'like',
        'properties': json.dumps({
            'itemId': 'choc-panama',
            'eventValue': 'true'
            })
        }, {
        'eventId': 'event2',
        'sentAt': '1553631782',
        'eventType': 'rating',
        'properties': json.dumps({
            'itemId': 'movie_ten',
            'eventValue': '4',
            'numRatings': '13'
            })
        }]
)
```

------
#### [ AWS CLI ]

```
aws personalize-events put-events \
    --tracking-id tracking_id \
    --user-id user555 \
    --session-id session1 \
    --event-list '[{
        "eventId": "event1",
        "sentAt": "1553631760",
        "eventType": "like",
        "properties": "{\"itemId\": \"choc-panama\", \"eventValue\": \"true\"}"
      }, {
        "eventId": "event2",
        "sentAt": "1553631782",
        "eventType": "rating",
        "properties": "{\"itemId\": \"movie_ten\", \"eventValue\": \"4\", \"numRatings\": \"13\"}"
      }]'
```

------

**Note**  
The properties keys use camel case names that match the fields in the Interactions schema\. For example, if the fields 'ITEM\_ID', 'EVENT\_VALUE', and 'NUM\_RATINGS,' are defined in the Interactions schema, the property keys should be `itemId, eventValue, and numRatings`\.

## Event Metrics<a name="event-metrics"></a>

To monitor the type and number of events sent to Amazon Personalize, use Amazon CloudWatch metrics\. For more information, see [Monitoring Amazon Personalize](personalize-monitoring.md)\. 

## Events and Solutions<a name="event-solutions"></a>

When training a model that uses event data, two parameters of the [CreateSolution](API_CreateSolution.md) operation are relevant\. The `eventType` parameter must be specified when multiple event types are recorded\. The `eventType` indicates which type of event Amazon Personalize uses as the label for model training\.

The `eventValueThreshold` parameter of the `SolutionConfig` object creates an event filter\. When this parameter is specified, only events with a value greater than or equal to the threshold are used for training the model\. You must specify the event type when using `eventValueThreshold`\.