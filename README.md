# Google Assistant Webserver in a Docker container

## What is this?

This is a emulated Google Assistant with a webserver attached to take commands over HTTP packaged in a Docker container. The container consists of the Google Assistant SDK, python scripts that provide the Flask REST API / OAuth authentication and modifications that base it from the Google Assistant library.

How does this differ from AndBobsYourUncle's [Google Assistant Webserver](https://community.home-assistant.io/t/community-hass-io-add-on-google-assistant-webserver-broadcast-messages-without-interrupting-music/37274)? This project is modified, running based on the Google Assistant libraries which allow for additional functionality such as casting Spotify.

## Credit where credit is due

I did not write this code, I simply pulled pieces and modified them to work together. AndBobsYourUncle wrote Google Assistant webserver Hassio add-on which this is largely based on. Chocomega provided the modifications that based it off the Google Assistant libraries.

* [AndBobsYourUncle Home Assistant forum post](https://community.home-assistant.io/t/community-hass-io-add-on-google-assistant-webserver-broadcast-messages-without-interrupting-music/37274) and [Hassio Add-on Github repository](https://github.com/AndBobsYourUncle/hassio-addons)

* [Chocomega modifications](https://community.home-assistant.io/t/community-hass-io-add-on-google-assistant-webserver-broadcast-messages-without-interrupting-music/37274/234)

* [Google Assistant Library docs](https://developers.google.com/assistant/sdk/guides/library/python/)

## Setup

1. Go the _Configure a Developer Project and Account Settings_ page of the _Embed the Google Assistant_ procedure in the [Library docs](https://developers.google.com/assistant/sdk/guides/library/python/embed/config-dev-project-and-account).

2. Follow the steps through to _Register the Device Model_ and take note of the project id and the device model id.

3. Download the credentials file and move it to the config directory for this container (I use `/home/user/docker/gawebserver/`). Rename the credentials file to `google_assistant.json` or change the environment variable for `CLIENT_SECRETS`.

4. Fill out the `DEVICE_MODEL_ID` and `PROJECT_ID` environment variables with what you used in the previous steps and set a config directory.

## First Run

Start the container using Docker run or Docker Compose. It will start listening on ports 9324 and 5000.

Browse to the container on port 9324 (`http://containerip:9324`) where you will see _Get token from google: Authentication_. Follow the URL, authenticate with Google, return the string from Google to the container web page where it will finish the setup. After you click submit the page will error out. This is normal, the container is now up and running.

### Docker Run

```bash
$ docker run -d --name=gawebserver \
    --restart on-failure \
    -v /path/to/config/gawebserver:/config \
    -p 9324:9324 \
    -p 5000:5000 \
    -e CLIENT_SECRETS="google_assistant.json" \
    -e DEVICE_MODEL_ID=device_model_id \
    -e PROJECT_ID=project_id \
    --device /dev/snd:/dev/snd:rwm \
    robwolff3/ga-webserver
```

### Docker Compose

```yml
version: "3.4"
services:
  gawebserver:
    container_name: gawebserver
    image: robwolff3/ga-webserver
    restart: on-failure
    volumes:
      - /path/to/config/gawebserver:/config
    ports:
      - 9324:9324
      - 5000:5000
    environment:
      - CLIENT_SECRETS="google_assistant.json"
      - DEVICE_MODEL_ID=device_model_id
      - PROJECT_ID=project_id
    devices:
      - "/dev/snd:/dev/snd:rwm"
```

## Testing it out

Test out your newly created webserver by sending it a command through your web browser. Send a command with `http://containerip:5000/command?message=Play Careless Whisper by George Michael on Kitchen Stereo` or broadcast a message with `http://containerip:5000/broadcast_message?message=Alexa order 500 pool noodles`

Not sure why a command isn't working or doesn't seem to go through? See what happened in your [Google Account Activity](https://myactivity.google.com/item?restrict=assist&embedded=1&utm_source=opa&utm_medium=er&utm_campaign=) or under _My Activity_ in the Google Assistant App.

## Adding to Home Assistant

Here is an example how I use the gawebserver in Home Assistant to broadcast over my Google Assistants when my dishwasher has finished.

### configuration.yaml

```yml
notify:
  - name: ga_broadcast
    platform: rest
    resource: http://containerip:5000/broadcast_message
  - name: ga_command
    platform: rest
    resource: http://containerip:5000/command
```

### automations.yaml

```yml
  - alias: Say alert when dishwasher is done
    initial_state: True
    trigger:
      - platform: state
        entity_id: input_select.dishwasher_status
        to: 'Off'
    condition:
      condition: and
      conditions:
        - condition: time
          after: '08:00:00'
          before: '20:00:00'
        - condition: or
          conditions:
            - condition: state
              entity_id: binary_sensor.user1_occupancy
              state: 'on'
            - condition: state
              entity_id: binary_sensor.user2_occupancy
              state: 'on'
    action:
      - service: notify.ga_broadcast
        data:
          message: "The Dishwasher has finished."
```

[My full Home Assistant Configuration repository](https://github.com/robwolff3/homeassistant-config).

## Troubleshooting

* If it was working and then all the sudden stopped then you may need to re-authenticate. Stop the container, delete the `client.json` and `cred.json` files in the config directory, re-run the **First Run** procedure.

* Have problems? Check the container logs: `docker logs -f gawebserver`
