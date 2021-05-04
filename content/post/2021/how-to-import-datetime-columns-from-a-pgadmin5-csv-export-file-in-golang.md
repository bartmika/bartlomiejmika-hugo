---
title: "How to Import Datetime Columns from a PgAdmin5 CSV export file in Golang"
date: 2021-03-10T14:34:05-05:00
draft: false
author: "Bartlomiej Mika"
categories:
- "Web Development"
tags:
- "Golang"
- "Postgres"
---

![](/img/2021/common/go-banner.png)

If you've exported from [``PgAdmin``](https://www.pgadmin.org/) a CSV file and tried to import that CSV into your Golang app, you may have noticed there is a challenge with importing the datatime. The purpose of this article contains the code I wrote to help me with this problem.

<!--more-->

# Problem

1. You exported a CSV file from [``PgAdmin``](https://www.pgadmin.org/).
2. You try to import your CSV file in a Golang app
3. Golang cannot convert to the datatime. Example: ``2018-06-17 21:23:59.241031-04``.

# Solution

I was unable to find the format that matched the available formats in the [std time](https://golang.org/pkg/time/) package.

{{< highlight go "linenos=false">}}
package utils

import (
    "strings"
    "time"
)

// Function will convert the timestamp string that `PgAdmin` exports (when you
// export as CSV format) into a Golang `time` data-format.
// TODO: Implement matching of timezones!
func ConvertPGAdminTimeStringToTime(dateTimeString string) (time.Time, error) {
    loc, err := time.LoadLocation("America/Toronto")
    if err != nil {
        return time.Now(), err
    }

    // Split our string so it will look like this:
    // INPUT: "2018-06-17 21:23:59.241031-04"
    // OUTPUT: ["2018-06-17", "21:23:59.241031-04"]
    dateTimeStringSplits := strings.Split(dateTimeString, " ")

    //
    // Convert the "date" format.
    //

    datePart, err := time.Parse("2006-01-02", dateTimeStringSplits[0])
    if err != nil {
        return time.Now(), err
    }

    //
    // Convert the "time" format.
    //

    // Split out our timezone format.
    // Ex: America/Toronto is "-04".
    // INPUT: "21:23:59.241031-04"
    // OUTPUT: ["21:23:59.241031", "-04"]
    timeSplits := strings.Split(dateTimeStringSplits[1], "-04")

    // The following code will take our "21:23:59.241031" string portion
    // and convert it into our Golang format. Please note you'll need to
    // read article [0] as to why I chose "15:04:05.000000" format.
    timePart, err := time.Parse("15:04:05.000000", timeSplits[0])
    if err != nil {
        return time.Now(), err
    }

    // Note; You'll need to see article [1] to see where I got these field.s
    dateTime := time.Date(datePart.Year(), datePart.Month(), datePart.Day(), timePart.Hour(), timePart.Minute(), timePart.Second(), timePart.Nanosecond(), loc)

    return dateTime, nil
}

// SPECIAL THANKS:
// [0] https://www.golangprograms.com/get-current-date-and-time-in-various-format-in-golang.html
// [1] https://golang.org/pkg/time/
{{</ highlight >}}

# How to use it

{{< highlight go "linenos=false">}}
package main

import (
    "fmt"
    "time"
    "<your-project-path>/utils"
)

// Example: "2018-06-17 21:23:59.241031-04"
func main() {
    timeStr := "2018-06-17 21:23:59.241031-04"

    timeVal, timeErr := utils.ConvertPGAdminTimeStringToTime(timeStr)
    if timeErr != nil {
        panic(timeErr)
    }
    fmt.Println(timeVal)
}
{{</ highlight >}}
