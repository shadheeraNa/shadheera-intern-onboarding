# BookSwap — Mock Smoke Test Report

## Setup

- **Initial test date:** 19 July 2026
- **Bruno verification date:** 23 July 2026
- **Environment:** Windows, Visual Studio Code integrated Command Prompt
- **OpenAPI file:** `shadheera-nanayakkara-day2-bookswap-openapi.yaml`
- **Mock server:** Stoplight Prism on `http://127.0.0.1:4010`
- **Command-line API client:** `curl.exe`
- **Graphical API client:** Bruno 3.5.3
- **Bruno request type:** HTTP
- **Bruno collection:** `tests/bruno/BookSwap Smoke Tests`
- **Authentication used for positive tests:** `Authorization: Bearer test-token`

The OpenAPI specification was validated before starting the mock server:

```cmd
npx @apidevtools/swagger-cli validate shadheera-nanayakkara-day2-bookswap-openapi.yaml
```

Validation result:

```text
shadheera-nanayakkara-day2-bookswap-openapi.yaml is valid
```

The Prism mock server was started with:

```cmd
npx @stoplight/prism-cli mock shadheera-nanayakkara-day2-bookswap-openapi.yaml --port 4010
```

Successful startup message:

```text
Prism is listening on http://127.0.0.1:4010
```

The five smoke tests were first executed with `curl.exe` and were later recreated as saved HTTP requests in Bruno. Each Bruno request contains a `res.status` assertion for its expected status code. The Bruno Collection Runner completed with five passed requests, zero failed requests, and zero skipped requests.

Bruno evidence:

The saved Bruno collection is stored at `tests/bruno/BookSwap Smoke Tests`, and the supporting screenshots are stored in the `bruno-testing/` directory.

## Reproduction Instructions

1. Open the folder containing the OpenAPI YAML file in Visual Studio Code.
2. Open an integrated **Command Prompt** terminal.
3. Run the validation command shown above.
4. Start Prism using the mock-server command shown above.
5. Leave the Prism terminal running.
6. Open a second Command Prompt terminal.
7. Run each test command below and record the HTTP status shown on the first response line.

### Reproducing with Bruno

1. Open Bruno.
2. Open the collection at `tests/bruno/BookSwap Smoke Tests`.
3. Confirm that Prism is still running on `http://127.0.0.1:4010`.
4. Open the collection runner.
5. Select all five requests.
6. Keep iterations set to `1`, parallel execution disabled, and timing and tag filters empty.
7. Run the collection.
8. Confirm that all five requests and all five `res.status` assertions pass.

## Test Commands and Results

### Test 1 — List books with pagination

```cmd
curl.exe -i "http://127.0.0.1:4010/books?page=1&pageSize=20" -H "Authorization: Bearer test-token"
```

- **Expected status:** `200`
- **Actual status:** `200`
- **Result:** Pass

Prism returned a response following the `BookPage` schema with `items`, `page`, `pageSize`, `total`, and `totalPages`.

### Test 2 — Create a book with a valid payload

```cmd
curl.exe -i -X POST "http://127.0.0.1:4010/books" -H "Authorization: Bearer test-token" -H "Content-Type: application/json" -d "{\"title\":\"Clean Code\",\"author\":\"Robert C. Martin\",\"isbn\":\"9780132350884\",\"condition\":\"GOOD\",\"available\":true}"
```

- **Expected status:** `201`
- **Actual status:** `201`
- **Result:** Pass

The response also included a `Location` header for the created resource.

### Test 3 — Create a book with the required title missing

```cmd
curl.exe -i -X POST "http://127.0.0.1:4010/books" -H "Authorization: Bearer test-token" -H "Content-Type: application/json" -d "{\"author\":\"Robert C. Martin\",\"isbn\":\"9780132350884\",\"condition\":\"GOOD\",\"available\":true}"
```

- **Expected status:** `400` or `422`
- **Actual status:** `422`
- **Result:** Pass

Prism identified the missing required property and returned:

```text
Request body must have required property 'title'
```

The response followed the reusable validation-error structure and included `code`, `message`, `requestId`, and field-level `details`.

### Test 4 — Request to borrow book `999`

```cmd
curl.exe -i -X POST "http://127.0.0.1:4010/books/999/borrow-requests" -H "Authorization: Bearer test-token"
```

- **Expected status:** `201`
- **Actual status:** `201`
- **Result:** Pass

The request body was omitted because the borrower message is optional.

### Test 5 — List books without an Authorization header

```cmd
curl.exe -i "http://127.0.0.1:4010/books?page=1&pageSize=20"
```

- **Expected status:** `401`
- **Actual status:** `401`
- **Result:** Pass

Prism enforced the global bearer security requirement and returned the reusable unauthorized response.

## Results Summary

| Metric | Target | Achieved |
|---|---:|---:|
| Tests run | 5 | 5 |
| Tests passing against the mock | 5 | 5 |
| Bruno assertions passing | 5 | 5 |
| Bruno runner failures | 0 | 0 |
| Negative tests present | 2 | 2 |
| API operations with explicit error responses | At least 4 | 15 of 15 |
| OpenAPI validation | No errors | Valid |

## Detailed Results

| # | Endpoint | Method | Body / Parameters | Expected | Actual | Result |
|---:|---|---|---|---:|---:|---|
| 1 | `/books` | GET | `page=1&pageSize=20` | 200 | 200 | Pass |
| 2 | `/books` | POST | Valid book payload | 201 | 201 | Pass |
| 3 | `/books` | POST | Valid JSON with `title` omitted | 400 or 422 | 422 | Pass |
| 4 | `/books/999/borrow-requests` | POST | Bearer header; no optional body | 201 | 201 | Pass |
| 5 | `/books` | GET | No `Authorization` header | 401 | 401 | Pass |

## Bruno Collection Evidence

The same five smoke tests were recreated as manually configured HTTP requests in Bruno and executed against the Prism mock server.

| Request | Assertion | Runner result |
|---|---|---|
| `01 - List books` | `res.status equals 200` | Passed |
| `02 - Create valid book` | `res.status equals 201` | Passed |
| `03 - Create book missing title` | `res.status equals 422` | Passed |
| `04 - Borrow book 999` | `res.status equals 201` | Passed |
| `05 - List books without authorization` | `res.status equals 401` | Passed |

The Bruno Collection Runner reported **5 passed**, **0 failed**, and **0 skipped**. The main runner screenshot is stored at `evidence/bruno-all-tests-passed.png`.

## Findings

### 1. Prism returns specification examples rather than implementing business behaviour

The valid `POST /books` request submitted **Clean Code**, but Prism returned the fixed **The Hobbit** example from the OpenAPI response. This is expected from a contract mock: it returns documented examples and does not store or echo submitted records like a real backend.

This confirmed that realistic and internally consistent examples are important because they become the visible mock responses used by API consumers.

### 2. The mock cannot verify database existence or resource state

`POST /books/999/borrow-requests` returned `201` even though a real system would first check whether book `999` exists and is available. Prism has no Azure SQL database and cannot evaluate rules such as:

- Whether the book exists
- Whether it belongs to the member's building
- Whether it is currently available
- Whether the borrower already has a pending request
- Whether the borrower owns the book

Those checks must be implemented and integration-tested in the real Node.js service.

### 3. Prism checks the bearer-header contract, not a real Microsoft Entra JWT

The mock correctly returned `401` when the `Authorization` header was omitted. However, it accepted the placeholder value `Bearer test-token`. Therefore, this test confirms that the security requirement is present in the OpenAPI contract, but it does not verify JWT signatures, issuers, audiences, expiry, or Microsoft Entra claims.

Real authentication testing must be performed against the implemented API and Microsoft Entra External ID.

### 4. Required-field validation was useful and specific

The missing-title test returned `422` and identified the exact missing property. This showed that the `required` list in `CreateBookRequest` is effective and that the reusable validation response is suitable for client-side error handling.

## Endpoints That Felt Awkward to Call

The endpoint paths and HTTP methods were generally predictable.

The only slightly confusing mock interaction was `POST /books/{bookId}/borrow-requests`. Its request body is optional, but the single documented success example contains a borrower message. When the endpoint was called without a body, Prism still returned the example message. Providing separate examples for requests with and without a message would make the mock behaviour clearer.

The Windows Command Prompt also requires escaped quotation marks for JSON payloads, making the `POST /books` curl command long. This is a command-shell issue rather than an API design problem.

## Specification Changes I Would Make

Line numbers refer to the validated version of `shadheera-nanayakkara-day2-bookswap-openapi.yaml` used for this report. They may shift after editing.

1. **Lines 110–140 — Add named create-book examples.**  
   Replace the single request and response examples under `POST /books` with named examples such as `hobbit` and `cleanCode`. Each response example should correspond to its request example. This would reduce confusion when Prism returns a fixed example rather than the submitted payload.

2. **Lines 342–350 — Add a borrow request example without a message.**  
   Change the single `201` example under `POST /books/{bookId}/borrow-requests` into named examples:
   - `withMessage`, containing a message string
   - `withoutMessage`, containing `message: null`

   This would reflect both valid ways of calling the endpoint and better match Smoke Test 4.

3. **Lines 355–356 and the reusable responses near lines 928–937 — Add a book-specific not-found response.**  
   Define a reusable `BookNotFound` response with:
   - `code: BOOK_NOT_FOUND`
   - `message: The requested book was not found.`

   Reference it from `POST /books/{bookId}/borrow-requests` instead of the generic `RESOURCE_NOT_FOUND` response. This would make the documented failure more precise, even though Prism still cannot determine whether a database record exists without being instructed to return the `404` example.

## Conclusion

All five required smoke tests passed against the Prism mock server using both `curl.exe` and the saved Bruno HTTP collection. Bruno's Collection Runner reported five passed requests, zero failures, and zero skipped requests, with each expected status code verified through a `res.status` assertion. The tests confirmed that pagination, successful creation, required-field validation, bearer-header enforcement, response examples, and documented status codes are represented correctly in the OpenAPI contract.

The mock also demonstrated its limits: it does not persist data, evaluate database state, or validate a real Microsoft Entra JWT. These behaviours must be verified later through integration and end-to-end tests against the implemented backend.
