# DKC OAuth forDev

## DKC OAuth API Service URL Endpoints

### DKC OAuth Login
**Authorize** — `GET /oauth/authorize` 
   `client_id`,`redirect_uri`, `response_type=code`, `state`
    (https://oauth.dhammakaya.network/oauth/authorize?client_id=xx&redirect_uri=http://xxx:8000/callback&response_type=code)

**Token** — `POST /oauth/token`
    `grant_type=authorization_code`, `client_id`,`client_secret`,`redirect_uri`,`code`
    response JSON with <access_token>,<refresh_token>

**Profile** — `GET /api/user` with `Authorization: Bearer` <access_token>
    response JSON {
                    "id": 12,
                    "name": "somchai",
                    "email": "somchai@example.org",
                    "email_verified_at": null,
                    "created_at": "2024-07-22T07:56:18.870000Z",
                    "updated_at": "2024-07-22T07:56:18.870000Z",
                    "username": "somchai",
                    "display_name": "สมชาย ใจดี",
                    "line_internal_id": "U1234...",
                    "line_name": "somchai",
                    "line_picture": "https://..."
                  }

**Logout** — `GET /logout` using with browser redirect, not an API call
    option `?redirect_url=` 


### DKC OAuth CAS (Central Approval System)
