# strava-postgres

This is a simple set of scripts that allow you to load all of your Strava activities into a Postgres database.

```sql
gomes=> select sport_type, count(*) from activities group by sport_type order by count desc;
   sport_type   | count
----------------+-------
 Run            |   711
 Ride           |   154
 VirtualRide    |   126
 Swim           |    91
 WeightTraining |    65
 Workout        |    30
 Walk           |    29
 AlpineSki      |    19
 Elliptical     |     8
 Hike           |     7
 Yoga           |     1
 Kayaking       |     1
 StairStepper   |     1
 Windsurf       |     1
 Tennis         |     1
 Rowing         |     1
 Surfing        |     1
(17 rows)

gomes=> \d activities
                             Table "public.activities"
            Column             |       Type       | Collation | Nullable | Default
-------------------------------+------------------+-----------+----------+---------
 resource_state                | double precision |           |          |
 athlete.id                    | double precision |           |          |
 athlete.resource_state        | double precision |           |          |
 name                          | text             |           |          |
 distance                      | double precision |           |          |
 moving_time                   | double precision |           |          |
 elapsed_time                  | double precision |           |          |
 total_elevation_gain          | double precision |           |          |
 type                          | text             |           |          |
 sport_type                    | text             |           |          |
 workout_type                  | double precision |           |          |
 id                            | double precision |           | not null |
 start_date                    | text             |           |          |
 start_date_local              | text             |           |          |
 timezone                      | text             |           |          |
 utc_offset                    | double precision |           |          |
 location_city                 | text             |           |          |
 location_state                | text             |           |          |
 location_country              | text             |           |          |
 achievement_count             | double precision |           |          |
 kudos_count                   | double precision |           |          |
 comment_count                 | double precision |           |          |
 athlete_count                 | double precision |           |          |
 photo_count                   | double precision |           |          |
 map.id                        | text             |           |          |
 map.summary_polyline          | text             |           |          |
 map.resource_state            | double precision |           |          |
 trainer                       | boolean          |           |          |
 commute                       | boolean          |           |          |
 manual                        | boolean          |           |          |
 private                       | boolean          |           |          |
 visibility                    | text             |           |          |
 flagged                       | boolean          |           |          |
 gear_id                       | text             |           |          |
 start_latlng                  | json             |           |          |
 end_latlng                    | json             |           |          |
 average_speed                 | double precision |           |          |
 max_speed                     | double precision |           |          |
 average_cadence               | double precision |           |          |
 average_temp                  | double precision |           |          |
 has_heartrate                 | boolean          |           |          |
 average_heartrate             | double precision |           |          |
 max_heartrate                 | double precision |           |          |
 heartrate_opt_out             | boolean          |           |          |
 display_hide_heartrate_option | boolean          |           |          |
 elev_high                     | double precision |           |          |
 elev_low                      | double precision |           |          |
 upload_id                     | double precision |           |          |
 upload_id_str                 | text             |           |          |
 external_id                   | text             |           |          |
 from_accepted_tag             | boolean          |           |          |
 pr_count                      | double precision |           |          |
 total_photo_count             | double precision |           |          |
 has_kudoed                    | boolean          |           |          |
Indexes:
    "activities_pkey" PRIMARY KEY, btree (id)
```

## How to use it?

1. Register a Strava app through https://www.strava.com/settings/api.
2. Take note of the Client ID and Client Secret.
3. Clone this repository.
4. Install dependencies with `bun install`.
5. Set up the `STRAVA_CLIENT_ID` and `STRAVA_CLIENT_SECRET` environment variables (e.g., using a `.envrc` file).
6. Run `bun get-access-token.ts` and follow the instructions. Make sure to copy the access token this script outputs at the end:

```bash
$ bun get-access-token.ts
Go to URL https://www.strava.com/oauth/authorize?client_id=123456&redirect_uri=http%3A%2F%2Flocalhost%2Fexchange_token&response_type=code&scope=activity%3Awrite%2Cactivity%3Aread%2Cactivity%3Aread_all and authorize application
Once you have authorized, you will be redirected to a 'localhost' address (don't worry if you see a 'This site can’t be reached' message)
✔ Copy the whole URL of the page from the browser and paste it here :  … http://localhost/exchange_token?state=&code=3ef024236b8c48891a23d318b9256fdf571210e8&scope=read,activity:write,activity:read,activity:read_all
Successfully retrieved access token: f6a7d88c02a3429fb62cc9f97fc54fb1cc868912
```

7. Spin up a Postgres instance and apply the `./schema.sql` file in this repository. This file does **not** create a database.
8. Set up a `PG_CONNECTION_STRING` environment variable that looks like `postgres://[user]:[password]@[hostname][:port]/[dbname]`
9. Run `bun load-activities.ts <access-token>`

(If you've made edits to existing activities and want to force them to get updated, you can run `bun load-activities.ts --update-existing <access-token>`)

10. Connect to your Postgres instance, and run `select * from <database_name>.activities`.

## How long does it take?

I have ~1250 activities on my Strava profile. This is how long it takes to update activities:

```bash
$ time bun load-activities.ts <access-token>
Page 1 (200 activities found)
Page 2 (200 activities found)
Page 3 (200 activities found)
Page 4 (200 activities found)
Page 5 (200 activities found)
Page 6 (200 activities found)
Page 7 (47 activities found)
bun load-activities.ts 518ccfd5859027a07577302ee8729a7fde462517  146.06s user 54.10s system 84% cpu 3:57.22 total
```

That's ~4 minutes. And this is how long it will take to load activities for **the first time**:

```bash
$ time bun load-activities.ts <access-token>
Page 1 (200 activities found)
[Activity 1] Inserting brand new activity.
[Activity 2] Inserting brand new activity.
[Activity 3] Inserting brand new activity.
[Activity 4] Inserting brand new activity.
[Activity 5] Inserting brand new activity.
...
Page 7 (47 activities found)
...
[Activity 1246] Inserting brand new activity.
[Activity 1247] Inserting brand new activity.
bun load-activities.ts 518ccfd5859027a07577302ee8729a7fde462517  263.20s user 94.43s system 90% cpu 6:33.19 total
```

That's 6min30secs.

## TODO

* This script could be made much faster by accepting a "Last Sync At" parameter that makes it search the Strava API for activities that have only occurred since a certain date.
* We could merge both the `get-access-token.ts` and the `load-activities.ts` scripts for ease of use.
* This project could be further automated by having a script that automatically starts Postgres in a local Docker container and loads data there.
* More data besides activities could be collected.
* We should have more examples in this README of what could be done with Strava data in Postgres
* This script could allow custom sleeps so it doesn't reach Strava's API limits for users with more data.