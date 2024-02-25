---
layout: single
excerpt_separator: <!--more-->
title: Temperature Logging with Dino and TempoDB
---

[Dino](http://github.com/austinbv/dino) is a Ruby gem that lets you control a connected Arduino with Ruby. It's very general purpose and already supports lots of hardware that you'd normally connect to an Arudino, including analog sensors, like the [TMP36 temperature sensor](http://learn.adafruit.com/tmp36-temperature-sensor) :

[![TMP36 Analog Temperature Sensor](/images/tmp36fritz.gif)](http://learn.adafruit.com/assets/476)

This script will read the value of the sensor connected to pin A0 of an attached Arduino and print the raw 10-bit reading from the ADC:

**Note: Some of these features are only available in the latest 0.12.0 branch of the dino gem.**

{% highlight ruby %}
require 'dino'
board = Dino::Board.new(Dino::TxRx::Serial.new)
tmp36 = Dino::Components::Sensor.new(pin: 'A0', board: board)

tmp36.read do |reading|
  puts reading
end
{% endhighlight %}

Wouldn't it be better to get the actual temperature though?

<!--more-->

## Calculating the Temperature

This [Adafruit](http://learn.adafruit.com/tmp36-temperature-sensor/using-a-temp-sensor) article shows how voltage varies with temperature for the TMP36. If we do some simple calculations in the block, we can make it print the temperature in ºC instead:

{% highlight ruby %}
tmp36.read do |reading|
  voltage = reading.to_f / 1024 * 5
  temp = (voltage - 0.5) * 100
  puts "#{temp} degrees Celsius"
end
{% endhighlight %}

Finding out the temperature is great, but what if we want to keep track of it over time? Storing values in a database is probably the way to go. Standard relational or document-oriented databases aren't well suited to this type of data though. Luckily, there's a new service called [TempoDB](https://tempo-db.com/) that makes it easy to store and work with time-series data like this.

## Logging with TempoDB

You can read more about how it works in their [documentation](https://tempo-db.com/docs/getting-started/), but the gist is that you create a series and fill it with timestamp/value data pairs. If we created a series called 'temp' and wrote the temperature every 10 seconds, the raw data might look something like this:

    2013-07-05T14:23:30.000, 32.0
    2013-07-05T14:23:40.000, 31.9
    2013-07-05T14:23:50.000, 31.7

Once you sign up (development is free) and create a database, access to the API is available via the `tempodb` [gem](https://github.com/tempodb/tempodb-ruby). Using it is straightforward. Connect with your database's key and secret, and use the included `TempoDB::DataPoint` class to construct data points before writing them to a series. Now we can modify the sensor script to write the temperature to TempoDB instead of printing it on screen:

{% highlight ruby %}
# Connect to TempoDB with the key and secret for your database.
require 'tempodb'
@client = TempoDB::Client.new('tempo-db-key', 'tempo-db-secret')

# Write a temperature and the current time to DB's 'temp' series.
def write(temp)
  data_point = TempoDB::DataPoint.new(Time.now.utc, temp)
  @client.write_key('temp', [data_point]) rescue nil
end

# Set up the Arduino and the sensor.
require 'dino'
board = Dino::Board.new(Dino::TxRx::Serial.new)
tmp36 = Dino::Components::Sensor.new(pin: 'A0', board: board)

# Poll the sensor at a 10 second interval instead of reading once.
tmp36.poll(10) do |reading|
  voltage = reading.to_f / 1024 * 5
  temp = (voltage - 0.5) * 100
  write(temp)
end

# Polling is in a separate thread. Sleep so our script doesnt die.
sleep
{% endhighlight %}

## Visualization

This is where one of TempoDB's best features comes in. To make sure it's working, log into the TempoDB web app, click on "View Data" for your database, choose the 'temp' series, and the web app will graph the data for you.

Refresh it and you should see new data points show up. Run it for a day and you'll end up with 8640 data points. This is where it gets even more useful. You can customize the time range, and run functions like `min`, `max`, and `mean` on custom intervals:


![TMP36 Analog Temperature Sensor](/images/tempo-db-graph.png)

The read API lets you get your data out and work with it, but I haven't used it much yet.


