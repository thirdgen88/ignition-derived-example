# Example Derived Image Solution for Ignition

This solution provides an example of how to implement a derived Ignition Docker image with the following features:

- Adding third-party modules such as MQTT Engine and Azure Injector.
- Bundling an integrated gateway backup.
- Setting default username and password of `default` built-in user source.
- Shimming the entrypoint script with a spot to introduce custom functionality to run prior to launching the gateway.

## How to Build

Before you get started, create a file called `gw-secrets/GATEWAY_ADMIN_PASSWORD` with the password that you want to associate with the `admin` user.  There is special functionality built into the "prep" stage of the multi-stage build that updates the `default` user source from the built-in gateway backup with your desired baseline password.  This is converted to a one-way salt+hash for storage in the gwbk.

There is a `docker-compose.yml` solution that can be used to build the image.  You can build+run the image with:

```
docker compose up --build -d
```

Note that the resultant gateway that will launch at http://localhost:8088 does not have a volume associated with it in order to assist with rapid testing of changes.

If you want to build the image with `docker build`, you can use something like the command below:

```
docker build \
    --build-arg IGNITION_VERSION=8.1.18 \
    --build-arg SUPPLEMENTAL_MODULES="azureiotinjector mqttengine" \
    --secret id=gateway-admin-password,src=gw-secrets/GATEWAY_ADMIN_PASSWORD \
    -t myimage:mytag \
    gw-build
```
