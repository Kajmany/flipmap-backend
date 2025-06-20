# What's All This, Then?

Elaborate API proxy for [**FlipMap, the Map App for Flip-phones!**](https://anticomputer.club/)

Relies on public external API's via OpenRouteService and Photon (hosted by Komoot).

Prevents us from having to ship those keys to untrustworthy client devices, and provides convenient input/output that simplifies the phone app.

## Requirements and Compiling

You may or may not need a suitable 'TLS backend' to build the project (TODO: check this), but you _will_ need one to run it. Most systems should already have this, but lightweight containers may require explicit installation. Ensure OpenSSL (often `libssl`) and the standard CA certificates for the distribution are installed. **Not having these will result in an early runtime panic upon attempting to construct the `ExternalRequester`.**

Ensure you have a functional Rust toolchain installed. See the [official installation guide](https://www.rust-lang.org/tools/install) for more.

Afterwards, Cargo takes the wheel: `cargo build`.

## Running

It is required to set an openrouteservice API key as an environmental variable: `ORS_API_KEY`. Not doing so will cause an early runtime panic, with a slightly more terse message telling you to do this.

Finally, running can be as simple as `<program> 127.0.0.1 1337` for testing on loopback with a very cool port. Use the `--help` flag in the application for up-to-date information on other options. It is possible to do other cool things like point the external API sources to arbitrary addresses.

## Endpoints

For now, all API endpoints are placed in `main.rs`. These take and return normal JSON, and not GeoJSON. This is intentional to simplify the app.

### /route

HTTP POST

Requires a starting lat/lon (most likely the users' current position) and a destination position. Returns an array of floats representing flattened coordinates of the form `[lat,lon,lat,lon...]` which are the waypoints forming the road-based route.

#### Input Dict Items

`src_lat: <number>` Additional Constraint: double-precision float where -90 <= n <= 90

`src_lon: <number>` Additional Constraint: double-precision float where -180 <= n <= 180

`dst_lat: <number>` Additional Constraint: double-precision float where -90 <= n <= 90

`dst_lon: <number>` Additional Constraint: double-precision float where -180 <= n <= 180

#### HTTP 200 Output Dict Items

`route: <array[number]>`

Where route is a flattened LineString of 2-element Positions, representing a contiguous array of waypoints for the route. 'LineString' and 'Position' are used as defined in [RFC 7946](https://datatracker.ietf.org/doc/html/rfc7946).

### /get_locations

HTTP POST

Queries a list of locations based upon search and starting lat/lon. Length of results may vary.

#### Input Dict Items

`lat: <number>` Additional Constraint: double-precision float where -90 <= n <= 90

`lon: <number>` Additional Constraint: double-precision float where -180 <= n <= 180

`query: <string>`

`amount: <number>` between 1 and 20

#### HTTP 200 Output Dict Items

`results: <array[lat: number, lon: number, name: string]>`

### Error for ALL Routes

HTTP 500:

`message: <string>`

Where message is an error that is purposely vague. See logs for more details!

HTTP 422:

`message: <string>`

Message is a more precise report of what's wrong with the user-provided input

HTTP 503:

`message: <string>` (body dict)

`RETRY_AFTER: <number>` (header)

Message is probably vague relating to how an external API being overtaxed makes the backend unavailable.
RETRY_AFTER is taken from the external API or generated by backend's own reckoning. Good faith estimate only.

## Troubleshooting

Tracing is enabled by default, but filters out some detail for brevity. Set the environment variables `RUST_BACKTRACE=1` and `RUST_LOG=trace` to maximize detail.

The error messages returned to the client will purposely not describe the specifics of internal failures. The error messages raised internally also may currently not log enough useful information. See the documentation `cargo doc --bins --document-private-items --open`
and refer to the `error.rs` enum `RouteError` for the most-up-to-date information on possible errors.

## Rate-Limiting

The application keeps internal state and timers in order to accurately rate-limit _external_ requests to APIs. It will heed well-formed `Retry-After` headers or set a static 'back-off' timer when they are not well-formed or when there is no header provided for an HTTP 429/503 request.

Because Komoot's Photon instance does not have a rate-limit, we also have an internal fixed-window implementation that hard-codes a by-minute and by-day limit on par with OpenRouteService's. This is subject to change.

## Deployment Consideration

This application expects to be able to make HTTPS requests to API endpoints. Errors will naturally result if firewalls or other configurations get in the way.

Neither TLS nor rate-limiting _for clients_ are implemented in the application. **It's strongly recommended to put the application behind a rate-limiting reverse proxy such as NGINX or Caddy.**

Containerization is supported as a first-class deployment method. Provided the TLS backend is present (see: Requirements), the effort to have full functionality should be minimal. Our production environment has multiple services on the reverse-proxy, so the files in this repository are not a _minimally viable_ example deployment, but they are quite simple. See `application.dockerfile`, `Caddyfile-backend`, and `compose.yaml` along with the Github deploy action for more.
