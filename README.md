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
    `Requester_KongId`,  ขออนุมัติด้วย รหัสกอง
            Requester_ADUser กับ Requester_KongId ต้องส่งอย่างหนึ่ง
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
        "client_id": 12,
        "client_secret": "aaa",
        "app_name": "MY_APP",
        "subject": "ขออนุมัติ",

        // channel: string เดิม ("email" | "line" | "both")
        // หรือ array ของ key ("email" | "line" | "hr_mobile")
        // หรือ "all" / ไม่ระบุ = ส่งทุก channel ที่ระบบรู้จัก
        "channel": ["email", "line", "hr_mobile"],

        "toAd": ["xx"],                    // required ถ้าเลือก channel ใดๆ ที่ไม่ใช่ email (line, hr_mobile)
        "to": ["someone@example.com"],     // required_without: toAd
        "email_source": "hr",              // nullable, "hr" | "ad", default "hr"

        "message": "ข้อความกลาง ใช้เป็น fallback ของทุก channel",
        // หมายเหตุ: hr_mobile ไม่มี field override เฉพาะตัว
        // ดังนั้นถ้าเลือกส่ง hr_mobile ต้องมี "message" เสมอ

        "email_body": "<p>สวัสดีครับ</p>",   // override เฉพาะ email (แทน message)
                                            // ** ถ้ามี HTML tag จริง และเลือกส่ง hr_mobile ด้วย
                                            //    ระบบจะแนบลิงก์ดูเนื้อหานี้ไปใน mobile push อัตโนมัติ **

        "line_text": "มีข้อความใหม่ถึงคุณ",  // override เฉพาะ line แบบ text (แทน message)
        "line_flex": {                     // override เฉพาะ line แบบ flex (มีความสำคัญกว่า line_text ถ้าใส่มาพร้อมกัน)
            "type": "flex",
            "altText": "ใบขอซื้อ WO022665 ได้รับอนุมัติ",
            "contents": { "type": "bubble", "body": { "...": "..." } }
        }
    }

    Response 200 (success — บาง channel อาจ fail แต่ status ยังเป็น success) {
        "status": "success",
        "tracking_id": "6c29dfac-3073-4ed8-9915-134f82ae4ded",
        "message": [
            "Email sent 1",
            "LINE sent 1",
            "HR Mobile push failed (500)"   // ตัวอย่างกรณี HR mobile ล้มเหลว แต่ request ยัง success
        ]
    }

    Response 200 (skipped — ไม่พบผู้รับที่ส่งได้เลยในทุก channel) {
        "status": "skipped",
        "tracking_id": "no_valid_recipient",
        "message": "ไม่พบช่องทางที่สามารถส่งข้อความได้สำหรับผู้รับที่ระบุ"
    }

    Response 401/422 (client_id / client_secret ไม่ถูกต้อง หรือ validate ไม่ผ่าน) {
        "errors": { "...": ["..."] }
    }

    Response 500 (exception ที่ไม่ถูกจับใน channel) {
        "status": "error",
        "message": "ส่งข้อความไม่สำเร็จ <รายละเอียด exception>"
    }
