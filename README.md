# OsintCat for Go

The official SDK for the [OsintCat API](https://docs.osintcat.net). Typed requests and responses for every endpoint. Go 1.21+.

```sh
go get github.com/OsintCatHQ/osintcat-go
```

## API key

Create a key under [Account > Developer](https://www.osintcat.net/account/developer). A key is shown once; you choose its scopes and when it
expires. Keep it on the server: never ship it in a browser or mobile app.

The SDK reads the key from the `OSINTCAT_API_KEY` environment variable when you do not pass one, and sends it
in the `X-API-KEY` header.

## Usage

```go
package main

import (
	"context"
	"fmt"

	osintcat "github.com/OsintCatHQ/osintcat-go"
	osintcatclient "github.com/OsintCatHQ/osintcat-go/client"
)

func main() {
	client := osintcatclient.NewClient() // reads OSINTCAT_API_KEY; or option.WithAPIKey("cat_...")
	ctx := context.Background()

	account, err := client.Account.Get(ctx)
	if err != nil {
		panic(err)
	}
	fmt.Println(*account.AccountInfo.Plan)

	breach, err := client.Breach.Search(ctx, &osintcat.SearchBreachRequest{Query: "user@example.com"})
	if err != nil {
		panic(err)
	}
	fmt.Println(*breach.ResultsCount)

	// E-mail lookups state why they are run.
	_, err = client.Email.Lookup(ctx, &osintcat.LookupEmailRequest{Query: "user@example.com", Purpose: "fraud prevention"})
}
```

Fields that differ between records (breach records, for example) are kept as they come, in each type's
`GetExtraProperties()`.

## Methods

| Method | Endpoint | What it does |
| --- | --- | --- |
| `client.Account.Get()` | [`GET /api/user`](https://docs.osintcat.net/api-reference/endpoint/user) | Account |
| `client.Account.Modules()` | [`GET /api/modules`](https://docs.osintcat.net/api-reference/endpoint/modules) | Modules |
| `client.Breach.Search()` | [`GET /api/breach`](https://docs.osintcat.net/api-reference/endpoint/breach) | Breach Lookup |
| `client.Breach.DatabaseSearch()` | [`GET /api/database-search`](https://docs.osintcat.net/api-reference/endpoint/database-search) | Database Search |
| `client.Breach.Domain()` | [`GET /api/domain`](https://docs.osintcat.net/api-reference/endpoint/domain) | Domain Lookup |
| `client.Email.Lookup()` | [`GET /api/email-osint`](https://docs.osintcat.net/api-reference/endpoint/email-osint) | Email OSINT |
| `client.Phone.Lookup()` | [`GET /api/phone-osint`](https://docs.osintcat.net/api-reference/endpoint/phone-osint) | Phone OSINT |
| `client.IP.Lookup()` | [`GET /api/ip`](https://docs.osintcat.net/api-reference/endpoint/ip) | IP Lookup |
| `client.DNS.Resolve()` | [`GET /api/dns-resolver`](https://docs.osintcat.net/api-reference/endpoint/dns-resolver) | DNS Resolver |
| `client.Minecraft.Player()` | [`GET /api/minecraft`](https://docs.osintcat.net/api-reference/endpoint/minecraft) | Minecraft Player |
| `client.Minecraft.Leaks()` | [`GET /api/minecraft-lookup`](https://docs.osintcat.net/api-reference/endpoint/minecraft-osint) | Minecraft Leak Search |
| `client.Minecraft.Profile()` | [`GET /api/minecraft-lookup-v2`](https://docs.osintcat.net/api-reference/endpoint/minecraft-profile) | Minecraft Profile |
| `client.Steam.Profile()` | [`GET /api/steam-lookup`](https://docs.osintcat.net/api-reference/endpoint/steam) | Steam Profile |
| `client.Xbox.Profile()` | [`GET /api/xbox-lookup`](https://docs.osintcat.net/api-reference/endpoint/xbox) | Xbox Profile |
| `client.Twitch.Profile()` | [`GET /api/twitch`](https://docs.osintcat.net/api-reference/endpoint/twitch) | Twitch Profile |
| `client.Chess.Lookup()` | [`GET /api/chess-osint`](https://docs.osintcat.net/api-reference/endpoint/chess) | Chess.com Lookup |
| `client.Github.Profile()` | [`GET /api/github-lookup`](https://docs.osintcat.net/api-reference/endpoint/github) | GitHub Profile |
| `client.Reddit.Profile()` | [`GET /api/reddit`](https://docs.osintcat.net/api-reference/endpoint/reddit) | Reddit Profile |
| `client.X.Profile()` | [`GET /api/twitter-osint`](https://docs.osintcat.net/api-reference/endpoint/twitter) | X (Twitter) Profile |
| `client.Tiktok.ResolveShareLink()` | [`GET /api/tiktok-resolver`](https://docs.osintcat.net/api-reference/endpoint/tiktok-resolver) | TikTok Share Link |
| `client.Instagram.ResolveShareLink()` | [`GET /api/instagram-resolver`](https://docs.osintcat.net/api-reference/endpoint/instagram-resolver) | Instagram Share Link |
| `client.Vin.Query()` | [`GET /api/vin`](https://docs.osintcat.net/api-reference/endpoint/vin) | VIN Decoder |
| `client.Chile.Person()` | [`GET /api/chilean-name`](https://docs.osintcat.net/api-reference/endpoint/chilean-name) | Chilean Person Search |
| `client.Chile.Vehicle()` | [`GET /api/chilean-car`](https://docs.osintcat.net/api-reference/endpoint/chilean-car) | Chilean Vehicle Search |

## Errors

Any answer other than 2xx returns an error; each status has its own type (`*osintcat.TooManyRequestsError`,
`*osintcat.ForbiddenError`, ...), all wrapping `*core.APIError`.

```go
_, err := client.Breach.Search(ctx, &osintcat.SearchBreachRequest{Query: "user@example.com"})
var limited *osintcat.TooManyRequestsError
if errors.As(err, &limited) {
	// allowance used up, or slow down
}
```

| Status | Meaning |
| --- | --- |
| 400 | The request is not valid (a missing or wrong parameter). |
| 401 | No API key, or an unknown one. |
| 402 | Your balance does not cover the lookup. |
| 403 | The key was revoked or has expired, lacks the scope, is used from an address it is not allowed from, or your plan does not include the module. |
| 404 | Nothing was found. |
| 429 | The daily allowance is used up (`LIMIT_REACHED`, resets at 00:00 UTC), or too many requests in a short time. |
| 424 | The lookup could not be completed: a data source failed or was too slow (`X-Upstream-Status` says which). |

Every error body carries `error` and usually `message`; some add `error_id` (quote it to support) or `code`.

## Retries

The SDK does not retry on its own. A lookup that is retried after a timeout may already have been counted
or charged, and a `429 LIMIT_REACHED` cannot succeed before the allowance resets at 00:00 UTC. Retry
yourself where it is safe for you.

## Timeouts

Pass a context with a deadline, or an `*http.Client` with a timeout:

```go
ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
defer cancel()
client := osintcatclient.NewClient(option.WithHTTPClient(&http.Client{Timeout: 60 * time.Second}))
```

## Links

- [API documentation](https://docs.osintcat.net)
- [Create an API key](https://www.osintcat.net/account/developer)
- [Full method reference](./reference.md)

## License

MIT
