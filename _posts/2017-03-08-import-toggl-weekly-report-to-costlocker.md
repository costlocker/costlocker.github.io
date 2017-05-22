---
title: "Import Toggl weekly report to Costlocker"
perex: "Automate synchronization between Toggl and Costlocker"
date: 2017-03-08 10:00:00 +0100
image: https://cloud.githubusercontent.com/assets/7994022/24607599/e3c37312-1872-11e7-82d2-e4ce7d7c490a.png
links:
  Blog: "/blog/"
  Toggl API: https://github.com/toggl/toggl_api_docs
  Costlocker API: http://docs.costlocker.apiary.io/
  PHP code: https://gist.github.com/costlockerbot/2e9f86340dd4f4967bb0ebc69dc623ea
---

Are you using Toggl as your primary tracking tool instead of Costlocker?
Do you still want to know the profitability of your business and employees?
No problem - Costlocker is now able to simply import all your Toggl tracked hours and process them.

Let’s get started and import your [Toggl weekly report](https://www.toggl.com/app/reports/weekly/)
to the [Costlocker timesheet](https://new-n1.costlocker.com/timesheet/weekly).
It’s a classic [ETL](https://en.wikipedia.org/wiki/Extract,_transform,_load) process:

1. Extract your weekly report from Toggl
1. Transform Toggl entries to Costlocker entries
1. Load the time entries in Costlocker

**tldr;** [check the PHP code if you prefer reading code](https://gist.github.com/costlockerbot/2e9f86340dd4f4967bb0ebc69dc623ea)

### 0. Configuration

[Toggl](https://www.toggl.com/app/profile) and 
[Costlocker](https://new.costlocker.com/api-token)
use authentication with personal access tokens.
It is neccessary to retrieve valid tokens and find a
[Toggl workspace id](https://github.com/toggl/toggl_api_docs/blob/master/reports.md#request-parameters).

```php?start_inline=true
$config = [
    'costlocker' => [
        'token' => 'COSTLOCKER_TOKEN',
    ],
    'toggl' => [
        'token' => 'TOGGL_TOKEN',
        'workspace' => 'TOGGL_WORKSPACE',
        'selectedWeek' => date('Y-m-d', strtotime('last monday')),
    ],
    'mapping' => [
        'projects' => [
        ],
        'users' => [
        ],
    ],
];
```

After the configuration is known, just follow the ETL process:

```php?start_inline=true
$togglWeeklyReport = getTogglWeeklyReport($config);
$costlockerEntries = convertTogglWeeklyReport($togglWeeklyReport, $config);
$createdEntries = createCostlockerEntries($costlockerEntries, $config);
```

### 1. Extract your weekly report from Toggl

Toggl offers [weekly report](https://github.com/toggl/toggl_api_docs/blob/master/reports/weekly.md)
in its API. 
Note that the API returns empty entries and aggregation for the week.
We have to ignore these entries in the next step.

```php?start_inline=true
function getTogglWeeklyReport(array $config)
{
    return httpRequest([
        'url' => 'https://toggl.com/reports/api/v2/weekly',
        'query' => [
            'workspace_id' => $config['toggl']['workspace'],
            'since' => $config['toggl']['selectedWeek'],
            'project_ids' => implode(',', array_keys($config['mapping']['projects'])),
            'user_ids' => implode(',', array_keys($config['mapping']['users'])),
            'user_agent' => 'api_test',
        ],
        'auth' => [
            $config['toggl']['token'],
            'api_token',
        ],
    ]);
}
```

### 2. Transform Toggl entries to Costlocker entries

Mapping entities between Toggl and Costlocker is **the hardest part** of the integration, 
because it’s not possible to just copy-paste the examples. 
You’ll have to define the rules for creating Costlocker assignments:

```
{
    "person_id": 1,
    "project_id": 1,
    "activity_id": 1,
    "task_id": null
}
```

You can map a toggl’s [`pid`](https://github.com/toggl/toggl_api_docs/blob/master/chapters/time_entries.md) 
to a `project_id/activity_id` and 
a [`uid`](https://github.com/toggl/toggl_api_docs/blob/master/chapters/time_entries.md) to a `person_id`.
If you are a premium Toggl user, you can map tasks as well.

```php?start_inline=true
$config = [
    'mapping' => [
        'projects' => [
            'TOGGL_PID' => [
                'project_id' => COSTLOCKER_PROJECT_ID,
                'activity_id' => COSTLOCKER_ACTIVITY_ID,
            ],
        ],
        'users' => [
            'TOGGL_UID' => [
                'person_id' => COSTLOCKER_PERSON_ID,
            ],
        ],
    ],
];
```

The final conversion is simple. Just transform one object to another...


```php?start_inline=true
function convertTogglWeeklyReport(array $togglWeeklyReport, array $config)
{
    $costlockerEntries = [];
 
    foreach ($togglWeeklyReport['data'] as $project) {
        foreach ($project['details'] as $user) {
            foreach ($user['totals'] as $dayIndex => $miliseconds) {
                if ($miliseconds && $dayIndex < 7) {
                    $costlockerEntries[] = [
                        "date" => date('Y-m-d H:i:s', strtotime("{$config['toggl']['selectedWeek']} + {$dayIndex} day 8:00")), 
                        "duration" => $miliseconds / 1000,
                        "description" => "Toggl import - user '{$user['title']['user']}', project '{$project['title']['project']}', client '{$project['title']['client']}'",
                        'assignment' => $config['mapping']['users'][$user['uid']] + $config['mapping']['projects'][$project['pid']],
                    ];
                }
            }
        }
    }
 
    return $costlockerEntries;
}
```

_**Tip:** Costlocker supports upsert. If you fill [`uuid`](http://docs.costlocker.apiary.io/#reference/0/time-entries/create/update-time-entries) field of existing time-entry, we won't create a new one, but update an existing one._

### 3. Load the time entries in Costlocker

Push entries to
[/api-public/v2/timeentries/](http://docs.costlocker.apiary.io/#reference/0/time-entries/create/update-time-entries).
Costlocker returns uuids of created/updated entries.

```php?start_inline=true
function createCostlockerEntries(array $costlockerEntries, array $config)
{
    return httpRequest([
        'url' => 'https://new.costlocker.com/api-public/v2/timeentries/',
        'json' => $costlockerEntries,
        'auth' => [
            'costlocker/import-toggl-entries',
            $config['costlocker']['token'],
        ],
    ]);
}
```

### 4. Compare reports in Toggl and Costlocker

If nothing went wrong, you should see identical weekly reports in both Toggl and Costlocker.

[Toggl weekly report](https://www.toggl.com/app/reports/weekly/)

![Toggl weekly report](https://cloud.githubusercontent.com/assets/7994022/24607599/e3c37312-1872-11e7-82d2-e4ce7d7c490a.png)

[Costlocker weekly report](https://new-n1.costlocker.com/timesheet/weekly)

![Costlocker weekly report](https://cloud.githubusercontent.com/assets/7994022/24607626/02cc69d0-1873-11e7-96a9-f73ee84f58ba.png)

**Did you find a mistake? Is something unclear or isn't the code working?
[Help us to improve the article](https://github.com/costlocker/costlocker.github.io/issues)!**
