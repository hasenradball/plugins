# OpenWeatherMap

This plugins can be used retrieve weather information from OpenWeatherMap (https://openweathermap.org/).

## Support
Support is provided trough the support thread within the smarthomeNG forum: [Smarthome.py](https://knx-user-forum.de/forum/supportforen/smarthome-py)


### Requirements

An api key from OpenWeatherMap is needed. It can be obtained free of charge from ```https://openweathermap.org```.

The api key which is available free of charge allows up to 60 calls a minute. Keep this in mind if you want to change the update frequency using the **`cycle`** parameter.

## Configuration

### plugin.yaml

Use the plugin configuration to configure the openweathermap plugin.

```yaml
# for etc/plugin.yaml configuration file:
weather_openweathermap:
    plugin_name: openweathermap
    key: xxxxyyyyxxxxyyyy
    instance: 'home'
    # latitude: '48.04712'
    # longitude: '11.81421'
    # altitude: '100'
    # lang: 'de'
    # cycle: 600
```

The API-key is configured via the **`key`** parameter.

You can configure multiple instances of the openweathermap plugin to collect data for multiple locations. If you don't need multiple locations / instances you may omit the **`instance`**-definition.

Positional information is optional and if it is omitted, the position-data from the **`etc/smarthome.yaml`** is considered.

The language for the responses is defined in the **`lang`** parameter.

The api key which is available free of charge allows up to 60 calls a minute. Keep this in mind if you want to change the update frequency using the **`cycle`** parameter. Keep in mind that this plugin uses multiple calls per update-cycle. It may sum up to 10 calls per cycle. If you use the same API-key in SmartVisu, this will as well consume your call-quota.

### items.yaml / Mapping of OWM-data to SHNG-items

The data provided by OpenWeatherMap is mapped to items by matchstrings. A matchstring is defining the path to a data-source within an API-response field. The available resources make use of the following APIs:

- [weather](https://openweathermap.org/current)
- [forecast](https://openweathermap.org/forecast5)
- ~~[uvi](https://openweathermap.org/api/uvi)~~ *deprecated*
- [onecall](https://openweathermap.org/api/one-call-api)
- onecall-0...onecall-4 uses the time-machine feature provided by the [one-call-api](https://openweathermap.org/api/one-call-api#history), to collect the "historic" data, which includes past values of today (onecall-0).
- [layer](https://openweathermap.org/api/weathermaps), providing map tile-data.
- [airpollution](https://openweathermap.org/api/air-pollution)

The list before is providing the names of the "data-source-keys" that are used within this plugin. You can see the results of the last calls within the web-interface of this plugin. The plugin identifies the need for downloading data from each endpoint by the definition of the respective matchstring. If no matchstring would require data from e.g. the data-source "weather", then this endpoint would not be called by the plugin. The prefferable way of reading weather-data is via the one-call API.

Data is retrieved in metric units (m, mm, hPa, °C).

In order to access the data that can be retrieved via the API an attribute of "owm_matchstring" is needed. Optionally you can add a "owm_match_prefix". That string is prepended to the "owm_matchstring" given.

``` yaml

forecast_daily1_no_prefix:
    type: str
    remark: This is a valid way of adressing the description of tomorrows weather
    owm_matchstring@home: day/1/weather/0/description

forecast_daily1_with_prefix:
    type: str
    remark: here the match-string is compiled as day/1/weather/0/description
    owm_match_prefix@home: day/1
    owm_matchstring@home: /weather/0/description
    
    temp_night:
        type: num
        remark: here the match-string is compiled as day/1/temp/night, inheriting the prefix from the parent-element.
        owm_match_prefix@home: ../.
        owm_matchstring@home: /temp/night
```

#### Soft-failing matching

The typical matching would access the root of the JSON-response of an API call and then follow the provided path to select the items from the response "tree". If the next node along the path cannot be matched an ERROR will be logged. Typically this is caused by typos or missing / wrong integer-indices on counted items.

Unfortunately not all responses from OWM will always contain a value. For example rain or snow data is only included if rain or snow has actually fallen or will be falling. For matchstrings ending in 'snow/3h', 'snow/1h', 'rain/3h' or 'rain/1h' an unmatched item will receive the value 0 instead of None. This will be logged as a DEBUG-level message (if enabled).

#### Matchstring-Rewriting / Data-Source identification

Matchstrings are re-written by the plugin to allow a clear distinction of the data-sources while maintaining readability. The start of a matchstring provides a hint on the utilized data-source:

- **begins with "virtual/"**, see [Virtual matchstrings](#virtual-matchstrings)
  example: `virtual/past24h/sum/rain/1h` for the total amount of rain in the past 24h.
- **begins with "forecast/daily/"**, see [Daily forecast (calculated)](#daily-forecast-calculated)
  example: `forecast/daily/0/main/temp_min` for tomorrows minimal temperature.
  consider using the one-call equivalents, e.g. `day/1/temp/min` to retrieve the same value as in the example
- **ends with "/eto" and begins with "current/" or "daily/"**, see [Evapotranspiration](#evapotranspiration)
  example: `daily/1/eto` for tomorrows ETO-value.
- **begins with "forecast/"** original data-source is the [forecast-API](https://openweathermap.org/forecast5#JSON):
  the prefix "forecast/" is replaced with "list/" when matching items in the JSON-source.
  example: `forecast/1/main/humidity` to retrieve the forecasted humidity three hours in the future.
- **begins with "uvi_"** original data-source is the [uvi-API](https://openweathermap.org/api/uvi#JSON):
  the prefix "uvi_" is removed when matching items in the JSON-source.
  example: `uvi_value` to get the current UV-index value
  as this API is deprecated, the replacement is `current/uvi`, it may be automatically replaced in future versions of this plugin.
- **begins with "current/"** original data-source is the [onecall-API](https://openweathermap.org/api/one-call-api), values are read directly.
  example: `current/weather/description` for a text describing the current weather in the defined language.
- **begins with "hour/I/"** where I is a number between 0 and 47 representing the relative hour from now onwards. Original data-source is the [onecall-API](https://openweathermap.org/api/one-call-api)
  the prefix "hour/" is replaced with "hourly/" when matching items in the JSON-source.
  example: `hour/2/feels_like` to get the perceived temperature two hours from now.
  complete set of data-points that can be retrieved for each hour:
    - `dt`: Point in time represented by this data-point
    - `temp`: Temperature in Celsius
    - `feels_like`: Perceived Temperature
    - `pressure`: Atmospheric pressure on the sea level, hPa
    - `humidity`: Relative Humidity in %
    - `dew_point`: Atmospheric temperature (varying according to pressure and humidity) below which water droplets begin to condense and dew can form. Celsius
    - `uvi`: UV index
    - `clouds`: Cloudiness %
    - `rain/1h`: Rain volume in mm
    - `snow/1h`: Snow volume in mm
    - `visibility`: Average visibility, metres
    - `wind_speed`: Wind speed in metre/sec (this can be extended as wind_speed/beaufort and wind_speed/description to get the Beaufort-value and the Description of the Speed-level in German or English)
    - `wind_deg`: Wind direction, degrees (meteorological)
    - `wind_gust`: Wind gust (peaks in speed) in metre/sec
    - `weather/0/id`: to get the weather condition id
    - `weather/0/main`: to get the group-name of weather parameters (Rain, Snow, Extreme etc.)
    - `weather/0/description`: to get the weather condition description within the group.
    - `weather/0/icon`: to get the weather icon id
    - `pop`: Propability of precipitation
- **begins with "day/N/"** where N is a number between 0 and 6. Be aware that -0 (see below) and 0 are returning different valid values! Original data-source is the [onecall-API](https://openweathermap.org/api/one-call-api).
  As you are using a positive value for N (including 0) outlook data is retrieved.
  the prefix "day/" is replaced with "daily/" when matching items in the JSON-source.
  example: `day/1/feels_like/night` to get tomorrows perceived temperature at night.
  complete set of data-points that can be retrieved for each day:
    - `dt`: Point in time represented by this data-point
    - `sunrise`: Sunrise of this day, UTC
    - `sunset`: Sunset of this day, UTC
    - `moonrise`: Moonrise of this day, UTC
    - `moonset`: Moonset of this day, UTC
    - `temp/morn`:  Morning temperature in Celsius.
    - `temp/day`:  Day temperature in Celsius.
    - `temp/eve`:  Evening temperature in Celsius.
    - `temp/night`:  Night temperature in Celsius.
    - `temp/min`:  Min daily temperature in Celsius.
    - `temp/max`:  Max daily temperature in Celsius.
    - `feels_like/morn`: Perceived Morning Temperature
    - `feels_like/day`: Perceived Day Temperature
    - `feels_like/eve`: Perceived Evening Temperature
    - `feels_like/night`: Perceived Night Temperature
    - `pressure`: Atmospheric pressure on the sea level, hPa
    - `humidity`: Relative Humidity in %
    - `dew_point`: Atmospheric temperature (varying according to pressure and humidity) below which water droplets begin to condense and dew can form. Celsius
    - `uvi`: Maximum UV index for the day
    - `clouds`: Cloudiness %
    - `rain`: Rain volume in mm
    - `snow`: Snow volume in mm
    - `pop`: Propability of precipitation
    - `visibility`: Average visibility, metres
    - `wind_speed`: Wind speed in metre/sec (this can be extended as wind_speed/beaufort and wind_speed/description to get the Beaufort-value and the Description of the Speed-level in German or English)
    - `wind_deg`: Wind direction, degrees (meteorological)
    - `wind_gust`: Wind gust (peaks in speed) in metre/sec
    - `weather/0/id`: to get the weather condition id
    - `weather/0/main`: to get the group-name of weather parameters (Rain, Snow, Extreme etc.)
    - `weather/0/description`: to get the weather condition description within the group.
    - `weather/0/icon`: to get the weather icon id
- **begins with "day/-N/"** where N is a number between 0 and 4. Be aware that -0 and 0 (see above) are returning different valid values! Original data-source is the [onecall-API with the time-machine feature](https://openweathermap.org/api/one-call-api#history).
  As you are using a negative value for N (including -0) historic data is retrieved. Appending an "hour/I/" to the matchstring results in selecting an hour "I" of that particular day. Warning: Accessing "day/-0/hour/18/..." at an earlier time than 6pm (UTC!!) will result in an ERROR as the API is not combining historic data with outlook data.
  Without appending hour, the daily summary will be retrieved (from the tree below "current/" within the JSON response).
  examples: 
    - `day/-1/hour/13/temp` to get yesterdays temperature at 1pm UTC.
    - `day/-2/pressure` to get the average(?) air-pressure from the day before yesterday.
- **begins with "airpollution"** Retrieves Air-Quality-Index and air-pollution component values. Original data-source is the [airpollution API](https://openweathermap.org/api/air-pollution).
    In general you can retrieve the following values:
    - `airpollution/main/aqi` to get the Air-Quality-Index
    - `airpollution/components/co`
    - `airpollution/components/no`
    - `airpollution/components/no2`
    - `airpollution/components/o3`
    - `airpollution/components/so2`
    - `airpollution/components/pm2_5`
    - `airpollution/components/pm10`
    - `airpollution/components/nh3`
    You may insert "/day/-1/hour/11/" between airpollution and main or component, where days can range from -1 to -4 and hour from 0 to 23. With that you can retrieve values of a certain hour from that day in the past.
    In order to retrieve forecast values you may insert "/hour/11" (not prepended with a day). This will provide access to the next 72 hours of forecast.
    Examples:
        - `airpollution/day/-1/hour/11/main/aqi` yesterday at 12:00 UTC
        - `airpollution/day/-4/hour/9/main/aqi` four days into the past at 9:00 UTC
        - `airpollution/hour/24/main/aqi` tommorrow, same time
- **ends with _new (see list below)** prepares a map-layer URL either from the given parameters owm_coord_x, owm_coord_y, owm_coord_z or from a translation of the current geo-coordinates to the tile-information
  Complete list of map-layers:
  - `clouds_new`
  - `precipitation_new`
  - `pressure_new`
  - `wind_new`
  - `temp_new`
- **everything else** is tried to be matched against the [weather-API](https://openweathermap.org/current).
  Complete list:
    - `base` / `cod` / `sys/id` / `sys/type` to get some internal parameters (if you can make sense of it).
    - `coord/lon` / `coord/lat` / `id` / `name` / `sys/country` / `timezone` for OWM's interpretation of your location data.
    - `clouds/all` / `visibility` to get the current cloud coverage and visibility range in meters.
    - `dt` / `sys/sunrise` / `sys/sunset` to get the request's time, sunrise and sunset time in UTC.
    - `main/temp` / `main/feels_like` / `main/temp_max` / `main/temp_min` to get current / today's temperature data.
    - `rain/1h` / `rain/3h` / `snow/1h` / `snow/3h` to get current precipitation data in mm
    - `main/humidity` / `main/pressure` to get current relative humidity (in %) and pressure values
    - `weather/0/id` to get the weather condition id
    - `weather/0/main` to get the group-name of weather parameters (Rain, Snow, Extreme etc.)
    - `weather/0/description` to get the weather condition description within the group.
    - `weather/0/icon` to get the weather icon id
    - `wind/deg` / `wind/speed` / `wind/gust` to get some facts about the wind (direction/speed/peak-speeds), (No Beaufort-suffixes possible here)

#### Matching lists

The weather-condition is stored as a list and can be correctly accessed via "current/weather/0/description". As the data-type list in "current/weather" is not obvious, the plugin will automatically insert a /0/ to match the first item of such list. Therefore "current/weather/description" will result in a value - and a WARNING in the log on every update. This feature is intended to find the issues easy and then permanently fix them by updating the matchstrings in your configuration.

In case you are working with dynamic lists such as "alerts" that might contain no to an undefined amount of data you can make use of the "@count"-directive in order to retrieve the number of records in that list. An example is "current/weather/@count" (always 1) or "alerts/@count", making a lot more sense.

#### Virtual matchstrings

Not all data can be directly retrieved via any API, some data needs to be aggregated via multiple data-sources. If you want to know the amount of rain of the past 24 hours at 10am you would need to query todays and yesterdays data and then summarize the data. This feature is built into the plugin. Virtual matchstrings are prefixed with the keyword "virtual".

```yaml

owm:
    rain_past_24h:
        type: num
        owm_matchstring@home: virtual/past24h/sum/rain/1h
    rain_next_24h:
        type: num
        owm_matchstring@home: virtual/next24h/sum/rain/1h
    avg_wind_next_24h:
        type: num
        owm_matchstring@home: virtual/next24h/avg/wind_speed
    max_wind_next_12h:
        type: num
        owm_matchstring@home: virtual/next12h/max/wind_gust

```

The virtual matchstrings consist of the following elements:

- prefix "virtual"
- a time-frame that could be:
    - past12h
    - next3d
    - ...
    The time-frame is compiled from a statement about the direction (past or next) a numeric literal and the unit, which could be "h" or "d" for hours or days. The maximum numbers are:
    - next6d
    - next48h
    - past4d
    - past96h
- an aggregation-function:
    - sum
    - max
    - min
    - avg
    - all (to generate a list with all items)
- a matchstring that would match an element in the [hourly one-call API](https://openweathermap.org/api/one-call-api#example). CAVE: For values as next#d the daily fields from the same API are matched!

An example usage of those virtual matchstrings is the rain_overview-widget for SmartVisu provided with this plugin:

```yaml
weather:
    as_of:
        type: num
        remark: This has to be a time-stamp to work properly, so no eval here
        owm_matchstring: current/dt
    rain_past_12h:
        type: list            
        owm_matchstring@home: virtual/past12h/all/rain/1h
    rain_next_12h:
        type: list
        owm_matchstring@home: virtual/next12h/all/rain/1h
```

```html
{% import "widgets_openweathermap.html" as owm %}
{{ owm.rain_overview('visual_id', 'weather.rain_past_12h', 'weather.rain_next_12h', 'weather.as_of') }}
```

#### Daily forecast (calculated)

Another type of virtual matchstrings are the values selected by a "forecast/daily/N/..."-matchstring. N represents a value between 0 and 4, where 0 represents tomorrow, 1 the day after tomorrow, etc.
Here the [forecast](https://openweathermap.org/forecast5#JSON)-data source is used. You may suffix "/min" or "/max" to the match-string in order to retrieve the respective aggregation. By default the average value is returned.

```yaml

owm:
    home:
        forecast_daily0:
            temp:
                type: num
                owm_matchstring@home: forecast/daily/0/main/temp

            temp_min:
                type: num
                owm_matchstring@home: forecast/daily/0/main/temp_min/min

            temp_max:
                type: num
                owm_matchstring@home: forecast/daily/0/main/temp_max/max
```

#### Evapotranspiration

The Evapotranspiration considers effects like wind, solar radiation (even indirect on cloudy days), pressure and relative humidity to calculate the loss of water from the ground by evaporation. The original data-source for the components considered is the [one-call API](https://openweathermap.org/api/one-call-api#example). The resulting value is a demand for irrigation in mm. This can be set in relation with the fallen rain to identify the real need.

Examples for matchstrings:
- `current/eto` / `daily/0/eto` get today's ETO
- `daily/1/eto`

More information can be retrieved at the original implementation found here: (https://github.com/MTry/homebridge-smart-irrigation)

The implementation of the calculation is based on: (https://edis.ifas.ufl.edu/pdffiles/ae/ae45900.pdf) and explained here: (http://www.fao.org/3/X0490E/x0490e00.htm#Contents)

Caveat: The formula used for ETO calculation makes use of a solar radiation feature. Unfortunately this value is not available for free via API. Luckily the UV-index matches the scale and should be somewhat equivalent to the actual value, so this is used in the calculation instead. Still: The usage of the UV-index instead of a real solar radiation feature is scientifically WRONG.

#### Struct to support irrigation control for daily irrigation (plants)

You can use the irrigation struct to switch an irrigation valve (solenoid) off automatically, based on the daily watering demand. If you combine that with an uzsu you will be able to even start the irrigation automatically. Using this method you will be able to water your plant based on the demand and not perform irrigation if there was enough rain.

```yaml
garden:
    gut_feeling_for_irrigation:
        type: num
        cache: yes
        remark: Value ranging from 0 to 2 where 1 would be normal, and 2 would double the amount
    irrigation_valve1:
        knx_dpt: 1
        knx_send: ...
        knx_cache: ...
        struct: 
            - owm.irrigation
            - uzsu.child  # in case you want to start automatically
        evaporation:
            exposure_factor:
                initial_value: 0.9  # Lightly shady area (greenhouses could be 0.7)
        rain:
            exposure_factor:
                initial_value: 0.5  # half covered by a roof (greenhouses would be 0)
        factors:
            flowrate_l_per_min:
                initial_value: 3.8  # liters per minute by irrigation system
            area_in_sqm:
                initial_value: 6  # area covered by irrigation system
            crop_coefficient:
                initial_value: 0.9  # depends on the type of crop, typically 0.3 to 0.9
            plant_density:
                initial_value: 1  # are your plants planted close (1.5) or wide apart (0.3), typically 0.3 to 1.5
            gut_feeling:
                eval: sum
                eval_trigger: 
                    - garden.gut_feeling_for_irrigation
```

The complete struct provides a hint how this is implemented:

```yaml

irrigation:
    type: bool
    autotimer: sh..schedule_seconds() = False
    visu_acl: rw
    enforce_updates: 'true'

    schedule_seconds:
        type: num
        initial_value: 0
        visu_acl: ro
        eval: round((sh...todays_water_demand_in_l() / sh...factors.flowrate_l_per_min()) * 60)
        eval_trigger:
            - ..factors.flowrate_l_per_min
            - ..todays_water_demand_in_l

        remaining_time:
            type: num
            visu_acl: ro
            enforce_updates: 'true'
            eval: sh...() - sh....age() if sh....() else 0
            eval_trigger: ...
            cycle: 1

    todays_water_demand_in_l:
        type: num
        eval: max(0, (sh...evaporation() * sh...evaporation.exposure_factor()) - (sh...rain() * sh...rain.exposure_factor())) * sh...factors()
        eval_trigger:
            - ..evaporation
            - ..evaporation.exposure_factor
            - ..rain
            - ..rain.exposure_factor
            - ..factors

    evaporation:
        type: num
        initial_value: 0
        owm_matchstring@instance: day/0/eto

        exposure_factor:
            remark: 'How exposed is your area to evaporation? Lower the factor for less exposure (e.g. shading, or wind-shields) or higher the factor if there is more sun (reflection) or wind (droughty areas).'
            type: num
            cache: yes
            initial_value: 1

    rain:
        type: num
        eval: sum
        eval_trigger:
            - .past_12h
            - .next_12h
        
        past_12h:
            type: num
            owm_matchstring@instance: virtual/past12h/sum/rain/1h            
        next_12h:
            type: num
            owm_matchstring@instance: virtual/next12h/sum/rain/1h

        exposure_factor:
            remark: 'How exposed is your area to rain? Lower the factor for less exposure (e.g. roofs or bushes) or higher the factor if additional water is put there (e.g. from roof-drains).'
            initial_value: 1
            type: num
            cache: yes

    factors:
        type: num
        eval: sh..area_in_sqm() * sh..crop_coefficient() * sh..plant_density() * sh..gut_feeling()
        eval_trigger:
            - .area_in_sqm
            - .crop_coefficient
            - .plant_density
            - .gut_feeling

        flowrate_l_per_min:
            remark: 'How much water is transported by your irrigation-system? liters per minute'
            initial_value: 4
            type: num
            cache: yes

        area_in_sqm:
            remark: 'This is the irrigated area. This is important for the effectivity of rain vs. evaporation.'
            initial_value: 1
            type: num
            cache: yes
        
        crop_coefficient:
            remark: 'This is the coefficient that can be set based on the plants. Typically 0.3 to 0.9'
            initial_value: 0.9
            type: num
            cache: yes
        
        plant_density:
            remark: 'How dense are the plants planted? Typically 0.3 to 1.5'
            initial_value: 1
            type: num
            cache: yes
        
        gut_feeling:
            remark: 'This is a factor that should be used to tweak irrigation based on gut-feelings, typically this should be assigned centrally for the whole yard (use eval).'
            initial_value: 1
            type: num
            cache: yes

```

This can be used from SmartVisu with a widget that is provided along with this plugin. Example, matching the YAML above:

```html

{% import "widgets_openweathermap.html" as owm %}
{{ owm.irrigation('valve_1', 'The greenhouse', 'garden.irrigation_valve1') }}

```

#### Struct to support irrigation control for weekly irrigation (lawn)

You can use the irrigation_weekly struct to switch an irrigation valve (solenoid) off automatically, based on the weekly watering demand. If you combine that with an uzsu you will be able to even start the irrigation automatically. Using this method you will be able to water your lawn based on the demand and not perform irrigation if there was enough rain.

In this case the past 4 days are considered and the outlook of the next 3 days.

```yaml
garden:
    gut_feeling_for_irrigation:
        type: num
        cache: yes
        remark: Value ranging from 0 to 2 where 1 would be normal, and 2 would double the amount
    irrigation_valve2:
        knx_dpt: 1
        knx_send: ...
        knx_cache: ...
        struct: 
            - owm.irrigation_weekly
            - uzsu.child  # in case you want to start automatically
        evaporation:
            exposure_factor:
                initial_value: 0.9  # Lightly shady area (greenhouses could be 0.7)
        rain:
            exposure_factor:
                initial_value: 0.5  # half covered by a roof (greenhouses would be 0)
        factors:
            flowrate_l_per_min:
                initial_value: 20   # liters per minute by irrigation system
            area_in_sqm:
                initial_value: 350  # area covered by irrigation system
            gut_feeling:
                eval: sum
                eval_trigger: 
                    - garden.gut_feeling_for_irrigation
```

The complete struct provides a hint how this is implemented:

```yaml

    irrigation_weekly:
        type: bool
        autotimer: sh..schedule_seconds() = False
        visu_acl: rw
        enforce_updates: 'true'

        schedule_seconds:
            type: num
            initial_value: 0
            visu_acl: ro
            eval: round((sh...weeks_water_demand_in_l() / sh...factors.flowrate_l_per_min()) * 60)
            eval_trigger:
                - ..factors.flowrate_l_per_min
                - ..weeks_water_demand_in_l

            remaining_time:
                type: num
                visu_acl: ro
                enforce_updates: 'true'
                eval: sh...() - sh....age() if sh....() else 0
                eval_trigger: ...
                cycle: 5

        weeks_water_demand_in_l:
            type: num
            eval: max(0, (sh...evaporation() * sh...evaporation.exposure_factor()) - (sh...rain() * sh...rain.exposure_factor())) * sh...factors()
            eval_trigger:
                - ..evaporation
                - ..evaporation.exposure_factor
                - ..rain
                - ..rain.exposure_factor
                - ..factors

        evaporation:
            type: num
            initial_value: 0
            eval: sum
            eval_trigger:
                - .day_past3
                - .day_past2
                - .day_past1
                - .day_past0
                - .day_next1
                - .day_next2
            day_past3:
                type: num
                owm_matchstring@instance: day/-3/eto
            day_past2:
                type: num
                owm_matchstring@instance: day/-2/eto
            day_past1:
                type: num
                owm_matchstring@instance: day/-1/eto
            day_past0:
                type: num
                owm_matchstring@instance: day/-0/eto
            day_next0:
                type: num
                owm_matchstring@instance: day/0/eto
            day_next1:
                type: num
                owm_matchstring@instance: day/1/eto
            day_next2:
                type: num
                owm_matchstring@instance: day/2/eto

            exposure_factor:
                remark: 'How exposed is your area to evaporation? Lower the factor for less exposure (e.g. shading, or wind-shields) or higher the factor if there is more sun (reflection) or wind (droughty areas).'
                type: num
                cache: yes
                initial_value: 1

        rain:
            type: num
            eval: sum
            eval_trigger:
                - .past_4d
                - .next_3d
            
            past_4d:
                type: num
                owm_matchstring@instance: virtual/past4d/sum/rain/1h            
            next_3d:
                type: num
                owm_matchstring@instance: virtual/next3d/sum/rain

            exposure_factor:
                remark: 'How exposed is your area to rain? Lower the factor for less exposure (e.g. roofs or bushes) or higher the factor if additional water is put there (e.g. from roof-drains).'
                initial_value: 1
                type: num
                cache: yes

        factors:
            type: num
            eval: sh..area_in_sqm() * sh..gut_feeling()
            eval_trigger:
                - .area_in_sqm
                - .gut_feeling

            flowrate_l_per_min:
                remark: 'How much water is transported by your irrigation-system? liters per minute'
                initial_value: 4
                type: num
                cache: yes

            area_in_sqm:
                remark: 'This is the irrigated area. This is important for the effectivity of rain vs. evaporation.'
                initial_value: 1
                type: num
                cache: yes
            
            gut_feeling:
                remark: 'This is a factor that should be used to tweak irrigation based on gut-feelings, typically this should be assigned centrally for the whole yard (use eval).'
                initial_value: 1
                type: num
                cache: yes

```

This can be used from SmartVisu with a widget that is provided along with this plugin. Example, matching the YAML above:

```html

{% import "widgets_openweathermap.html" as owm %}
{{ owm.irrigation_weekly('valve_2', 'Lawn in the backyard', 'garden.irrigation_valve2') }}

```



#### Weather alerts

Weather alerts are forwarded from the respective authority, e.g. the "Deutscher Wetterdienst". If there is an alert, typically two items are added to the list, one in the national language and another one in English. The plugin is ensuring there is always at least one alarm. If there is no alarm condition the plugin is adding a "Placebo"-Alert that is described as "No Alert". This is done to ensure that the matchstring "alerts/0/event" will always return a value, otherwise not even the "alerts"-node is defined in the API-response, resulting in ERROR-messages in the log. By using "alerts/@count" one can identify whether there is an alert or not. If only the placebo-alert is defined, "alerts/@count" will return the numerical value "0", although there is an item in the list.

One way to display the alerts in the SmartVisu is the usage of the status.activelist-widget:

```html
{{ status.activelist('', 'weather.alerts', 'event', 'start', 'description', '') }} 
```


#### Caveats

* All times are in UTC. So if you query "yesterdays" values for Germany you will have a 1hr or 2hr time-frame from the next day and a missing time-frame of the same day.
* The formula used for ETO calculation makes use of a solar radiation feature. Unfortunately this value is not available for free via API. Luckily the UV-index matches the scale and should be somewhat equivalent to the actual value, so this is used in the calculation instead. Still: The usage of the UV-index instead of a real solar radiation feature is scientifically WRONG.
* For an unknown reason ([Thanks for discovering Sisamiwe](https://knx-user-forum.de/forum/supportforen/smarthome-py/1246998-support-thread-zum-openweathermap-plugin?p=1672747#post1672747)) "weather" is a list, so you have to use "weather/0/id" to get the id value.

#### Tips and Tricks

To convert the time in the dt values to a local value you may want to use an eval string and generate a printable value.

  ```yaml
  conditions_as_of:
        type: str
        owm_matchstring: day/1/dt 
        eval: datetime.datetime.fromtimestamp(value, datetime.timezone.utc).astimezone().strftime('%Y-%m-%d %H:%M:%S %Z%z')
  ```

#### Example: items.yaml
Example configuration of an item-tree for the openweathermap plugin in yaml-format:

https://gist.github.com/sisamiwe/cad440d4ba643324577fc8cbf8b0930b

Please keep in mind to replace the _priv_openweathermap with openweathermap, or how ever the plugin-path is named.
The plugin.yaml that is part of that gist doesn't need to be copied, it is already part of this plugin-version.

### logic.yaml

No logic configuration implemented.

## Methods / Functions

### get_beaufort_number(value_in_meter_per_second)

### get_beaufort_description(bft_number)
