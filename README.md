[![GoDoc](https://pkg.go.dev/badge/github.com/rsjethani/rsling.svg)](https://pkg.go.dev/github.com/rsjethani/rsling)
[![Build Status](https://github.com/rsjethani/rsling/actions/workflows/checks.yml/badge.svg?branch=main)](https://github.com/rsjethani/rsling/actions)
[![Go Report Card](https://goreportcard.com/badge/github.com/rsjethani/rsling)](https://goreportcard.com/report/github.com/rsjethani/rsling)


# Overview
rsling is a Go HTTP client library for creating and sending API requests.

rslings store HTTP Request properties to simplify sending requests and decoding responses. Check [usage](#usage) or the [examples](examples) to learn how to compose a rsling into your API client.

**Fair Notice:** rsling started its life as a normal fork of the already popular and awesome
[sling](https://github.com/dghubble/sling). However overtime I noticed for some reason PRs like
[this](https://github.com/dghubble/sling/pull/73) and [this](https://github.com/dghubble/sling/pull/85) etc. were not 
being entertained at all for unknown reasons hence I decided to make this repo into a hard/detached fork and go my 
separate way so that exciting new features and improvements can be added without hinderance. Having said that I would 
like to give a big thanks to the contributors and owners of the sling repository because without them this package 
could not have existed. 

**Differences from sling:**
- Easier to perform context aware requests, [see](https://github.com/rsjethani/rsling/commit/bd74a7a823f4e5f89b56115e3a91a301e74ce4ec)
- Ability to provide separate response decoders for success and failure cases, [see](https://github.com/rsjethani/rsling/commit/b04385a7c3839ccd651f7d58eda13326b3d3333b)
- Ability to pass on response data as it is for data streaming purposes or deferred decoding, [see](https://github.com/rsjethani/rsling/commit/b285c20f7ca3aef7ba7c7c1c5a0f5e3b837c2996)
- Much more intuitive way of extending URL paths using the `Extend` API, [see](https://github.com/rsjethani/rsling/commit/7a1ff5d2a4e5eaef8895eafad9ed074293abe14a)

### Features
* Method Setters: Get/Post/Put/Patch/Delete/Head
* Add or Set Request Headers
* Base/Path: Extend a rsling for different endpoints
* Encode structs into URL query parameters
* Encode a form or JSON into the Request Body
* Receive JSON success or failure responses
* Control a request's lifetime via context

## Install
```
go get github.com/rsjethani/rsling
```

## Documentation
Read [GoDoc](https://pkg.go.dev/badge/github.com/rsjethani/rsling)

## Usage
Use a rsling to set path, method, header, query, or body properties and create an `http.Request`.

```go
type Params struct {
    Count int `url:"count,omitempty"`
}
params := &Params{Count: 5}

req, err := rsling.New().Get("https://example.com").QueryStruct(params).Request()
client.Do(req)
```

### Path
Use `Path` to set or extend the URL for created Requests. Extension means the path will be resolved relative to the existing URL.

```go
// creates a GET request to https://example.com/foo/bar
req, err := rsling.New().Base("https://example.com/").Path("foo/").Path("bar").Request()
```

Use `Get`, `Post`, `Put`, `Patch`, `Delete`, `Head`, `Options`, `Trace`, or `Connect` which are exactly the same as `Path` except they set the HTTP method too.

```go
req, err := rsling.New().Post("http://upload.com/gophers")
```

### Headers
`Add` or `Set` headers for requests created by a rsling.

```go
s := rsling.New().Base(baseUrl).Set("User-Agent", "Gophergram API Client")
req, err := s.New().Get("gophergram/list").Request()
```

### Query

#### QueryStruct
Define [url tagged structs](https://godoc.org/github.com/google/go-querystring/query). Use `QueryStruct` to encode a struct as query parameters on requests.

```go
// Github Issue Parameters
type IssueParams struct {
    Filter    string `url:"filter,omitempty"`
    State     string `url:"state,omitempty"`
    Labels    string `url:"labels,omitempty"`
    Sort      string `url:"sort,omitempty"`
    Direction string `url:"direction,omitempty"`
    Since     string `url:"since,omitempty"`
}
```

```go
githubBase := rsling.New().Base("https://api.github.com/").Client(httpClient)

path := fmt.Sprintf("repos/%s/%s/issues", owner, repo)
params := &IssueParams{Sort: "updated", State: "open"}
req, err := githubBase.New().Get(path).QueryStruct(params).Request()
```

### Body

#### JSON Body
Define [JSON tagged structs](https://golang.org/pkg/encoding/json/). Use `BodyJSON` to JSON encode a struct as the Body on requests.

```go
type IssueRequest struct {
    Title     string   `json:"title,omitempty"`
    Body      string   `json:"body,omitempty"`
    Assignee  string   `json:"assignee,omitempty"`
    Milestone int      `json:"milestone,omitempty"`
    Labels    []string `json:"labels,omitempty"`
}
```

```go
githubBase := rsling.New().Base("https://api.github.com/").Client(httpClient)
path := fmt.Sprintf("repos/%s/%s/issues", owner, repo)

body := &IssueRequest{
    Title: "Test title",
    Body:  "Some issue",
}
req, err := githubBase.New().Post(path).BodyJSON(body).Request()
```

Requests will include an `application/json` Content-Type header.

#### Form Body

Define [url tagged structs](https://godoc.org/github.com/google/go-querystring/query). Use `BodyForm` to form url encode a struct as the Body on requests.

```go
type StatusUpdateParams struct {
    Status             string   `url:"status,omitempty"`
    InReplyToStatusId  int64    `url:"in_reply_to_status_id,omitempty"`
    MediaIds           []int64  `url:"media_ids,omitempty,comma"`
}
```

```go
tweetParams := &StatusUpdateParams{Status: "writing some Go"}
req, err := twitterBase.New().Post(path).BodyForm(tweetParams).Request()
```

Requests will include an `application/x-www-form-urlencoded` Content-Type header.

#### Plain Body

Use `Body` to set a plain `io.Reader` on requests created by a rsling.

```go
body := strings.NewReader("raw body")
req, err := rsling.New().Base("https://example.com").Body(body).Request()
```

Set a content type header, if desired (e.g. `Set("Content-Type", "text/plain")`).

### Extend a rsling

Each rsling creates a standard `http.Request` (e.g. with some path and query
params) each time `Request()` is called. You may wish to extend an existing rsling to minimize duplication (e.g. a common client or base url).

Each rsling instance provides a `New()` method which creates an independent copy, so setting properties on the child won't mutate the parent rsling.

```go
const twitterApi = "https://api.twitter.com/1.1/"
base := rsling.New().Base(twitterApi).Client(authClient)

// statuses/show.json rsling
tweetShowrsling := base.New().Get("statuses/show.json").QueryStruct(params)
req, err := tweetShowrsling.Request()

// statuses/update.json rsling
tweetPostrsling := base.New().Post("statuses/update.json").BodyForm(params)
req, err := tweetPostrsling.Request()
```

Without the calls to `base.New()`, `tweetShowrsling` and `tweetPostrsling` would reference the base rsling and POST to
"https://api.twitter.com/1.1/statuses/show.json/statuses/update.json", which
is undesired.

Recap: If you wish to *extend* a rsling, create a new child copy with `New()`.

### Sending

#### Receive

Define a JSON struct to decode a type from 2XX success responses. Use `ReceiveSuccess(successV interface{})` to send a new Request and decode the response body into `successV` if it succeeds.

```go
// Github Issue (abbreviated)
type Issue struct {
    Title  string `json:"title"`
    Body   string `json:"body"`
}
```

```go
issues := new([]Issue)
resp, err := githubBase.New().Get(path).QueryStruct(params).ReceiveSuccess(issues)
fmt.Println(issues, resp, err)
```

Most APIs return failure responses with JSON error details. To decode these, define success and failure JSON structs. Use `Receive(successV, failureV interface{})` to send a new Request that will automatically decode the response into the `successV` for 2XX responses or into `failureV` for non-2XX responses.

```go
type GithubError struct {
    Message string `json:"message"`
    Errors  []struct {
        Resource string `json:"resource"`
        Field    string `json:"field"`
        Code     string `json:"code"`
    } `json:"errors"`
    DocumentationURL string `json:"documentation_url"`
}
```

```go
issues := new([]Issue)
githubError := new(GithubError)
resp, err := githubBase.New().Get(path).QueryStruct(params).Receive(issues, githubError)
fmt.Println(issues, githubError, resp, err)
```

Pass a nil `successV` or `failureV` argument to skip JSON decoding into that value.

### Modify a Request

rsling provides the raw http.Request so modifications can be made using standard net/http features. For example, in Go 1.7+ , add HTTP tracing to a request with a context:

```go
req, err := rsling.New().Get("https://example.com").QueryStruct(params).Request()
// handle error

trace := &httptrace.ClientTrace{
   DNSDone: func(dnsInfo httptrace.DNSDoneInfo) {
      fmt.Printf("DNS Info: %+v\n", dnsInfo)
   },
   GotConn: func(connInfo httptrace.GotConnInfo) {
      fmt.Printf("Got Conn: %+v\n", connInfo)
   },
}

req = req.WithContext(httptrace.WithClientTrace(req.Context(), trace))
client.Do(req)
```

### Build an API

APIs typically define an endpoint (also called a service) for each type of resource. For example, here is a tiny Github IssueService which [lists](https://developer.github.com/v3/issues/#list-issues-for-a-repository) repository issues.

```go
const baseURL = "https://api.github.com/"

type IssueService struct {
    rsling *rsling.rsling
}

func NewIssueService(httpClient *http.Client) *IssueService {
    return &IssueService{
        rsling: rsling.New().Client(httpClient).Base(baseURL),
    }
}

func (s *IssueService) ListByRepo(owner, repo string, params *IssueListParams) ([]Issue, *http.Response, error) {
    issues := new([]Issue)
    githubError := new(GithubError)
    path := fmt.Sprintf("repos/%s/%s/issues", owner, repo)
    resp, err := s.rsling.New().Get(path).QueryStruct(params).Receive(issues, githubError)
    if err == nil {
        err = githubError
    }
    return *issues, resp, err
}
```

### Controlling lifetime via context
All the above functionality of a rsling can be made context aware.

Getting a context aware request:
```go
ctx, cancel := context.WithTimeout(context.Background(),10*time.Second)
req, err := rsling.New().Get("https://example.com").RequestWithContext(ctx)
```
Receiving in a context aware manner
```go
success := &struct{}{}
failure := &struct{}{}
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
resp, err := rsling.New().Path("https://example.com").Get("/foo").ReceiveWithContext(ctx,success,failure)
```
After making the request you can first check whether request completed in time before proceeding with the response:
```go
if errors.Is(err, context.DeadlineExceeded) {
    // Take action accordingly
}
```
For more details about effectively using context please see: https://go.dev/blog/context

## Example APIs using rsling

* Digits [rsjethani/go-digits](https://github.com/rsjethani/go-digits)
* GoSquared [drinkin/go-gosquared](https://github.com/drinkin/go-gosquared)
* Kala [ajvb/kala](https://github.com/ajvb/kala)
* Parse [fergstar/go-parse](https://github.com/fergstar/go-parse)
* Swagger Generator [swagger-api/swagger-codegen](https://github.com/swagger-api/swagger-codegen)
* Twitter [rsjethani/go-twitter](https://github.com/rsjethani/go-twitter)
* Stacksmith [jesustinoco/go-smith](https://github.com/jesustinoco/go-smith)

Create a Pull Request to add a link to your own API.

## Motivation

Many client libraries follow the lead of [google/go-github](https://github.com/google/go-github) (our inspiration!), but do so by reimplementing logic common to all clients.

This project borrows and abstracts those ideas into a rsling, an agnostic component any API client can use for creating and sending requests.

## Contributing

See the [Contributing Guide](https://gist.github.com/rsjethani/be682c123727f70bcfe7).

## License

[MIT License](LICENSE)
