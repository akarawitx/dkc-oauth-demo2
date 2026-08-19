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
**Request** — `POST /api/cas/request`  for start approve request
    `client_id`,`client_secret`,`ExternalRefID`,
    `ApproveType` : 1 = HeadKong only (หัวหน้ากอง)
                    2 = HeadKong and HeadSamnak (หัวหน้าสำนัก)
    `Requester_ADUser`,
    `CallbackURL`,
    `MsgSubject` | optional
    `MsgHTMLForHead` | optional for Email and HR App
    `MsgFlexForHead` | optional for LINE

    response JSON data {
            "success": true,
            "ADApprover": "xxx",
            "RequestKey": "xxxx",
            "HeadFullName": xxx xxxx",
            "HeadShowEmail": "xxx@xxx.com",
            "Position": "xxx",
            "Organization": "xxxx"
        }

**Request Status** — `POST /api/cas/reqstat`
    `client_id`,`ExternalRefID`

    response JSON data {
            'client_id',
            'ExternalRefID',
            'ApproveType',
            'ApproveStep',
            'Requester_ADUser',
            'Approver_ADUser',
            'Title',
            'Summary',
            'DetailJSON',
            'Status',
            'CallbackURL',
            'created_at',
            'updated_at',
        }

### DKC OAuth Message (Central Messaging System)
**Send Message** — `POST /api/send-message`

    JSON Body {
        "client_id": xx,
        "client_secret": "aaa",
        "app_name": "MY_APP",
        "subject": "ขออนุมัติ",
        "channel": | `email`, `line`, or `both`,
        "toAd": [
            "xx"
        ],
        "to" | conditional | Array of literal destinations (email addresses)
        "email_body": HTML "สวัสดีครับ",
        "line_text": "มีข้อความใหม่ถึงคุณ",
        "line_flex": json
            {
            "type": "flex",
            "altText": "ใบขอซื้อ WO022665 ได้รับอนุมัติ",
            "contents": { "type": "bubble", "body": { "...": "..." } }
            }
    }

    response JSON {
        "status": "success",
        "tracking_id": "6c29dfac-3073-4ed8-9915-134f82ae4ded",
        "message": [
            "Email sent 1",
            "LINE sent 1"
        ]
    }
