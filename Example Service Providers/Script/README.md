# Script Service Provider Examples

Each `.json` file in this folder is a Script service provider you can import into Profisee. The provider's JavaScript is stored in the `scripts[0].Configuration.scriptContent` property as a single JSON string, so its quotes are written as `\"` and its line breaks as `\r\n`. Connect turns these back into a normal script when the file is imported. This page explains what each script does and shows each script formatted for reading.

## What the Import and Export scripts have in common

- `import { fetch } from "Profisee";` makes Profisee's `fetch` function available. It sends an HTTP request and returns the response directly, without `await`.
- `retryErrorCodes` lists the HTTP statuses worth retrying (408, 429, 500, 502, 503, 504).
- `execute(input)` is the function Connect calls. It returns `isSuccess`, plus `error` and `shouldRetry` when the call fails. Import scripts also return the `records` to import.
- Every request is checked the same way, and a failure returns `isSuccess: false` with the error:
  - An HTTP error status sets `shouldRetry` to `true` only for the statuses in `retryErrorCodes`.
  - A request that could not be sent at all (for example, the host could not be reached) passes on the `shouldRetry` value that `fetch` reports.
  - Any other problem, such as a response that is not valid JSON, sets `shouldRetry` to `false`.
- `input.EndpointUrl` is the **Endpoint URL** setting from the service configuration. It is a required setting in all three providers. The Import scripts also check it and fail with "Endpoint URL is not set" if it is empty.

## ScriptProvider_Import_Example

An Import that reads every record in one call. The default endpoint is `https://jsonplaceholder.typicode.com/users`, a public test API that returns 10 records with placeholder data.

`execute`:

1. Sends a GET request to the Endpoint URL.
2. Parses the response, which must be a JSON array.
3. Turns each item into a record: `id` becomes `Code`, `name` becomes `Name`, and `email` becomes `Email`.
4. Returns the records, which Connect imports using the strategy's Import Map.

`input.records` is always empty for an Import, so the script does not use it.

### Script

This is the `scriptContent` from [ScriptProvider_Import_Example.json](ScriptProvider_Import_Example.json), formatted for reading.

```javascript
import { fetch } from "Profisee";

// Import example: one GET call to a REST endpoint, no authentication.
// The endpoint must return a JSON array of objects with id, name and email properties.
// The default, https://jsonplaceholder.typicode.com/users, is a public test API with placeholder data.
// Import scripts receive no source records. The script fetches data and returns it in records,
// and each record's keys must match the output keys defined on the service.
const retryErrorCodes = [408, 429, 500, 502, 503, 504];

export function execute(input) {
  var url = input.EndpointUrl;
  if (!url) {
    return { isSuccess: false, error: "Endpoint URL is not set", shouldRetry: false };
  }

  var response = fetch(url, {
    method: "GET",
    headers: { "Accept": "application/json" }
  });

  if (response.exceptionType) {
    return {
      isSuccess: false,
      error: response.error || "Network error",
      shouldRetry: response.shouldRetry || false
    };
  }

  if (!response.ok) {
    return {
      isSuccess: false,
      error: "HTTP " + response.status + " from " + url,
      shouldRetry: retryErrorCodes.indexOf(response.status) >= 0
    };
  }

  var items = null;
  try {
    items = JSON.parse(response.body);
  } catch (e) {
    return { isSuccess: false, error: "Failed to parse response", shouldRetry: false };
  }

  if (!Array.isArray(items)) {
    return { isSuccess: false, error: "Expected a JSON array from " + url, shouldRetry: false };
  }

  var records = [];
  for (var i = 0; i < items.length; i++) {
    var item = items[i];
    records.push({
      Code: item.id != null ? String(item.id) : null,
      Name: item.name || null,
      Email: item.email || null
    });
  }

  return { isSuccess: true, records: records };
}
```

## ScriptProvider_PaginatedImport_Example

An Import that reads records one page at a time, for sources too large for one call. The default endpoint is `https://jsonplaceholder.typicode.com/comments`, which has 500 records. The page size is the service's `maximumRecordsPerRequest` of 100, or the strategy's Maximum Records Per Job if that is set lower.

The script has two functions that Connect calls:

- `paginate` runs once at the start. It requests a single record (`?_start=0&_limit=1`) and returns the `X-Total-Count` response header as `totalRecords`. Connect divides that by the page size to decide how many pages, and so how many jobs, to run.
- `execute` runs once per page. `input.paging` holds the `pageNumber` (starting at 1) and the `pageSize`. The script requests that page with `_start=(pageNumber - 1) * pageSize` and `_limit=pageSize`, then builds `Code`, `Name`, and `Email` records the same way as the Import example.

Three helper functions keep `paginate` and `execute` short: `get` sends the request and handles errors, `pageUrl` adds the paging parameters to the Endpoint URL, and `getTotalCount` reads the header. For an API that pages or counts differently, change `pageUrl` and `getTotalCount`. `response.headers` does not include content headers, so a total reported in `Content-Range` cannot be read this way.

### Script

This is the `scriptContent` from [ScriptProvider_PaginatedImport_Example.json](ScriptProvider_PaginatedImport_Example.json), formatted for reading.

```javascript
import { fetch } from "Profisee";

// Paginated Import example: GET calls to a REST endpoint, no authentication.
// The endpoint must accept _start (offset) and _limit (page size) query parameters, return a JSON array
// of objects with id, name and email properties, and report the total record count in an X-Total-Count
// response header. The default, https://jsonplaceholder.typicode.com/comments, is a public test API
// with placeholder data. For another API, change pageUrl and getTotalCount to match how it pages.
// Connect calls paginate once per run to learn how many records there are, splits them into pages,
// then calls execute once per page with input.paging = { pageNumber, pageSize }. pageNumber starts at 1.
const retryErrorCodes = [408, 429, 500, 502, 503, 504];

function get(url) {
  var response = fetch(url, {
    method: "GET",
    headers: { "Accept": "application/json" }
  });

  if (response.exceptionType) {
    return {
      failure: {
        isSuccess: false,
        error: response.error || "Network error",
        shouldRetry: response.shouldRetry || false
      }
    };
  }

  if (!response.ok) {
    return {
      failure: {
        isSuccess: false,
        error: "HTTP " + response.status + " from " + url,
        shouldRetry: retryErrorCodes.indexOf(response.status) >= 0
      }
    };
  }

  return { response: response };
}

// Adds the paging parameters, whether or not the endpoint URL already has a query string.
function pageUrl(input, start, limit) {
  var url = input.EndpointUrl;
  var separator = url.indexOf("?") >= 0 ? "&" : "?";
  return url + separator + "_start=" + start + "&_limit=" + limit;
}

// Header names are not case-sensitive, so look the header up without regard to case.
function getTotalCount(response) {
  var headers = response.headers || {};
  for (var name in headers) {
    if (name.toLowerCase() === "x-total-count") {
      return Number(headers[name]);
    }
  }
  return NaN;
}

function missingEndpoint(input) {
  if (!input.EndpointUrl) {
    return { isSuccess: false, error: "Endpoint URL is not set", shouldRetry: false };
  }
  return null;
}

// Called once per run. Returns the total number of records the source holds.
// input.pageSize is the page size Connect will use for this run.
export function paginate(input) {
  var invalid = missingEndpoint(input);
  if (invalid) {
    return invalid;
  }

  // Ask for a single record; the X-Total-Count header holds the count for the whole source.
  var result = get(pageUrl(input, 0, 1));
  if (result.failure) {
    return result.failure;
  }

  var total = getTotalCount(result.response);
  if (isNaN(total)) {
    return { isSuccess: false, error: "The response has no X-Total-Count header", shouldRetry: false };
  }

  return { isSuccess: true, totalRecords: total };
}

// Called once per page.
export function execute(input) {
  var invalid = missingEndpoint(input);
  if (invalid) {
    return invalid;
  }

  var pageNumber = input.paging.pageNumber;
  var pageSize = input.paging.pageSize;
  var start = (pageNumber - 1) * pageSize;

  var result = get(pageUrl(input, start, pageSize));
  if (result.failure) {
    return result.failure;
  }

  var items = null;
  try {
    items = JSON.parse(result.response.body);
  } catch (e) {
    return { isSuccess: false, error: "Failed to parse response", shouldRetry: false };
  }

  if (!Array.isArray(items)) {
    return { isSuccess: false, error: "Expected a JSON array from " + input.EndpointUrl, shouldRetry: false };
  }

  var records = [];
  for (var i = 0; i < items.length; i++) {
    var item = items[i];
    records.push({
      Code: item.id != null ? String(item.id) : null,
      Name: item.name || null,
      Email: item.email || null
    });
  }

  return { isSuccess: true, records: records };
}
```

## ScriptProvider_Export_Example

An Export that sends Profisee records to an endpoint. The default endpoint is `https://httpbin.org/post`, which echoes the request back and stores nothing.

`execute`:

1. Reads `input.records`. Each record holds the `Code`, `Name`, and `Email` values mapped on the strategy's Export Map tab. If there are none, it returns success without calling the endpoint.
2. Sends them as a JSON POST: `{ "records": [ { "code": ..., "name": ..., "email": ... } ] }`.
3. Returns only `isSuccess`, plus the error when the request fails. An Export writes nothing back to Profisee.

### Script

This is the `scriptContent` from [ScriptProvider_Export_Example.json](ScriptProvider_Export_Example.json), formatted for reading.

```javascript
import { fetch } from "Profisee";

// Export example: sends Profisee records to a REST endpoint as a JSON POST.
// The default endpoint, https://httpbin.org/post, echoes the request back and stores nothing,
// so the example runs as-is. Point Endpoint URL at your own API to use it for real.
// Export scripts return only a status. Any records returned are ignored.
const retryErrorCodes = [408, 429, 500, 502, 503, 504];

export function execute(input) {
  var records = input.records || [];
  if (records.length === 0) {
    return { isSuccess: true };
  }

  var payload = [];
  for (var i = 0; i < records.length; i++) {
    var record = records[i];
    payload.push({
      code: record.Code,
      name: record.Name,
      email: record.Email || null
    });
  }

  var response = fetch(input.EndpointUrl, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Accept": "application/json"
    },
    body: JSON.stringify({ records: payload })
  });

  if (response.exceptionType) {
    return {
      isSuccess: false,
      error: response.error || "Network error",
      shouldRetry: response.shouldRetry || false
    };
  }

  if (!response.ok) {
    return {
      isSuccess: false,
      error: "HTTP " + response.status + ": " + response.body,
      shouldRetry: retryErrorCodes.indexOf(response.status) >= 0
    };
  }

  return { isSuccess: true };
}
```

## ScriptProvider_Example

An Enrichment provider that shows every part of a service configuration: each input presenter type, Service and Data scope inputs, outputs, and OAuth2 `authSettings`. Its script is an empty template: it imports `fetch` and `getOAuthToken`, defines `retryErrorCodes`, and leaves `execute` for you to write.

### Script

This is the `scriptContent` from [ScriptProvider_Example.json](ScriptProvider_Example.json), formatted for reading.

```javascript
import { fetch, getOAuthToken } from "Profisee";
const retryErrorCodes =
    [
        408, // Request Timeout  
        429, // Too Many Requests  
        500, // Internal Server Error  
        502, // Bad Gateway  
        503, // Service Unavailable  
        504, // Gateway Timeout
    ];

export function execute(input) {
    // write your code here
}
```

For the full input and output contract of each flow, see the [Script Service Provider Documentation](../../Service%20Provider%20Documentation/Script%20Service%20Provider%20Documentation.md).
