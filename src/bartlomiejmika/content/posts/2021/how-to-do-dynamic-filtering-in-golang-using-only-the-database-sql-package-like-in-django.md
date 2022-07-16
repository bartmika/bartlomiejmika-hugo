---
title: "How to do Dynamic Filtering in Golang using only the Database SQL Package like in Django"
subtitle: "Learn how to add filters to your list functions in Golang using the database/sql package"
date: 2021-05-07T15:15:47-04:00
lastmod: 2022-02-28T20:12:32-05:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Web Development"
tags:
- "Golang"
- "SQL"
cover:
    image: "/img/2021/common/go-banner.png"
    alt: "Go"
    caption: "" # display caption under cover
    relative: false # when using page bundles set this to true
    hidden: false # only hide on current single page
---
Recently I had to do some advanced filtering using the [``database/sql``](https://golang.org/pkg/database/sql/) package found in the Golang standard library. In this article I share how I handled advanced filtering. If you are a beginner in Golang wanting to learn how to improve your understanding of querying the database, this article is for you.

<!--more-->

This article is broken up as follows:

* Context
* Setting up the problem in Python
* Setting up the problem in Golang
* Understand the problem
* First attempt (Static filtering)
* Second attempt (Too complicated)
* The “variadic function” to the rescue
* The Solution
* Extra Solutions

Let's begin!

# Context
Before using Golang to write web-applications, I've was using [``Django Web Framework``](https://www.djangoproject.com) which is written in Python. Django provides a [object relational-mapping (ORM)](https://docs.djangoproject.com/en/3.2/topics/db/models/) system to handle your python code interacting with the database. Through all my years, I still think the Django ORM is *the best I have ever seen*.

Now that I switched my programming from Python to Golang, I no longer have access to this *phenomenal library*. There are other ORM solutions but none that I've found to my liking; as a result, I am using only the [``database/sql``](https://golang.org/pkg/database/sql/) package.

The transition from Django to Golang has not been easy due to fact of how much Django *spoiled me*. Django has *abstracted away* a lot algorithms and computer science theory for me to use in a easy to use plug-in-play code! With the transition I am going through, this situation reminds me of a great article called [Is Software Abstraction Killing Civilization?](https://www.datagubbe.se/endofciv/) in which it details talks about how programmers lose certain knowledge through time because we keep using abstracted code; as a result, I'd I am writing this article to detail my learning.

# Setting up the problem in Django

Let's assume we have the following data-structure and you populated the table with some dummy data:

{{< highlight sql "linenos=false">}}
CREATE TABLE things (
    tenant_id BIGINT NOT NULL,
    id BIGSERIAL PRIMARY KEY,
    manufacturer_id BIGINT NOT NULL,
    part_id BIGINT NOT NULL,
    state SMALLINT NOT NULL,
    name VARCHAR (127) NOT NULL
);
{{</ highlight >}}

If we want to lookup "3D Printer" in the ``name`` then we would write:

{{< highlight python "linenos=false">}}
from django.db.models import Q

series = Thing.objects.filter(name="3D Printer")
{{</ highlight >}}

Or if we want to lookup a ``manufacturer_id`` which does NOT have a particular ``part_id``, we would write:

{{< highlight python "linenos=false">}}
from django.db.models import Q

series = Thing.objects.filter(
    Q(manufacturer_id=1) &
    ~Q(part_id=444)
)
{{</ highlight >}}

And how about if we lookup all the things with the ``name`` of 3D printers with a particular ``part_id``?

{{< highlight python "linenos=false">}}
from django.db.models import Q

series = Thing.objects.filter(
    Q(name="3D Printer") &
    Q(part_id=123456789)
)
{{</ highlight >}}


**So the Django ORM is quite good at being flexible enough to handle whatever filtering you can throw at it. How do we build something so flexible in Golang?**

# Setting up the problem in Golang

I will provide important code snippets to help demonstrate. Please note that this code is following the [repository pattern](https://adodd.net/posts/go-ddd-repository-pattern/) so if you don't understand then please read up before proceeding further.

When following the **repository pattern**, I break up my code into the *models* and *repositories* packages.

* **models** - Is responsible strictly for the data structure.
* **repositories** - Will implement the interface from *models*.

The code is as follows:

### models/thing.go

```golang
// github.com/bartmika/hello-server/internal/models/thing.go
package models

import (
    "context"
    "time"

    null "gopkg.in/guregu/null.v4"
)

// Structure used to hold filtering request by the user. If using `null` package
// then filter is optional, else the value is required!
type ThingFilter struct {
	TenantId       uint64   `json:"tenant_id"`
	ManufacturerId null.Int `json:"manufacturer_id"`
	PartId         null.Int `json:"part_id"`
	State          null.Int `json:"state"`
	LastSeenId     uint64   `json:"last_seen_id"`
	Limit          uint64   `json:"limit"`
}

// The interface we will be using for our `Thing` model. Please note we will be
// using the `repository pattern` so we are adding `Repo` suffix to remind us.
type ThingRepo interface {
    ListByFilter(ctx context.Context, filter *ThingFilter) ([]*Thing, error)
    CountByFilter(ctx context.Context, filter *ThingFilter) (uint64, error)

    // ...
}

// The model we will be using.
type Thing struct {
    TenantId       uint64      `db:"tenant_id" json:"tenant_id"`
    Id             uint64      `db:"id" json:"id"`
    ManufacturerId uint64      `db:"manufacturer_id" json:"manufacturer_id"`
    PartId         uint64      `db:"part_id" json:"part_id"`
    State          uint        `db:"state" json:"state"`
    Name           string      `db:"state" json:"name"`
}
```

### repositories/thing.go

```golang
// github.com/bartmika/hello-server/internal/repositories/thing.go
package repositories

import (
    "context"
    "database/sql"
    "time"
    "strconv"

    // null "gopkg.in/guregu/null.v4"

    "github.com/bartmika/hello-server/internal/models"
)

type ThingRepoImpl struct {
    db *sql.DB
}

func NewThingRepoImpl(db *sql.DB) *ThingRepoImpl {
    return &ThingRepoImpl{db: db}
}

func (s *ThingRepoImpl) ListByFilter(ctx context.Context, filter *models.ThingFilter) ([]*models.Thing, error) {
    // TODO
    return nil, nil
}

func (s *ThingRepoImpl) CountByFilter(ctx context.Context, filter *models.ThingFilter) (uint64, error) {
    // TODO
    return 0, nil
}
```

### Example Usage

#### Filter only one column

Here is how we lookup a single column:

```golang
// Assume you setup the context
// ctx := ...

// Assume you connected to the database
// db := ...

thingRepo := repositories.NewThingRepoImpl(db)

// Lookup all the `things` for a particular manufacturer.
thingFilter := models.ThingFilter{
    ManufacturerId: 1,
}
things, err := thingRepo.ListByFilter(ctx, thingFilter)
log.Println(things, err)
```

#### Filter more then one column

Here is how we lookup three columns:

```golang
// Assume you setup the context
// ctx := ...

// Assume you connected to the database
// db := ...

thingRepo := repositories.NewThingRepoImpl(db)

// Lookup all the `things` for a particular manufacturer and part
thingFilter := models.ThingFilter{
    ManufacturerId: 1,
    PartId: 123,
    State: 1,
}
things, err := thingRepo.ListByFilter(ctx, thingFilter)
log.Println(things, err)
```

Now that we've seen filtering code prototype and usage, let's discuss implementation of the ``TODO`` items in our repository.

# Understand the problem

If we look at our ``ThingFilter`` structure we see that the user has the option of using three optional filters and 3 required:

* TenantId -> Required
* ManufacturerId -> Optional
* PartId -> Optional
* State -> Optional
* LastSeenId -> Required
* Limit -> Required

Therefore our filtering code should handle these cases.

## First attempt (Static filtering)

My first attempt at solving the problem was to create the dedicated functions for whatever filtering we will have. Take a look at the ``repositories/thing.go`` file:

```golang
// github.com/bartmika/hello-server/internal/repositories/thing.go
// ...

func (s *ThingRepoImpl) ListAll(ctx context.Context) ([]*models.Thing, error) {
    // ...
}

func (s *ThingRepoImpl) ListByStateId(ctx context.Context, stateId uint64) ([]*models.Thing, error) {
    // ...
}

func (s *ThingRepoImpl) ListByManufacturerId(ctx context.Context, manufacturerId uint64) ([]*models.Thing, error) {
    // ...
}

func (s *ThingRepoImpl) ListByManufacturerIdAndState(ctx context.Context, manufacturerId uint64, stateId uint64) ([]*models.Thing, error) {
    // ...
}

func (s *ThingRepoImpl) ListByManufacturerIdAndPartId(ctx context.Context, manufacturerId uint64, partId uint64) ([]*models.Thing, error) {
    // ...
}

func (s *ThingRepoImpl) ListByManufacturerIdAndPartIdAndStateId(ctx context.Context, manufacturerId uint64, partId uint64, stateId uint64) ([]*models.Thing, error) {
    // ...
}

// Etc, etc
// ...
```

As you can see there are a number of problems to take into consideration:

* Every time you want to add a new column to filter, a lot more functions need to be created.
* It's not practical - The more filters you want to do, the more functions you have to write to maintain.

What pattern do you notice? This pattern does not work, an alternate solutions needs to exist.

## Second attempt (Too complicated)

My second attempt involved creating a unified function which encapsulates all the filtering capability.

Take a look at the ``repositories/thing.go`` file:

```golang
// github.com/bartmika/hello-server/internal/repositories/thing.go
// ...

func (s *ThingRepoImpl) queryRowsWithFilter(ctx context.Context, querySelect string, filter *models.ThingFilter) (*sql.Rows, error) {
    if filter.ManufacturerId.IsZero() {
        if filter.PartId.IsZero() {
            if filter.State.IsZero() {
                log.Println("Filtering by: Nothing")
                query := querySelect + `
                WHERE
                    tenant_id = $1
                AND
                    id > $2
                ORDER BY
                    id
                DESC LIMIT $3`

                return s.db.QueryContext(ctx, query, filter.TenantId, filter.LastSeenId, filter.Limit)
            } else {
                log.Println("Filtering by: Only State")
                query := querySelect + `
                WHERE
                    tenant_id = $1
                AND
                    id > $2
                AND
                    state = $3
                ORDER BY
                    id
                DESC LIMIT $4`

                return s.db.QueryContext(ctx, query, filter.TenantId, filter.State, filter.LastSeenId, filter.Limit)
            }
        } else {
            if filter.State.IsZero() {
                // Not writing code ...
            } else {
                // Not writing code ...
            }
        }
    } else {
        if filter.PartId.IsZero() {
            if filter.PartId.State() {
                // Not writing code ...
            } else {
                // Not writing code ...
            }
        } else {
            if filter.PartId.State() {
                // Not writing code ...
            } else {
                // Not writing code ...
            }
        }
    }
}


func (s *ThingRepoImpl) ListByFilter(ctx context.Context, filter *models.ThingFilter) ([]*models.Thing, error) {
    ctx, cancel := context.WithTimeout(ctx, 7*time.Second)
    defer cancel()

    querySelect := `
    SELECT
        tenant_id,
        id,
        manufacturer_id,
        part_id,
        state,
        name
    FROM
        things`

    rows, err := s.queryRowsWithFilter(ctx, querySelect, filter)
    if err != nil {
        return nil, err
    }

    var arr []*models.Thing
    defer rows.Close()
    for rows.Next() {
        m := new(models.Thing)
        err := rows.Scan(
            &m.TenantId,
            &m.Id,
            &m.ManufacturerId,
            &m.PartId,
            &m.State,
            &m.Name,
        )
        if err != nil {
            return nil, err
        }
        arr = append(arr, m)
    }
    err = rows.Err()
    return arr, err
}
```

As you can see there are a number of problems to take into consideration:

* Every time you want to add a new column to filter, the code get's super complicated quickly with repetition.
* It's complicated
* It's not practical - The more filters you want to do, the more complex it becomes

What pattern do you notice? The parameters we want to filter needs to change every time for the ``QueryContext`` function and we need to update the ``sql statement`` as well. The SQL statement is a string so we can easily modify the string, but how do we call the ``QueryContext`` function multiple times with different parameters?

## The "variadic function" to the rescue

In Golang, we have a [variadic function](https://golangbot.com/variadic-functions/) which can be used to load up different parameters into the ``QueryContext`` function.

# The Solution

The solution which works is to utilize [variadic functions](https://golangbot.com/variadic-functions/) and our code looks as follows.

#### repositories/thing.go

```golang
// github.com/bartmika/hello-server/internal/repositories/thing.go
package repositories

import (
    "context"
    "database/sql"
    "time"
    "strconv"

    "github.com/bartmika/hello-server/internal/models"
)

type ThingRepoImpl struct {
    db *sql.DB
}

func NewThingRepoImpl(db *sql.DB) *ThingRepoImpl {
    return &ThingRepoImpl{db: db}
}

func (s *ThingRepoImpl) queryRowsWithFilter(ctx context.Context, query string, filter *models.ThingFilter) (*sql.Rows, error) {
    // Array will hold all the unique values we want to add into the query.
    var filterValues []interface{}

    // The SQL query statement we will be calling in the database, start
    // by setting the `tenant_id` placeholder and then append our value to
    // the array.
    filterValues = append(filterValues, filter.TenantId)
    query += `WHERE tenant_id = $` + strconv.Itoa(len(filterValues))

    //
    // The following code will add our OPTIONAL filters
    //

    if !filter.ManufacturerId.IsZero() {
        filterValues = append(filterValues, filter.ManufacturerId)
        query += `AND manufacturer_id = $` + strconv.Itoa(len(filterValues))
    }
    if !filter.PartId.IsZero() {
        filterValues = append(filterValues, filter.PartId)
        query += `AND part_id = $` + strconv.Itoa(len(filterValues))
    }
    if !filter.State.IsZero() {
        filterValues = append(filterValues, filter.State)
        query += `AND state = $` + strconv.Itoa(len(filterValues))
    }

    //
    // The following code will add our REQUIRED filters
    // (Please note we use these for pagination purposes!)
    //

    if filter.LastSeenId > 0 {
        filterValues = append(filterValues, filter.LastSeenId)
        query += `AND id < $` + strconv.Itoa(len(filterValues))
    }
    query += `ORDER BY id `
    filterValues = append(filterValues, filter.Limit)
    query += `DESC LIMIT $` + strconv.Itoa(len(filterValues))


    //
    // Execute our custom built SQL query to the database.
    // (Notice our usage of the `variadic function`?)
    //

    return s.db.QueryContext(ctx, query, filterValues...)
}

func (s *ThingRepoImpl) ListByFilter(ctx context.Context, filter *models.ThingFilter) ([]*models.Thing, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    querySelect := `
    SELECT
        tenant_id,
        id,
        manufacturer_id,
        part_id,
        state,
        name
    FROM
        things`

    rows, err := s.queryRowsWithFilter(ctx, querySelect, filter)
    if err != nil {
        return nil, err
    }

    var arr []*models.Thing
    defer rows.Close()
    for rows.Next() {
        m := new(models.Thing)
        err := rows.Scan(
            &m.TenantId,
            &m.Id,			
            &m.ManufacturerId,
            &m.PartId,
            &m.State,
            &m.Name,
        )
        if err != nil {
            return nil, err
        }
        arr = append(arr, m)
    }
    err = rows.Err()
    return arr, err
}

func (s *ThingRepoImpl) CountByFilter(ctx context.Context, filter *models.ThingFilter) (uint64, error) {
    ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
    defer cancel()

    // The result we are looking for.
    var count uint64

    // Array will hold all the unique values we want to add into the query.
    var filterValues []interface{}

    // The SQL query statement we will be calling in the database, start
    // by setting the `tenant_id` placeholder and then append our value to
    // the array.
    filterValues = append(filterValues, filter.TenantId)
    query := `
    SELECT COUNT(id) FROM
        things
    WHERE
        tenant_id = $` + strconv.Itoa(len(filterValues))

    //
    // The following code will add our filters
    //

    if !filter.ManufacturerId.IsZero() {
        filterValues = append(filterValues, filter.ManufacturerId)
        query += `AND manufacturer_id = $` + strconv.Itoa(len(filterValues))
    }

    if !filter.PartId.IsZero() {
        filterValues = append(filterValues, filter.PartId)
        query += `AND part_id = $` + strconv.Itoa(len(filterValues))
    }

    if !filter.State.IsZero() {
        filterValues = append(filterValues, filter.State)
        query += `AND state = $` + strconv.Itoa(len(filterValues))
    }

    //
    // Execute our custom built SQL query to the database.
    //

    err := s.db.QueryRowContext(ctx, query, filterValues...).Scan(&count)
    return count, err
}
```

#  Extra Solutions

*2022-02-28 UPDATE: Recently I had someone email me the following question in regards to this article:*

> How do you apply that approach to check for other than "=" conditions?

*I think that's a great question! As a result, I'd like to expand this article to answer.*

*If you dear reader have any questions, don't hesitate to reach out and [contact me](/about/)!*

## How do I filter by Date/time?

Let's setup the following SQL structure. Let's assume the `metric_id` points to another table table with sensors (ex: UV Index), the implementation of that table doesn't matter. Here is our `time_series_data` table:

{{< highlight sql "linenos=false">}}
CREATE TABLE time_series_data (
    metric_id BIGINT NOT NULL,
    timestamp TIMESTAMPTZ NOT NULL,
    value DOUBLE PRECISION NULL,
    FOREIGN KEY (metric_id) REFERENCES metrics(id) ON DELETE CASCADE,
    PRIMARY KEY(timestamp, metric_id)
);
{{</ highlight >}}

If we want to lookup all records before *2022/02/28* date then we would write:

{{< highlight python "linenos=false">}}
import datetime

series = TimeSeriesData.objects.filter(timestamp__lte=datetime.date(2022, 02, 28))
{{</ highlight >}}

Furthermore if we want to find data within a range we can write the follow:

{{< highlight python "linenos=false">}}
from datetime import date, timedelta

startdate = date.today()
enddate = startdate + timedelta(days=6)
series = TimeSeriesData.objects.filter(timestamp__range=[startdate, enddate])
{{</ highlight >}}

Given the above cases, how do we support it? Let's apply what we learned so far with a few additions. In Django's ORM, if you want to filter, you have the following options:

* `lte` - Less then or equal to.
* `lt` - Less then.
* `gte` - Greater then or equal.
* `gt` - Greater then.

As a result, we'll create fields for each filter case so the code will look as follows:


### models/time_series_data.go

```go
// github.com/bartmika/hello-server/internal/models/time_series_data.go
package models

import (
    "context"
    "time"

    null "gopkg.in/guregu/null.v4"
)

type TimeSeriesDataFilter struct {
    // MetricID is used for filtering data from a particular sensor / instrument.
    MetricID                    uint64    `json:"metric_id"`

    // SortOrder is used for you to pick which ordering to use. Either use `asc` or `desc`.
    SortOrder                   string    `json:"sort_order"`

    // SortField is used for the specific field to order by.
    SortField                   string    `json:"sort_field"`

    Offset                      uint64    `json:"offset"`
    Limit                       uint64    `json:"limit"`

    // The following filters are to be used for specific date/time.

    TimestampGreaterThen        null.Time `json:"timestamp_gt,omitempty"`
    TimestampGreaterThenOrEqual null.Time `json:"timestamp_gte,omitempty"`
    TimestampLessThen           null.Time `json:"timestamp_lt,omitempty"`
    TimestampLessThenOrEqual    null.Time `json:"timestamp_lte,omitempty"`
}

// TimeSeriesDatum is a row in the `time_series_data` table.
type TimeSeriesDatum struct {
    MetricID  uint64     `json:"metric_id"`
    Timestamp time.Time  `json:"timestamp"`
    Value     null.Float `json:"value"`
}

type TimeSeriesDatumRepository interface {
    ListByFilter(ctx context.Context, filter *TimeSeriesDataFilter) ([]*TimeSeriesDatum, error)
}
```

### repositories/time_series_data.go

```go
// github.com/bartmika/hello-server/internal/repositories/time_series_data.go
package repositories

import (
    "context"
    "database/sql"
    "strconv"
    "time"

    "github.com/bartmika/hello-server/internal/models"
)

type TimeSeriesDatumRepo struct {
    db *sql.DB
}

func NewTimeSeriesDatumRepo(db *sql.DB) *TimeSeriesDatumRepo {
    return &TimeSeriesDatumRepo{
        db: db,
    }
}

func (s *TimeSeriesDatumRepo) listQueryRowsWithFilter(ctx context.Context, query string, f *models.TimeSeriesDataFilter) (*sql.Rows, error) {
    // Array will hold all the unique values we want to add into the query.
    var filterValues []interface{}

    // The SQL query statement we will be calling in the database, start
    // by setting the `tenant_id` placeholder and then append our value to
    // the array.
    filterValues = append(filterValues, f.MetricID)
    query += ` WHERE metric_id = $` + strconv.Itoa(len(filterValues))

    //
    // The following code will add our filters
    //

    if !f.TimestampGreaterThenOrEqual.IsZero() {
        filterValues = append(filterValues, f.TimestampGreaterThenOrEqual)
        query += ` AND timestamp >= $` + strconv.Itoa(len(filterValues))
    }
    if !f.TimestampGreaterThen.IsZero() {
        filterValues = append(filterValues, f.TimestampGreaterThen)
        query += ` AND timestamp > $` + strconv.Itoa(len(filterValues))
    }
    if !f.TimestampLessThenOrEqual.IsZero() {
        filterValues = append(filterValues, f.TimestampLessThenOrEqual)
        query += ` AND timestamp <= $` + strconv.Itoa(len(filterValues))
    }
    if !f.TimestampLessThen.IsZero() {
        filterValues = append(filterValues, f.TimestampLessThen)
        query += ` AND timestamp < $` + strconv.Itoa(len(filterValues))
    }

    //
    // The following code will add our pagination.
    //

    if f.Offset > 0 {
        // This step is necessary so please do not delete.
        f.Offset = f.Offset - 1
    }
    query += ` ORDER BY ` + f.SortField + ` ` + f.SortOrder
    filterValues = append(filterValues, f.Limit)
    query += ` LIMIT $` + strconv.Itoa(len(filterValues))
    filterValues = append(filterValues, f.Offset)
    query += ` OFFSET $` + strconv.Itoa(len(filterValues))

    //
    // Execute our custom built SQL query to the database.
    //

    // // For debugging purposes only.
    // log.Println("TimeSeriesDatumRepo | query:", query)
    // log.Println("TimeSeriesDatumRepo | filterValues:", filterValues)
    // log.Println("TimeSeriesDatumRepo | f:", f)

    return s.db.QueryContext(ctx, query, filterValues...)
}

func (s *TimeSeriesDatumRepo) ListByFilter(ctx context.Context, filter *models.TimeSeriesDataFilter) ([]*models.TimeSeriesDatum, error) {
	ctx, cancel := context.WithTimeout(ctx, 5*time.Second)
	defer cancel()

    querySelect := `
    SELECT
        metric_id,
        timestamp,
        value
    FROM
        time_series_data
    `

    rows, err := s.listQueryRowsWithFilter(ctx, querySelect, filter)
    if err != nil {
        return nil, err
    }

    var arr []*models.TimeSeriesDatum
    defer rows.Close()
    for rows.Next() {
        m := new(models.TimeSeriesDatum)
        err := rows.Scan(
            &m.MetricID,
            &m.Timestamp,
            &m.Value,
        )
        if err != nil {
            return nil, err
        }
        arr = append(arr, m)
    }
    err = rows.Err()
    if err != nil {
        return nil, err
    }

    if arr == nil {
        return []*models.TimeSeriesDatum{}, nil
    }
    return arr, err
}
```

### Example of Filtering with Dates
#### Filter by Less than or Equal to
The following example is how you would get all the data from the beginning to yesterdays date for a particular metric:

```go
// ...

var mid uint64 = 1 // Pretend its a UV Index meter.
var nowDT time.Time = time.Now()
var yesterdayDT time.Time = time.Date(nowDT.Year(), nowDT.Month(), nowDT.Day()-1, 0, 0, 0, 0, nowDT.Location())

f := models.TimeSeriesDataFilter{
    MetricID:  mid,
    SortField: "timestamp",
    SortOrder: "DESC",
    Offset:                      0,
    Limit:                       100,
    TimestampLessThenOrEqual:    yesterdayDT,
}

arr, err := TimeSeriesDatumRepo.ListByFilter(ctx, &f)
if err != nil {
    return err
}

log.Println(arr) // Return your data.
```

#### Filter by Range
The following example is how you would get all the data from yesterday to today (a.k.a. range):

```go
// ...

var mid uint64 = 1 // Pretend its a UV Index meter.
var nowDT time.Time = time.Now()
var yesterdayDT time.Time = time.Date(nowDT.Year(), nowDT.Month(), nowDT.Day()-1, 0, 0, 0, 0, nowDT.Location())

f := models.TimeSeriesDataFilter{
    MetricID:  mid,
    SortField: "timestamp",
    SortOrder: "DESC",
    Offset:                      0,
    Limit:                       100,
    TimestampGreaterThen:        yesterdayDT,
    TimestampLessThenOrEqual:    nowDT,
}

arr, err := TimeSeriesDatumRepo.ListByFilter(ctx, &f)
if err != nil {
    return err
}

log.Println(arr) // Return your data.
```
