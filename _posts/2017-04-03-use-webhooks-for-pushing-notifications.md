---
title: "Use webhooks for pushing notifications"
perex: "Push Costlocker notifications to Slack, ..."
date: 2017-04-03 10:00:00 +0100
icon: https://a.slack-edge.com/ae7f/img/services/outgoing-webhook_512.png
image: https://cloud.githubusercontent.com/assets/26380189/24649765/700b1c7a-1928-11e7-9753-0445bb7dddbc.png
links:
  Blog: "/blog/"
  Webhooks API: http://docs.costlocker.apiary.io/#reference/0/webhooks
  Slack API: https://api.slack.com/incoming-webhooks
  Webhooks debugger: https://requestb.in
  Example code: https://gist.github.com/costlockerbot/db92c7f2064930660b088e7687558221
---

If you're familiar with webhooks it shouldn't take long to integrate webhooks from Costlocker.

1. Create webhooks
1. Edit time entries via API or at [costlocker.com](https://new.costlocker.com/timesheet/detailed)
1. Receive webhooks
1. Delete webhooks

Today we notify you about about created and updated time entries.

### 0. Configuration

Prepare environment variables so curl example are little more readable.

```bash
CL_AUTH="webhooks_test:YOUR_PERSONAL_TOKEN"
CL_API="https://new.costlocker.com/api-public/v2"
CL_WEBHOOK="UUID_OF_EXISTING_WEBHOOK (create variable after first step)"
```

### 1. Create webhooks

Let's prepare webhooks request. I recommend using [https://requestbin.com/](https://requestbin.com/)
for testing webhooks before you start writing production integration.

```bash
curl -X POST -u $CL_AUTH "$CL_API/webhooks" -d @webhooks-request.json
```

_webhooks-request.json_

```json
[
   {
      "url":"http://requestb.in/123v1fw1?all",
      "events":[
         "timeentries.create",
         "timeentries.update"
      ]
   }
]
```

Costlocker returns uuid of created webhook. Save it in the environment variable
[CL_WEBHOOK](#0-configuration).

```json
{
   "meta":{
      "created":1,
      "updated":0
   },
   "data":[
      {
         "uuid":"2113a6ab-7d6a-42e6-852e-e9e7713df981",
         "links":{
            "webhook":"https://new.costlocker.com/api-public/v2/webhooks/2113a6ab-7d6a-42e6-852e-e9e7713df981"
         }
      }
   ]
}
```

### 2. Edit time entries

Trigger some changes in time entries.
You could simply add/edit entries at [costlocker.com](https://new.costlocker.com/timesheet/detailed),
[import time entries from Toggl](/blog/2017-03-08-import-toggl-weekly-report-to-costlocker.html) etc.
This article isn't about creating time entries. So we expect that you somehow edit at least one entry :)

### 3. Receive webhooks

Immediately after editing entries you should see webhook event in `requestb.in`:

![requestbin.com example](https://cloud.githubusercontent.com/assets/7994022/24615706/4feb7370-188f-11e7-859d-64b600c27e03.png)

_**Tip**: use [/webhooks/uuid/test](http://docs.costlocker.apiary.io/#reference/0/webhooks/get-webhook-example) endpoint 
for getting event structure without triggering any change._

```bash
curl -X GET -u $CL_AUTH "$CL_API/webhooks/$CL_WEBHOOK/test"
```

### 4. Delete webhooks

If you don’t need webhook anymore you can delete it.

```bash
curl -X DELETE -u $CL_AUTH "$CL_API/webhooks/$CL_WEBHOOK"
```

### Slack notifications

Now you can push notifications to Slack. Add Slack webhook to `url` in [1st step](#1-create-webhooks).
Check [slack documentation](https://api.slack.com/incoming-webhooks) if you've never worked with Slack webhooks.

Don't forget to transform Costlocker events to
[Slack message](https://api.slack.com/incoming-webhooks#advanced_message_formatting).
You can test it [without a running server](https://gist.github.com/costlockerbot/db92c7f2064930660b088e7687558221)
by copy-pasting payload from [https://requestbin.com/](https://requestbin.com/).
Or test your message in [Message Builder](https://api.slack.com/docs/messages/builder).

![Slack message preview](https://cloud.githubusercontent.com/assets/26380189/24649765/700b1c7a-1928-11e7-9753-0445bb7dddbc.png)

**Did you find a mistake? Is something unclear or isn't the code working?
[Help us to improve the article](https://github.com/costlocker/costlocker.github.io/issues)!**
