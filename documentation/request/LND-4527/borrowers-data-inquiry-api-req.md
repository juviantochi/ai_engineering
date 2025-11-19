repository scope: fs-brick-service

The output of this request is API Documentation template for `Borrowers Data Inquiry` endpoint.

Endpoint : `POST` `'fs/brick/service/veefin/v1/inquiry-borrowers'`

Request should contain :
- header :
`veefin-x-token` containing static token generated for veefin.
- request :
```json
{
  "ktpNumber": "1234567890123456",
  "requestId": "{{random-uuid}}"
}
```
- response:

```json
{
  "ktpNumber": "1234567890123456",
  "requestId": "{UUID matching the one in request}",
  "code": 200,
  "description" : "success",
  "data": {
    "jmlFasilitas": 750,
    "jmlHariTunggakanTerburuk" : 12,
    "jmlSaldoTerutang" : 1233.12
  }
}
```

error code list:
- 200 : success
- 400 : bad request (invalid ktp number format, missing fields, etc)
- 401 : invalid Token
- 404 : data not found for given ktp number
