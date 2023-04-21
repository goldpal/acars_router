# sdr-enthusiasts/acars_router

`acars_router` receives, validates, deduplicates, modifies and routes ACARS and VDLM2 JSON messages.

## Example Feeder `docker-compose.yml`

```yaml
version: "3.8"

volumes:
  - acarshub_run

services:
  acarsdec:
    image: ghcr.io/sdr-enthusiasts/docker-acarsdec:latest
    tty: true
    container_name: acarsdec
    restart: always
    devices:
      - /dev/bus/usb:/dev/bus/usb
    environment:
      - TZ=${FEEDER_TZ}
      - SERIAL=XXX
      - FREQUENCIES=XXX;XXX;XXX
      - GAIN=XXX
      - SERVER=acars_router
      - SERVER_PORT=5550
    tmpfs:
      - /run:exec,size=64M
      - /var/log

  dumpvdl2:
    image: ghcr.io/sdr-enthusiasts/docker-dumpvdl2:latest
    tty: true
    container_name: dumpvdl2
    restart: always
    devices:
      - /dev/bus/usb:/dev/bus/usb
    environment:
      - TZ=${FEEDER_TZ}
      - SERIAL=XXX
      - FREQUENCIES=XXX;XXX;XXX
      - GAIN=XXX
      - ZMQ_MODE=server
      - ZMQ_ENDPOINT=tcp://0.0.0.0:45555
    tmpfs:
      - /run:exec,size=64M
      - /var/log

  acars_router:
    image: ghcr.io/sdr-enthusiasts/acars_router:latest
    tty: true
    container_name: acars_router
    restart: always
    environment:
      - TZ=${FEEDER_TZ}
      - AR_SEND_UDP_ACARS=acarshub:5550
      - AR_SEND_UDP_VDLM2=acarshub:5555
      - AR_RECV_ZMQ_VDLM2=dumpvdl2:45555
      - AR_OVERRIDE_STATION_NAME=${FEEDER_NAME}
    tmpfs:
      - /run:exec,size=64M
      - /var/log

  acarshub:
    image: ghcr.io/sdr-enthusiasts/docker-acarshub:latest
    tty: true
    container_name: acarshub
    hostname: acarshub
    restart: always
    ports:
      - 8080:80
    environment:
      - TZ=${FEEDER_TZ}
      - ADSB_LAT=${FEEDER_LAT}
      - ADSB_LON=${FEEDER_LONG}
      - ENABLE_ADSB=true
      - ENABLE_ACARS=external
      - ENABLE_VDLM=external
    volumes:
      - acarshub_run:/run/acars
    tmpfs:
      - /run:exec,size=64M
      - /var/log
```

Change the `GAIN`, `FREQUENCIES`, `SERIAL`, `TZ`, `ADSB_LAT`, `ADSB_LON` and `AR_OVERRIDE_STATION_NAME` to suit your environment.

With the above deployment:

- JSON messages generated by `acarsdec` will be sent to `acars_router` via UDP.
- `acars_router` will connect to `dumpvdl2` via TCP/ZMQ and receive JSON messages.
- `acars_router` will set your station ID on all messages.
- `acars_router` will then output to `acarshub`.