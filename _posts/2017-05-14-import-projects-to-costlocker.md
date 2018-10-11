---
title: "Import projects to Costlocker"
perex: "Automate Costlocker projects management via API v2"
date: 2017-05-14 10:00:00 +0100
icon: https://www.getharvest.com/assets/press/harvest-logo-icon-77a6f855102e2f85a7fbe070575f293346a643c371a49ceff341d2814e270468.png
image: https://user-images.githubusercontent.com/7994022/29269015-b2eba44a-80ef-11e7-923e-dd97f0fdb86c.png
links:
  Blog: "/blog/"
  Projects API: http://docs.costlocker.apiary.io/#reference/0/projects/create/update-projects
  Import Harvest projects (demo): https://harvest.integrations.costlocker.com
  Import Harvest projects (source code): https://gitlab.com/costlocker/integrations/tree/master/harvest
---

Are you using Harvest, Teamwork, JIRA, Asana, Toggl, &hellip; but missing
some functionality like profits, revenues, etc&hellip;?
Costlocker is now able to create and update projects via its API.

* [Project API introduction](#project-api-introduction)
* [_Project items_](#items) - [personnel costs (activity, person, task)](#personnel-costs),
  [project expenses](#project-expenses), [discount](#discount), [billing](#billing)
* [API shortcuts](#api-shortcuts)
* [_Demo_ - Import projects from Harvest](#demo-import-projects-from-harvest)

_**{{ '2018-04-26' | date: '%B %-d, %Y' }}**: we've added [project budgets](#budgets)_

_**{{ '2018-09-11' | date: '%B %-d, %Y' }}**: we've added [no-budget](#no-budget-no_budget)_

## Project API introduction

We tried to unify the project structure as much as possible.
If you're familiar with 
[/api-public/v2/projects/ endpoint](http://docs.costlocker.apiary.io/#reference/0/projects)
it should be simple to execute any operation with project.

### Project detail

Basic project information is available in the standard json format.
More advanced concepts (personnel costs, billing, &hellip;) are defined in [`items`](#items).

```json
{
   "id":"123456",
   "name":"Website",
   "client": "Google",
   "dates":{
      "date_start":"2017-03-20",
      "date_end":"2017-05-20"
   },
   "tags":["Billable"],
   "responsible_people": ["test@example.com"],
   "budget": {
      "type": "time_estimates.person_activity"
   },
   "items":[]
}
```

### Items

Field `items` are used for defining personnel costs, project expenses, billing and discount.
The returned item is always of the same format:

* The `item` contains `type` and Costlocker ids
* The item is described in remaining fields (typically, there is `<entity>_id` in the `item` and the detail in the `<entity>` field)
* The response after creating or updating projects contains only the `item` (_you can use returned ids for storing mapping information_)
* The `action` is optional, the `upsert` operation is used by default. You can use the `delete` action to delete any items

```json
{
   "items":[
      {
         "action": "upsert",
         "item":{
            "type":"expense",
            "expense_id":"123457"
         },
         "expense":{
            "description":"Invision",
            "purchased":{
               "total_amount":500,
               "date":null
            },
            "billed":{
               "total_amount":500
            }
         }
      }
   ]
}
```

## Personnel costs

You can define an activity, a person and a task. We recommend to create just _leafs_ though.
Activity budget represents an aggregation of personal budgets. The personal budget is an aggregation of tasks.
This way, you can just create the leaf and the activity/person is upserted.

### Activity

Every activity has a name and an hourly rate. The existing client rate is used if the `hourly_rate` is missing.
If you create a new activity without an hourly rate, zero rate is used.

```json
{
    "item": {
        "type": "activity",
        "activity_id": 123456
    },
    "activity": {
        "name": "Social media",
        "hourly_rate": 100
    }
}
```

### Person

It is necessary to specify the email, names, role and salary for every new person.
The role, salary and names are used only when creating a new person. _An existing person is never updated!_

```json
{
    "item": {
        "type": "person",
        "activity_id": 123456,
        "person_id": 123456
    },
    "hours": {
        "budget": 5
    },
    "activity": {
        "name": "Social media",
        "hourly_rate": 20
    },
    "person": {
        "email": "manager@example.com",
        "first_name": "Monthly",
        "last_name": "Salary",
        "role": "MANAGER",
        "salary": {
            "payment": "monthly",
            "salary": 10000,
            "hours": 160,
            "date_start": "2017-02-01 10:00:00"
        }
    }
}
```

_The same person definition can be used in `responsible_people` field._

### Task

Tasks are assigned to a person and activity. A parent can be referenced by ids in the `item`
 or by a name/email in the relevant field. References are also described in [shortcuts](#api-shortcuts).

```json
{
    "item": {
        "type": "task",
        "activity_id": 123456,
        "person_id": 123456,
        "task_id": 123456
    },
    "hours": {
        "budget": 20
    },
    "task": {
        "name": "new task"
    },
    "person": {
        "email": "employee@example.com",
        "first_name": "Hourly",
        "last_name": "Rate",
        "role": "ADMIN",
        "salary": {
            "payment": "hourly",
            "hourly_rate": 0
        }
    }
}
```


## Project expenses

Partial updates are supported. Please see the example in [billing](#billing).

```json
{
    "item": {
        "type": "expense",
        "expense_id": 123457
    },
    "expense": {
        "description": "new description",
        "purchased": {
            "total_amount": 1000,
            "date": "2017-05-15"
        },
        "billed": {
            "total_amount": 500
        }
    }
}
```

## Discount

Every project has one discount. No ids are used, there is only one amount that needs to be adjusted.

```json
{
    "item": {
        "type": "discount"
    },
    "discount": {
        "total_amount": 100
    }
}
```

## Billing

Be aware that revenue (_personnel costs + project expenses - discount_) cannot be smaller than billing.

```json
{
    "item": {
        "type": "billing"
    },
    "billing": {
        "description": "INV201705150001",
        "total_amount": 900,
        "date": "2017-05-15",
        "status": "draft"
    }
}
```

You can execute partial updates, such as changing the billing status without modifying the amount&hellip;

```json
{
    "item": {
        "type": "billing",
        "billing_id": 123456
    },
    "billing": {
        "status": "sent"
    }
}
```

---

## Budgets

We support five budget types since [April  2018](https://costlocker.docs.apiary.io/#introduction/changelog/april-2018):

| `type` | Description |
| ------ | ----------- |
| `time_estimates.person_activity` | Activity hourly rate * Person hours estimates |
| `time_estimates.activity` | Activity hourly rate * Activity hours estimate |
| `timesheet` | Activity hourly rate * Tracked hours |
| `no_budget` | All tracked hours are non-billable |
| `fixed_price.activity` | Activity fixed price |
| `fixed_price.project` | Project fixed price |

Below you can see what fields are required for each budget type.
You can send for example person hours budget to timesheet budget, but we'll ignore it
_(webhook would contain zero hours budget)_.

![Budget fields](https://user-images.githubusercontent.com/7994022/39293649-b40244e8-4939-11e8-97eb-1fb5f4a6abe3.png)

### Person estimates (`time_estimates.person_activity`)

Budget is defined in `person` and `activity` item.
Take a look at [personnel costs](#personnel-costs).

```json
{
    "item": {
        "type": "person",
        "activity_id": 123456,
        "person_id": 123456
    },
    "activity": {
        "name": "Social media",
        "hourly_rate": 20
    },
    "hours": {
        "budget": 5
    },
    "person": "manager@example.com"
}
```
### Activity estimates (`time_estimates.activity`)

Budget is defined in `activity` item:

```json
{
    "item": {
        "type": "activity",
        "activity_id": 123456
    },
    "activity": {
        "name": "Social media",
        "hourly_rate": 100
    },
    "hours": {
        "budget": 20
    }
}
```

### Timesheet (`timesheet`)

Budget is defined in `activity` item:

```json
{
    "item": {
        "type": "activity",
        "activity_id": 123456
    },
    "activity": {
        "name": "Social media",
        "hourly_rate": 100
    }
}
```

### No budget (`no_budget`)

Hourly rate or budget amount is ignored. Tracked time is never billed!

```json
{
    "item": {
        "type": "activity",
        "activity_id": 123456
    },
    "activity": {
        "name": "Social media"
    }
}
```

### Activity fixed price (`fixed_price.activity`)

Budget is defined in `activity` item:

```json
{
    "item": {
        "type": "activity",
        "activity_id": 123456
    },
    "activity": {
        "name": "Social media"
    },
    "budget": {
        "total_amount": 2000
    }
}
```

### Project fixed price (`fixed_price.project`)

Budget is defined in `project` item:

```json
{
    "item": {
        "type": "project"
    },
    "budget": {
        "total_amount": 2000
    }
}
```

---

## API shortcuts

You can use shortcuts when creating a new project or experimenting with the API.
This is an experimental feature for developers. If you don’t like it, you can use 
the standard formats returned by the API.

| Shortcut | Full representation | Description |
| -------- | ------------------- | ----------- |
| `item` | `item.type` | Item type definition |
| `activity` | `activity.name` | Activity referenced by name |
| `person` | `person.email` | Person referenced by email |
| `task` | `task.name` | Task referenced by name |
| `hours` | `hours.budget` | Hours budget |
| `client` | `client.id` or `client.name` | Client referenced by id or name |
| `tag` | `tag.id` or `rag.name` | Tag referenced by id or name |


Let's say you want to create a new task for an existing person:

```json
{
   "name":"Shortcuts experiment",
   "client":"Google",
   "responsible_people":[
      "existing-perso@example.com",
      {
         "email":"manager@example.com"
      }
   ],
   "tags":[
      "social",
      {
         "name":"Billable"
      }
   ],
   "items":[
      {
         "item":"task",
         "task":"new task",
         "activity":"Social media",
         "person":"existing-person@example.com",
         "hours":5
      }
   ]
}
```

Be aware that shortcuts are only used in requests.
A response always contains standard `item` definition.

```json
{
   "id": "2707",
   "items":[
      {
         "action":"upsert",
         "item":{
            "type":"task",
            "activity_id":579,
            "person_id":123456,
            "task_id":"855"
         }
      }
   ]
}
```

## _Demo:_ Import projects from Harvest

We've built _Harvest projects importer_ during development of the new API, try it at 
[https://harvest.integrations.costlocker.com](https://harvest.integrations.costlocker.com)
([source code](https://gitlab.com/costlocker/integrations/tree/master/harvest)).

**Let us know if you've created similar application that connects Costlocker to a project management tool!**

![Import harvest projects to Costlocker](https://user-images.githubusercontent.com/7994022/29269015-b2eba44a-80ef-11e7-923e-dd97f0fdb86c.png)

---

**Did you find a mistake? Is something unclear or isn't the code working?
[Help us to improve the article](https://github.com/costlocker/costlocker.github.io/issues)!**
