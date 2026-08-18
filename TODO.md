# Home Assistant MQTT discovery

* Add [MQTT discovery messages](https://www.home-assistant.io/integrations/mqtt/#discovery-messages) for each configured device.
* Subscribe to homeassistant/status and listen for payload "online" ("birth message", both configurable)
* Subscribe to homeassistant/+/config to check if other enocean-mqtt bridges already published discovery messages
* when birth message is received, wait for a random time, then send any discovery message not yet received w/ retain flag set

Check if possible:
* publish information on enocean-mqtt bridge w/ network MAC and its own availability

