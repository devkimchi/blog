---
title: "Using Office Scripts to Get DateTimeOffset in Power Automate"
slug: getting-datetimeoffset-in-power-automate
description: "This post demonstrates how to use Office Scripts in Excel to retrieve a timezone offset and integrate it with a Power Automate workflow."
date: "2025-01-15"
author: Justin-Yoo
tags:
- power-automate
- excel
- office-scripts
- power-platform
cover: /2025/01/getting-datetimeoffset-in-power-automate-00.png
fullscreen: true
---

When creating a [Power Automate][pau] workflow, you often need to use the [date/time functions][pau functions] like `utcNow()`, `convertTimeZone()`, `convertFromUtc()` and `convertToUtc()`. Because of the nature of Power Automate, date/time calculation should always start from UTC and convert it to the designated timezone.

But here's a problem. The functions `convertTimeZone()`, `convertFromUtc()` and `convertToUtc()` don't keep timezone information in their results. After the conversion, if you convert this value to the [ISO-8601][iso8601] format, it becomes like `2025-01-15T12:34:56+00:00`. No matter how you change the timezone, it always comes with `+00:00`, which represents the UTC offset. The result may confuse users whether it's the correctly converted or not.

Therefore, to include the timezone offset in the date string format, you must calculate it separately.

Although there will be many ways to get this value, in this post, I'm going to show how to extract the timezone offset value using the [Office Scripts][m365 excel office scripts] feature in [Excel][m365 excel] within a Power Automate workflow.

## Extracting Timezone Offset Value with Office Scripts

Let's start with the Office Scripts function. Office Scripts is basically JavaScript, so if you are familiar with JavaScript, you can easily accommodate yourself. Here's the Office Scripts function that returns the timezone offset value based on the given `timestamp` and `timezone`.

```javascript
function main(
    workbook: ExcelScript.Workbook,
    // timestamp in 'yyyy-MM-dd' format
    timestamp: string,
    // timezone value in IANA format.
    // eg) Asia/Seoul, Australia/Melbourne, Australia/Brisbane, America/Los_Angeles, etc.
    timezone: string): string {

    // 1. Create a Date object with the given timestamp.
    //    At this stage, timezone offset is not guaranteed, even with 'Z'.
    const date = new Date(`${timestamp}T00:00:00Z`);

    // 2. Convert the timestamp to the timezone.
    const timestampInTZ = date.toLocaleString("en-US", { timeZone: timezone });
    const dateInTZ = new Date(timestampInTZ);

    // 3. Convert the timestamp to UTC.
    const timestampInUTC = date.toLocaleString("en-US", { timeZone: 'UTC' });
    const dateInUTC = new Date(timestampInUTC);

    // 4. Calculate the timezone difference.
    const diffInMS = dateInTZ.getTime() - dateInUTC.getTime();
    
    const sign = diffInMS >= 0 ? "+" : "-";
    const diffInABS = Math.abs(diffInMS);

    const totalHours = diffInABS / (1000 * 60 * 60);
    const hours = Math.floor(totalHours);
    const mins = Math.round((totalHours - hours) * 60);

    // 5. Format the timezone offset value.
    const hoursInFormat = String(hours).padStart(2, "0");
    const minsInFormat = String(mins).padStart(2, "0");

    return `${sign}${hoursInFormat}:${minsInFormat}`;
}
```

All you need is to make sure that you pass the `timestamp` value in the `yyyy-MM-dd` format and the `timezone` value in the [IANA format][iana tz] like `Asia/Seoul`, `Australia/Melbourne`, `America/Los_Angeles`, etc. With these parameters, it calculates the timezone offset value by comparing the converted timestamp with the UTC time.

Save the script as `getTimeZoneOffset` in an Excel file. Of course you can save it under a different name. Now, let's see how to call this script in a Power Automate workflow.

## Calling Office Scripts in Power Automate Workflow to Get TimeZone Offset Value

Add a new action in the Power Automate workflow and select the `Run Script` action.

![Select Run Script action][image-01]

Select the Excel file where the `getTimeZoneOffset` script is saved, provide the script name (`getTimeZoneOffset` in this case), and input the `timestamp` and `timezone` values. As mentioned earlier, the `timestamp` must follow the `yyyy-MM-dd` format, and the `timezone` must be in the IANA format (`Asia/Seoul` in this case).

![Enter Run Script action parameters][image-02]

Then, in the next action, use several conversion functions as shown in the figure below.

![Convert the date/time format][image-03]

1. Use `convertFromUtc()` and `utcNow()` functions to convert the current UTC time to local (or given timezone's) time.
1. The converted time doesn't contain the timezone offset information, so format the timestamp to `yyyy-MM-ddTHH:mm:ss.fff`.
1. Use the `concat()` function to combine the converted timestamp string with the timezone offset value calculated from the previous action.

When you run the Power Automate workflow, you're now able to see the timezone offset value properly attached as a string.

![Result with correct timezone offset value][image-04]

---

So far, we've walked through how to extract the timezone offset value using Office Scripts in a Power Automate workflow. With this script, you can easily get the timezone offset value in a Power Automate workflow.

By combining Office Scripts with Power Automate, you can handle complex timezone requirements without relying on external services or complicated logic.

I hope this solution adds more flexibility when building Power Automate workflows.

## More Resources

If you want to learn more about the topics covered in this post, please check out the following links:

- [Office Scripts in Excel][m365 excel office scripts]
- [Run Office Scripts with Power Automate][m365 excel office scripts pau]
- [Power Automate Date/Time functions][pau functions]
- [Power Automate Run Script connector][pau connectors excel run script]

[image-01]: /2025/01/getting-datetimeoffset-in-power-automate-01.png
[image-02]: /2025/01/getting-datetimeoffset-in-power-automate-02.png
[image-03]: /2025/01/getting-datetimeoffset-in-power-automate-03.png
[image-04]: /2025/01/getting-datetimeoffset-in-power-automate-04.png

[pau]: https://www.microsoft.com/power-platform/products/power-automate
[pau functions]: https://learn.microsoft.com/azure/logic-apps/workflow-definition-language-functions-reference#date-and-time-functions
[pau connectors excel run script]: https://learn.microsoft.com/connectors/excelonlinebusiness/#run-script

[m365 excel]: https://www.microsoft.com/microsoft-365/excel
[m365 excel office scripts]: https://learn.microsoft.com/office/dev/scripts/overview/excel
[m365 excel office scripts pau]: https://learn.microsoft.com/office/dev/scripts/develop/power-automate-integration

[iso8601]: https://ko.wikipedia.org/wiki/ISO_8601
[iana tz]: https://www.iana.org/time-zones
