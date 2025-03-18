---
layout: post
title: "Google Calendar Automation"
date: 2025-03-18 08:46
---

# Introduction
Why did I want to do this? My school has an app which contains calendar information.
But, the school's app does many things like checking your results,
your credits (used to purchase items at school), and other things.
But I mainly use the application to check my calendar. It's actually quite simple,
but it only took me weeks because I refused to search for stuff and guessed by way
through things.

# Scouting
While messing around in my school's student login, and messing around in the student page.
I noticed that the browser receives a JSON request literally called "classes". So, I was like
"nice." I quickly whipped up a Python script and a request to inspect the JSON. It was pretty well
structured.

```
{
  "MODULE_NAME": "Basic Finance",
  "DAY": "MON",
  "LOCATION": "XXX",
  "TIME_FROM_ISO": "2025-03-10T08:30:00+08:00",
  "TIME_TO_ISO": "2025-03-10T10:30:00+08:00",
  "GROUPING": "G1",
  "CLASS_CODE": "SAFI___AAQS006-3-C-BF-L-1___2025-01-08",
  "COLOR": "yellow"
}
```

This is heavily modified to preserve privacy. It was super good, and well structured.
So I searched for the google calendar API and was again like "Nice :)"

# Pulling data from the API
```py
def get_timetable():
    response = r.get(timetable_endpoint)
    timetables = response.json()

    my_classes = filter(lambda session: session["INTAKE"] == <INTAKE>, timetables)
    keys = ["MODULE_NAME", "TIME_FROM_ISO", "TIME_TO_ISO", "MODID", "ROOM"]
    my_classes = map(lambda class_: {key: class_[key] for key in keys}, my_classes)

    filtered_classes = []
    for details in my_classes:
        module_name = details['MODULE_NAME']
        title = f"{module_name} @ {details['ROOM']}"

        start = details["TIME_FROM_ISO"]
        # If the class date is less than now, ignore it.
        if not is_class_after_now(start):
            # Make sure events are in the future
            continue
        elif module_name == "Image Processing, Computer Vision and Pattern Recognition":
            # I'm not taking this
            continue

        description = f"Module code: {details['MODID']}"
        body = {
            "summary": title,
            "description": description,
            "start": {"dateTime": start},
            "end": {"dateTime": details["TIME_TO_ISO"]}
        }
        filtered_classes.append(body)

    return filtered_classes
```

This is pretty straightforward. I make a request then filter through the JSON and produce my own JSON.
I make sure the intake is mine because the JSON request has classes for *every* single intake. I also
had to make sure the class was in the future so there's just less stuff to filter.

# Getting data from google calendar

```py
def get_existing_classes(event, calendar_id, my_classes):
    class_names = [detail["summary"] for detail in my_classes]

    now = datetime.now(ZoneInfo(<location>))
    now = now.isoformat()

    all_events = event.list(
        calendarId=calendar_id["id"],
        timeMin=now,
        timeZone="UTC"
    ).execute()

    all_events = map(lambda events: events, all_events["items"])
    all_events = filter(lambda events: "summary" in events and "dateTime" in events["start"], all_events)
    all_events = {events["summary"]: events["start"]["dateTime"] for events in all_events if events["summary"] in class_names}

    return all_events
```

The part I struggled with was here.
- `calendarId` - Takes in an ID of your calendar and will pull events only from that calendar.

In this case, the calendar named "School" to only pull classes.

- `timeMin` - Takes all events startin from the specified time.

This is where I struggled because before I discovered `timeMin` was that I couldn't get all of my events
from the calendar I specified. This was because the total number of events collected is fixed. Setting `timeMin`
fixes the issue.

This is to get all classes, which helps to prevent duplicate rescheduling.

# Scheduling the events
```py
    unscheduled_classes = [my_class for my_class in my_classes if my_class["summary"] not in existing_classes]
    if len(unscheduled_classes) == 0:
        print("Nothing to schedule.")
    else:
        schedule_classes(event, unscheduled_classes)
```

Here the timetable from the school, plus the existing classes from the calendar. I just
make sure there are no duplicates by making sure that if the classes from the school timetable
is in the google calendar, I discard it. Otherwise, I store it in a list. Then I call `schedule_classes`.

```py
def schedule_classes(event, classes_to_schedule):
    calendar_address = <address>

    for details in my_classes:
        # Make sure it isn't scheduled already to avoid repeats
        event.insert(calendarId=calendar_address, body=details).execute()
```

Then I wait 30 seconds and then it appears in my calendar.

# Automating
```
0 20 * * 6 ~/Documents/Scripts/calendar.fish
```

I setup a cronjob and this is `calendar.fish`.

```
#!/bin/fish

cd /home/star/Documents/Projects
source .venv/bin/activate.fish
python3 main.py
```

So it will run at 8pm every friday.
