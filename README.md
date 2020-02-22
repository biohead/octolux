# LuxPower Inverter / Octopus Time-of-use Tariff Integration

This is a Ruby script to parse [Octopus ToU tariff](https://octopus.energy/agile/) prices and control a [LuxPower ACS inverter](https://www.luxpowertek.com/ac-ess.html) according to rules you specify.

The particular use-case of this is to charge your home batteries when prices are cheap, and use that power at peak times.

There's also support for toggling Raspberry Pi GPIO pins as an added bonus.

## Installation

You'll need Ruby - at least 2.3 should be fine, which can be found in all good Linux distributions. Git is installed here too so you can clone the repository.

```
sudo apt-get install ruby ruby-bundler git
```

Clone this repository to your machine:

```
git clone https://github.com/celsworth/octolux.git
cd octolux
```

If you are running on a Raspberry Pi and want to use the GPIO support, you need to enable installing it with:

```
bundle config --local --delete without pi
```

Now install the gems. You may occasionally need to re-run this as I update the repository and bring in new dependencies or update existing ones.  This will install gems to `./vendor/bundle`, and so should not need root.

```
bundle install
```

Create a `config.ini` using the `doc/config.ini.example` as a template:

```
cp doc/config.ini.example config.ini
```

This script needs to know:

* where to find your Lux inverter, host and port.
* the serial numbers of your inverter and datalogger (the plug-in WiFi unit), which are normally printed on the sides.
* which Octopus tariff you're on, AGILE-18-02-21 is my current one for Octopus Agile.
* optionally on the Pi, a list of GPIOs you'll be controlling.

Copy `rules.rb` from the example as a starting point:

```
cp doc/rules.example.5p.rb rules.rb
```

This default one simply enables AC charging when the tariff price is 5p or lower, and disables it otherwise. Perhaps more exotic examples to follow.

The idea behind keeping the rules separate is you can edit it and be unaffected by any changes to the main script in the git repository (hopefully).

### Inverter Setup

By default, the datalogger plugged into the Lux sends statistics about your inverter to LuxPower in China. This is how their web portal and phone app knows all about you.

We need to configure it to open another port that we can talk to. Open a web browser to your datalogger IP (might have to check your DHCP server to find it) and login with username/password admin/admin. Click English in the top right :)

You should see:

![](doc/lux_run_state.png)

Tap on Network Setting in the menu. You should see two forms, the top one is populated with LuxPower's IP in China - the second one we can use. Configure it to look like the below and save:

![](doc/lux_network_setting.png)

After the datalogger reboots (this takes only a couple of seconds and does not affect the main inverter operation, it will continue as normal), port 4346 on your inverter IP is accessible to our Ruby script. You should be sure that this port is only accessible via your LAN, and not exposed to the Internet, or anyone can control your inverter.


## Usage

There are two components.

### server.rb

`server.rb` starts a HTTP server and is a long-running process that monitors the inverter for status packets (these include things like battery state-of-charge). We can then use this SOC in `octolux.rb`.

It's split like this because there's no way to ask the inverter for the current battery SOC. You just have to wait (up to two minutes) for it to tell you. The server will return the latest SOC on-demand via HTTP. If you're not interested in battery SOC you can ignore server.rb for now.

If you do want to run it, the simplest thing to do is just start it in screen:

```
screen
./server.rb
```

A systemd unit file will be added at some point so it starts automatically on boot.

### octolux.rb

`octolux.rb` is intended to be from cron, and enables or disables AC charge depending on the logic written in `rules.rb` (you'll need to copy/edit an example from docs/).

There's also a wrapper script, `octolux.sh`, which will divert output to a logfile (`octolux.log`), and also re-runs `octolux.rb` if it fails the first time (usually due to transient failures like the inverter not responding, which can occasionally happen). You'll want something like this in cron:

```
0,30 * * * * /home/pi/octolux/octolux.sh
```



## Development Notes

In your `rules.rb`, you have access to a few objects to do some heavy lifting.

*`octopus`* contains Octopus tariff price data. The most interesting method here is `price`:

  * `octopus.price` - the current tariff price, in pence
  * `octopus.prices` - a Hash of tariff prices, starting with the current price. Keys are the start time of the price, values are the prices in pence.

*`lc`* is a LuxController, which can do the following:

  * `lc.charge(true)` - enable AC charging
  * `lc.charge(false)` - disable AC charging
  * `lc.discharge(true)` - enable forced discharge
  * `lc.discharge(false)` - disable forced discharge
  * `lc.charge_pct` - get AC charge power rate, 0-100%
  * `lc.charge_pct = 50` - set AC charge power rate to 50%
  * `lc.discharge_pct` - get discharge power rate, 0-100%
  * `lc.discharge_pct = 50` - set discharge power rate to 50%

Forced discharge may be useful if you're paid for export and you have a surplus of stored power when the export rate is high.

Setting the power rates is probably a bit of a niche requirement. Note that discharge rate is *all* discharging, not just forced discharge. This can be used to cap the power being produced by the inverter. Setting it to 0 will disable discharging, even if not charging.

*`gpio`* is a GPIO controller (only available on the Raspberry Pi). These two methods take a string which corresponds to your configuration. See the example config under the `[gpios]` section.

  * `gpio.on('zappi')` - turn on the GPIO pin identified as *zappi* in your config.
  * `gpio.off('zappi')` - turn off the GPIO pin identified as *zappi* in your config.
  * `gpio.set('zappi', true)` - alternative way of turning on a GPIO. Pass `false` to turn it off.
