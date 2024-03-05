# Games Global QA Test Task Report

## Deliverables

This directory provides the following files:
- `README.md` - this report in Markdown format
- `tests.json` - a Postman collection containing the required tests
- `env.json` - a Postman environment definition, which must be supplied when running the tests
- `test-results.html` - a HTML document describing the execution result of the tests

The Postman collection is further comprised of two folders:

- `OpenAPI Schema Tests` - tests verifying the conformance of the API to its [OpenAPI specification](https://reqres.in/api-docs/swagger-ui-init.js), automatically generated using [Portman](https://github.com/apideck-libraries/portman). Ideally, these tests would not be checked into source control but rather be generated and ran on the fly in CI based on the OpenAPI schema the API publishes.

- `Tests` - functional tests that aim to verify expected behavior of the API. Please note that these tests may omit some schema validations, as they are already covered by the tests derived from the OpenAPI specification.

### Running tests

Dependencies:

- node

```sh
npm i
npx newman run tests.json -e env.json
```

### Generating HTML report of test execution

```sh
npm i
# The report will appear in the `newman` subdirectory
npx newman run tests.json -e env.json -r htmlextra
```

## Findings

### Missing OpenAPI schema definitions

This is not a defect in the API itself, but rather its OpenAPI specification.

Some response definitions (namely, `404`) for certain endpoints have been omitted. This means that
tests derived from the schema fail due to a contract violation - see below:

```
  #  failure               detail                                                                              
                                                                                                               
 1.  AssertionError        [GET]::/:resource/:id - Status code is 2xx                                          
                           expected response code to be 2XX but found 404                                      
                           at assertion:0 in test-script                                                       
                           inside "OpenAPI Schema Tests / {resource} / {id} / Fetches an unknown resource"     
                                                                                                               
 2.  AssertionError        [GET]::/users/:id - Status code is 2xx                                              
                           expected response code to be 2XX but found 404                                      
                           at assertion:0 in test-script                                                       
                           inside "OpenAPI Schema Tests / users / {id} / Fetches a user"                       
                                                                                                               
 3.  AssertionError        [POST]::/login - Status code is 2xx                                                 
                           expected response code to be 2XX but found 400                                      
                           at assertion:0 in test-script                                                       
                           inside "OpenAPI Schema Tests / login / Creates a session"                           
                                                                                                               
 4.  AssertionError        [POST]::/register - Status code is 2xx                                              
                           expected response code to be 2XX but found 400                                      
                           at assertion:0 in test-script                                                       
                           inside "OpenAPI Schema Tests / register / Creates a user"                           
```

The schema also didn't include:

- a `POST` mapping for `/users`
- a `POST` mapping for `/{resource}` (e.g. `/{unknown}`)

The failing tests have been omitted for the time being.

### Lack of any password validation

#### Authentication

`/login` accepts any password for any user, and returns an authentication token.

We assume that this is the expected behavior of the mock API and have not added any test cases to cover this.

In a real scenario, this part of the system would of course be tested according to whatever the established password
requirements are.

#### Registration

`/register` allows any non-blank password, and has no length or complexity requirements.

We assume that this is the expected behavior of the mock API and have not added any test cases to cover this.

### Duplicate keys in request bodies

It seems that the application accepts duplicate keys in various request bodies - for example, when `PATCH`ing a `User` schema.

There are two sensible ways to deal with this scenario:
- reject the request by issuing a `400` (`Bad Request`)
- select one of the values deterministically

The application already exhibits the latter behaviour by always selecting the last specified key in the body,
so tests have been added to ensure that this strategy continues to be enforced.

For example, given the request:

```json
{
    "job": "A",
    "job": "B"
}
```

We assert that the response should contain:

```json
{
    "job": "B"
    // [...]
}
```

### Incorrect handling of invalid path parameters

The API responds with `2xx` status codes to requests with invalid parameter values that it ought to reject.

#### `GET` `/users?page=<int>&per_page=<int>`

The API **must** reject any request where either `page` or `path` are not positive integers with a `400 Bad Request`.

Currently, the API responds by reinterpreting any values that fail validation to some internal default value.
For example, completely nonsensical requests like `GET /users?page=apple` are served the first page of users.

#### `GET` `/users/{id}`

*The same points apply.*

#### `GET` `/{resource}`

*The same points apply.*

#### `GET` `/{resource}/{id}`

*The same points apply.*


## Recommendations

In conclusion, we recommend that:

- The OpenAPI schema is updated to accurately reflect the behavior of the API to enable integration standardized tooling
- The handling of duplicate keys in requests continues to be implemented consistently
- The validation surrounding password authentication is revised, such that it is secure and robust
- The validation surrounding path parameters is revised, such that clearly invalid requests are rejected

