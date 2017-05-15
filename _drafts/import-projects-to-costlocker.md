---
title: "Import projects to Costlocker"
perex: "Automate managing Costlocker projects via API v2"
date: 2017-05-14 10:00:00 +0100
image: https://cloud.githubusercontent.com/assets/7994022/26050527/43dbc29c-395f-11e7-9902-4585f9f7f463.png
links:
  Blog: "/blog/"
  Projects API: http://docs.costlocker.apiary.io/#reference/0/projects/create/update-projects
  Import Harvest projects (demo): http://integrations.costlocker.com
  Import Harvest projects (source code): https://github.com/costlocker/integrations/harvest
---

Are you using Harvest, Teamwork, JIRA, Asana, Toggl, &hellip; but you are missing
some functionality like profits, revenues, &hellip;?
Costlocker is now able to create and update projects via API.

* [Projects API introduction](#projects-api-introduction)
* [_Project items_](#items) - [people costs (activity, person, task)](#people-costs),
  [project expenses](#project-expenses), [discount](#discount), [billing](#billing)
* [API shortcuts](#api-shortcuts)
* [_Demo_ - Import projects from Harvest](#demo-import-projects-from-harvest)

## Projects API introduction

We tried to unify project structure as much as possible.
If you're get familiar with 
[/api-public/v2/projects/ endpoint](http://docs.costlocker.apiary.io/#reference/0/projects)
it should be simple to execute any operation with project.

### Project detail

Basic information about projects are available in classic json.
More advanced concepts (people costs, billing, &hellip;) are defined in [`items`](#items).

```
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
   "items":[]
}
```

### Items

Field `items` is used for defining people costs, project expenses, billing and discount.
Returned item has always same format:

* `item` contains `type` and Costlocker ids
* item is described in remaining fields (typically you have `<entity>_id` in `item` and detail in `<entity>` field)
* response after creating/updating projects contains only `item` (_you can use returned ids for storing mapping information_)
* `action` is optional, by default `upsert` operation is used. You can use `delete` action for deleting item

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

## People costs

You can define an activity, a person and a task. But we recommend to create just _leafs_.
Activity budget is aggregation of person budgets. Person budget is aggregation of tasks.
So you can just create the leaf and activity/person is upserted.

### Activity

Every activity has name and hourly rate. Existing client rate is used if `hourly_rate` is missing.
Zero rate is used, if you create a new activity without hourly rate.

```json
{
    "item": {
        "type": "activity"
    },
    "activity": {
        "name": "Social media",
        "hourly_rate": 100
    }
}
```

### Person

You must specify email, names, role and salary for new persons.
Role, salary and names are used only for creating new person. _Existing person is never updated!_

```
{
    "item": {
        "type": "person"
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

_Same person definition can be used in `responsible_people` field._

### Task

Task is assigned to person and activity. Parent can be referenced by ids in `item`
or by name/email in relevant field. References are also described in [shortcuts](#api-shortcuts).

```
{
    "item": {
        "type": "task",
        "activity_id": 123456
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

Partial updates are supported, take a look at example in [billing](#billing)

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

Every project has one discount. No ids are used, you are just setting an amount

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

Be aware that revenue (_people costs + project expenses - discount_) cannot be smaller than billing

```json
{
    "item": {
        "type": "billing"
    },
    "billing": {
        "description": "INV201705150001",
        "total_amount": 900,
        "date": "2017-05-15",
        "status": "issued"
    }
}
```

You can do partial update, like changing billing status without modifying amount&hellip;

```
{
    "item": {
        "type": "billing",
        "billing_id": 123456
    },
    "billing": {
        "status": "invoiced"
    }
}
```

---

## API shortcuts

You can use shortcuts when you are creating new project or experimenting with the API.
Take it as experimental feature for developers. If you don't like it you can use
standard format returned by the API.

| Shortcut | Full representation | Description |
| -------- | ------------------- | ----------- |
| `item` | `item.type` | Item type definition |
| `activity` | `activity.name` | Activity referenced by name |
| `person` | `person.email` | Person referenced by email |
| `task` | `task.name` | Task referenced by name |
| `hours` | `hours.budget` | Hours budget |
| `client` | `client.id` or `client.name` | Client referenced by id or name |
| `tag` | `tag.id` or `rag.name` | Tag referenced by id or name |


Let's say that you want to create a new task for existing person:

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

Be aware that shortcuts are used only in requests.
Response always contains standard `item` definition.

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
[http://integrations.costlocker.com](http://integrations.costlocker.com)
([source code](https://github.com/costlocker/integrations/harvest)).

**Let us know if you've created similar application that connects Costlocker with a project management tool!**

![Import harvest projects to Costlocker](https://cloud.githubusercontent.com/assets/7994022/26050527/43dbc29c-395f-11e7-9902-4585f9f7f463.png)

---

**Did you find a mistake? Is something unclear or the code isn't working?
[Help us to improve the article](https://github.com/costlocker/costlocker.github.io/issues)!**
