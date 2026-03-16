# Tailscale DERP Server

Docker images for running a Tailscale DERP server.

## Features

The DERP Server support two certmodes: `letsencrypt`, `manual`.

- `letsencrypt`:
  + Automatic TLS certificate via LetsEncrypt.
  + Requires valid domain name with DNS pointing to server.
- `manual`:
  + Self-signed certificates generated on startup.
  + Supports IP addresses as hostname (e.g., `127.0.0.1`).

## Usage

### LetsEncrypt Mode

#### Environment Vars

| Variable | Required | Default | Description |
| :---------- | :-------- | :-------- | :------------- |
| `DERP_HOSTNAME` | Yes | `derp-hostname.com` | Server hostname (must be valid domain) |
| `DERP_CERT_DIR` | No | `/app/certs` | Certificate directory |
| `DERP_ADDR` | No | `:443` | HTTPS listen address |
| `DERP_HTTP_PORT` | No | `80` | HTTP port |
| `DERP_STUN` | No | `true` | Enable STUN server |
| `DERP_STUN_PORT` | No | `3478` | STUN port |
| `DERP_VERIFY_CLIENTS` | No | `false` | Require client certificate verification |
| `DERP_VERIFY_CLIENT_URL` | No | `""` | URL to verify clients |

#### Run a DERP server by `docker compose`

```yaml
name: tailscale

services:
  derper:
    image: ghcr.io/misklinga/tailscale-derper:latest
    container_name: tailscale-derper
    restart: always
    ports:
      - "80:80"
      - "443:443"
      - "3478:3478/udp"
    environment:
      - DERP_HOSTNAME=derp-hostname.com
      - DERP_CERT_DIR=/app/certs
      - DERP_VERIFY_CLIENTS=true
    volumes:
      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock:ro
      - ./certs:/app/certs
```

#### Add the custom DERP servers to your tailnet

Each region has a unique region ID. The region ID values 900 to 999 are reserved for use as custom, user-specified regions and will not be used by Tailscale.

1. Go to the ACL edit Page - [`Tailscale Admin`](https://login.tailscale.com/admin/acls/file).
2. Add the following configuration to enable a custom DERP server with the hostname `derp-hostname.com`.

    ```json
    {
      // ... other parts of tailnet policy file
      "derpMap": {
        "OmitDefaultRegions": false,  // Keep official servers as backup
        "Regions": {
          "900": {
            "RegionID": 900,
            "RegionCode": "myderp",
            "RegionName": "My Custom Derper",
            "Nodes": [
              {
                "Name": "1",
                "RegionID": 900,
                "HostName": "derp-hostname.com",
                "DERPPort": 443,
                // IPv4 and IPv6 are optional, but recommended, to reduce
                // potential DERP connectivity issues if DNS is unavailable
                // or having issues. Addresses must be publicly routable
                // and not in private IP ranges.
                "IPv4": "203.0.113.15",
                "IPv6": "2001:db8::1"
              }
            ]
          }
        }
      }
    }
    ```

#### Verify the custom DERP server

1. Run `tailscale netcheck` to check DERP server connectivity:

   ```bash
   tailscale netcheck
   ```

   Look for your custom region (e.g., "myderp") in the DERP regions list.

2. Ping another client in your tailnet to verify traffic flows through your DERP server:

   ```bash
   tailscale ping <another-client>
   ```

   Use `tailscale status` to see which Tailscale node to ping.

### Manual Mode

#### Environment Vars

| Variable | Required | Default | Description |
| :---------- | :-------- | :-------- | :------------- |
| `DERP_HOSTNAME` | Yes | `127.0.0.1` | Server hostname or IP address |
| `DERP_CERT_DIR` | No | `/app/certs` | Certificate directory |
| `DERP_ADDR` | No | `:443` | Server listen address |
| `DERP_HTTP_PORT` | No | `80` | HTTP port |
| `DERP_STUN` | No | `true` | Enable STUN server |
| `DERP_STUN_PORT` | No | `3478` | STUN port |
| `DERP_VERIFY_CLIENTS` | No | `false` | Require client certificate verification |

#### Run a DERP server by `docker compose`

```yaml
name: tailscale

services:
  derper:
    image: ghcr.io/misklinga/tailscale-derper:latest-manual
    container_name: tailscale-derper
    restart: always
    ports:
      - "80:80"
      - "443:443"
      - "3478:3478/udp"
    environment:
      - DERP_VERIFY_CLIENTS=true
    volumes:
      - /var/run/tailscale/tailscaled.sock:/var/run/tailscale/tailscaled.sock:ro
```

#### Add the custom DERP servers to your tailnet

Each region has a unique region ID. The region ID values 900 to 999 are reserved for use as custom, user-specified regions and will not be used by Tailscale.

1. Go to the ACL edit Page - [`Tailscale Admin`](https://login.tailscale.com/admin/acls/file).
2. Add the following configuration to enable a custom DERP server.

    ```json
    {
      // ... other parts of tailnet policy file
      "derpMap": {
        "OmitDefaultRegions": false,  // Keep official servers as backup
        "Regions": {
          "900": {
            "RegionID": 900,
            "RegionCode": "myderp",
            "RegionName": "My Custom Derper",
            "Nodes": [
              {
                "Name": "1",
                "RegionID": 900,
                "DERPPort": 443,
                "IPv4": "203.0.113.15",
                "InsecureForTests": true,  // Because it is a self-signed certificate, the client does not do verification
              }
            ]
          }
        }
      }
    }
    ```

#### Verify the custom DERP server

1. Run `tailscale netcheck` to check DERP server connectivity:

   ```bash
   tailscale netcheck
   ```

   Look for your custom region (e.g., "myderp") in the DERP regions list.

2. Ping another client in your tailnet to verify traffic flows through your DERP server:

   ```bash
   tailscale ping <another-client>
   ```

   Use `tailscale status` to see which Tailscale node to ping.

## License

Distributed under the MIT License. See the [`LICENSE.txt`](LICENSE.txt) file for more information.

## Acknowledgments

- [Tailscale](https://tailscale.com)
- [Custom DERP servers](https://tailscale.com/kb/1118/custom-derp-servers/)
