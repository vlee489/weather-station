Weather Station
==============

Data logging code for the Raspberry Pi Weather Station HAT

## Instructions to deploy

1. Start with a fresh install of Raspbian. Boot up as per usual and expand the filesystem to fill the SD card.
1. Un-blacklist the I²C module.

  `sudo nano /etc/modprobe.d/raspi-blacklist.conf`

  Comment out the line `blacklist i2c-bcm2708` by putting a hash `#` at the start of the line.
  
  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano.

1. Set the required modules to load automatically on boot.

  `sudo nano /etc/modules`
  
  Add the following lines to the bottom of the file:
  
  ```
  i2c-dev
  rtc_pcf8523
  w1-gpio
  w1-therm
  ```
  
  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano.

1. Reboot for these changes to take effect.

  `sudo reboot`

1. Ensure that a CR/BR1225 3 volt coin cell battery has been inserted. Positive `+` side facing up.
1. Enable the Real Time Clock (RTC). First drop into root:
  
  `sudo -s`
  
  Enable the RTC:

  `echo "pcf8523 0x68" > /sys/class/i2c-adapter/i2c-1/new_device`
  
  Check that that it now appears in `/dev`
  
  `ls /dev/rtc*`
  
  Expected result: `/dev/rtc0`
  
  Press `Ctrl - D` to come out of root.
  
1. Initialise the RTC with the correct time.

  Use the `date` command to check the current system time is correct. If correct then you can set the RTC time from the system clock with the following command:
  
  `sudo hwclock -w`
  
  If not then you can set the RTC time manually using the command below (you'll need to change the `--date` parameter, this example will set the date to the 1st of January 2014 at midnight):
  
  `sudo hwclock --set --date="2014-01-01 00:00:00" --utc`
  
  Then set the system clock from the RTC time.
  
  `sudo hwclock -s`
  
  You can passively display the time in the RTC using: `sudo hwclock -r`

1. Enable loading the RTC driver and setting the system clock at boot time.

  `sudo nano /etc/rc.local`
  
  Insert the following lines before `exit 0` at the bottom of the file:
  
  ```
  echo "pcf8523 0x68" > /sys/class/i2c-adapter/i2c-1/new_device
  sleep 2
  echo "Setting System clock from RTC..."
  hwclock -s
  hwclock -r
  ```
  
  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano.

1. Install the necessary software packages.

  ```
  sudo apt-get update
  sudo apt-get install i2c-tools python-smbus telnet apache2 mysql-server python-mysqldb php5 libapache2-mod-php5 php5-mysql -y
  ```
  
  This will take some time. You will be prompted to create and confirm a password for the root user of the MySQL database server.

1. Create the database within MySQL.

  `mysql -u root -p`
  
  Enter the password that you chose during installation.
  
  You'll now be at the MySQL prompt `mysql>`, first create the database:
  
  `CREATE DATABASE weather;`
  
  Expected result: `Query OK, 1 row affected (0.00 sec)`
  
  Switch to that database:
  
  `USE weather;`
  
  Expected result: `Database changed`
  
  Create the table that will store all of the weather measurements:
  
  ```
  CREATE TABLE WEATHER_MEASUREMENT(
    ID BIGINT NOT NULL AUTO_INCREMENT,
    REMOTE_ID BIGINT,
    AMBIENT_TEMPERATURE DECIMAL(6,2) NOT NULL,
    GROUND_TEMPERATURE DECIMAL(6,2) NOT NULL,
    AIR_QUALITY DECIMAL(6,2) NOT NULL,
    AIR_PRESSURE DECIMAL(6,2) NOT NULL,
    HUMIDITY DECIMAL(6,2) NOT NULL,
    WIND_DIRECTION DECIMAL(6,2) NULL,
    WIND_SPEED DECIMAL(6,2) NOT NULL,
    WIND_GUST_SPEED DECIMAL(6,2) NOT NULL,
    RAINFALL DECIMAL (6,2) NOT NULL,
    CREATED TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY ( ID )
  );
  ```
  
  Expected result: `Query OK, 0 rows affected (0.05 sec)`
  
  Press `Ctrl - D` or type `exit` to quit MySQL.

1. Remove the fake hardware clock package.

  ```
  sudo update-rc.d fake-hwclock remove
  sudo apt-get remove fake-hwclock -y
  ```

1. Test that the I²C devices are online and working.

  `sudo i2cdetect -y 1`
  
  Expected output:
  
  ```
       0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
  00:          -- -- -- -- -- -- -- -- -- -- -- -- --
  10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
  20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
  30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
  40: 40 -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
  50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
  60: -- -- -- -- -- -- -- -- UU 69 -- -- -- -- -- --
  70: -- -- -- -- -- -- -- 77
  ```
  
  - `40` = HTU21D. Humidity and temperature sensor.
  - `77` = BMP180. Barometric pressure sensor.
  - `68` = PCF8523. Real Time Clock, it will show as `UU` because it's reserved by the driver.
  - `69` = MCP3427. Analogue to Digital Converter.

  Note: `40` and `77` will only show if you have connected the **AIR** board to the main board.

1. Download the data logging code.

  ```
  cd ~
  git clone https://github.com/raspberrypi/weather-station.git
  ```
  
  This will create a new folder in the home directory called `weather-station`.
1. Start the Weather Station daemon and test it.

  `sudo ~/weather-station/interrupt_daemon.py start`
  
  Expected result: `PID: 2345` (your number will be different)
  
  A continually running process is required to monitor the rain gauge and the anemometer. These are reed switch sensors and the code uses interrupt detection. These interrupts can occur at any time as opposed to the timed measurements of the other sensors. You can use the *telnet* program to test or monitor it.
  
  `telnet localhost 49501`
  
  Expected result:
  
  ```
  Trying 127.0.0.1...
  Connected to localhost.
  Escape character is '^]'.
  OK
  ```
  
  The following text commands can be used:
  
  - `RAIN`: displays rainfall in ml
  - `WIND`: displays average wind speed in kph
  - `GUST`: displays wind gust speed in kph
  - `RESET`: resets the rain gauge and anemometer interrupt counts to zero
  - `BYE`: quits
  
  Use the `BYE` command to quit.

1. Set the Weather Station daemon to automatically start at boot time.

  `sudo nano /etc/rc.local`
  
  Insert the following lines before `exit 0` at the bottom of the file:
  
  ```
  echo "Starting Weather Station daemon..."
  /home/pi/weather-station/interrupt_daemon.py start
  ```
  
1. If you wish you can register your Weather Station with a cloud based **Oracle** database so that your data can be used by other schools.

  [Oracle Apex Database](https://apex.oracle.com/pls/apex/f?p=84942:LOGIN_DESKTOP:9427101834476:&tz=0:00)
  
  You will need to complete a form whereupon you must wait for a Raspberry Pi admin to approve your school in the database. Once approved an activation email will be sent to you containing a verification code. Log in using your **school name** for the username and the password that you chose. You will then be prompted for the verification code from the email.
  
  Many weather stations can belong to one school. Once you have logged in you'll need to create a new weather station under your school. The *latitude* and *longitude* of the weather station will be required for this. Once you have created a weather station it will have its own password automatically generated, this is used by the weather station itself when it uploads the measurements to Oracle and is separate to your school login.
  
  *Note:* There is a known bug here where the *Add Weather Station* screen does not show a `Create` button, but only a `Return` button on the right. If you experience this just log out and back in and that should fix it.
  
1. Add the weather station name and password to the local Oracle credentials file. This allows the code that uploads to Oracle to know what credentials to use.

  `cd ~/weather-station`
  
  `nano credentials.oracle.template`
  
  Replace the `name` and `key` parameters with the `Weather Station Name` and `Passcode` of the weather station as specified in Oracle (under *Home > Weather Stations*). The double quotes `"` enclosing these values in this file are important so take care not to remove them by mistake.
  
  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano.
  
1. Rename the Oracle credentials template file to enable it.

  `mv credentials.oracle.template credentials.oracle`

1. Update the to the local MySQL credentials file with the password for the MySQL *root* user that you chose during installation.

  `nano credentials.mysql`
  
  The PASSWORD field is probably the only one you need to change (unless you also chose `raspberry`). The double quotes `"` enclosing the values are important so take care not to remove them by mistake.
  
  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano.

1. The main entry points for the code are `log_all_sensors.py` and `upload_to_oracle.py`. These will be called by the [cron](http://en.wikipedia.org/wiki/Cron) scheduler to automatically take measurements. The measurements will be saved in the local MySQL database as well as uploaded to the Oracle Apex Database online (if you registered).

  The template crontab file `crontab.save` is provided as a default. If you wish to change the measurement or upload frequency then edit this file before going onto the next step:
  
  `nano crontab.save`
  
  Press `Ctrl - O` then `Enter` to save and `Ctrl - X` to quit nano when you're done.

1. Heads up! At this point you may wish to stop and check out the Weather Station scheme of work ([here](https://github.com/raspberrypilearning/weather-station-sow)).
1. Enable cron to automatically start taking measurements, also known as *data logging mode*. 

  `crontab < crontab.save`

  Your weather station is now live and recording data at timed intervals.
  
  You can disable data logging mode at any time by removing the crontab with the command below:
  
  `crontab -r`
  
  To enable data logging mode again use the command below:
  
  `crontab < ~/weather-station/crontab.save`
  
  *Note: Do not have data logging mode enabled while you're working through the data Collection lessons in the scheme of work.*
  
1. You can manually cause a measurement to be taken at any time with the following command:

  `sudo ~/weather-station/log_all_sensors.py`
  
  Don't worry if you see `Warning: Data truncated for column X at row 1`, this is expected.

1. You can manually trigger an upload too with the following command:

  `sudo ~/weather-station/upload_to_oracle.py`
  
1. You can also view the data in the database using the following commands:

  `mysql -u root -p`
  
  Enter the password. Then switch to the `weather` database:
  
  `USE weather;`
  
  Run a select query to return the contents of the `WEATHER_MEASUREMENT` table.
  
  `SELECT * FROM WEATHER_MEASUREMENT;`
  
  After a lot of measurements have been recorded it will be sensible to use the SQL *where* clause to only select records that were created after a specific date and time:
  
  `SELECT * FROM WEATHER_MEASUREMENT WHERE CREATED > '2014-01-01 12:00:00';`
  
  Press `Ctrl - D` or type `exit` to quit MySQL.

1. Go [here](https://github.com/raspberrypi/weather-station-www) to download and install the demo website.
