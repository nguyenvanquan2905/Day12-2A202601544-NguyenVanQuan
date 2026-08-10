# Đặt ảnh chụp màn hình bản deploy vào thư mục này

(.venv) PS D:\AI_THUCCHIEN\Day12-2A202601544-NguyenVanQuan> $URL = "https://day12-chat-production-4e57.up.railway.app"
(.venv) PS D:\AI_THUCCHIEN\Day12-2A202601544-NguyenVanQuan> $TOKEN = "WY3_fspdHVKU6SgnBSPAPH7cPoRsQhXS40b4vL7-x7c"
(.venv) PS D:\AI_THUCCHIEN\Day12-2A202601544-NguyenVanQuan>
(.venv) PS D:\AI_THUCCHIEN\Day12-2A202601544-NguyenVanQuan> curl.exe -i "$URL/healthz"
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 10 Aug 2026 09:59:55 GMT
Server: railway-hikari
x-railway-request-id: QggZumbdR9iKKdmq9fVATg
Content-Length: 64
x-hikari-trace: sin1.tr00
x-railway-edge: sin1
Connection: keep-alive

{"status":"ok","service":"day12-chat-service","version":"1.0.0"}
(.venv) PS D:\AI_THUCCHIEN\Day12-2A202601544-NguyenVanQuan> curl.exe -i "$URL/readyz"
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 10 Aug 2026 09:59:55 GMT
Server: railway-hikari
x-railway-request-id: KvqXbfwCTUCK6nI_Y53eZw
Content-Length: 31
x-hikari-trace: sin1.hs0s
x-railway-edge: sin1
Connection: keep-alive

{"status":"ready","redis":true}
(.venv) PS D:\AI_THUCCHIEN\Day12-2A202601544-NguyenVanQuan> curl.exe -i -X POST "$URL/chat" -H "Content-Type: application/json" -d '{\"message\": \"test\"}'
HTTP/1.1 401 Unauthorized
Content-Type: application/json
Date: Mon, 10 Aug 2026 09:59:55 GMT
Server: railway-hikari
www-authenticate: Bearer
x-railway-request-id: vHNoaAHFR2mtTari2h0iww
Content-Length: 44
x-hikari-trace: sin1.98a6
x-railway-edge: sin1
Connection: keep-alive

{"detail":"invalid or missing bearer token"}
(.venv) PS D:\AI_THUCCHIEN\Day12-2A202601544-NguyenVanQuan> curl.exe -i -X POST "$URL/chat" -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" -d '{\"message\": \"Xin chao\"}'
HTTP/1.1 200 OK
Content-Type: application/json
Date: Mon, 10 Aug 2026 09:59:58 GMT
Server: railway-hikari
x-railway-request-id: jSsTEVXETzqwJaJ_YqVb7A
Content-Length: 276
x-hikari-trace: sin1.d1nj
x-railway-edge: sin1
vary: accept-encoding
Connection: keep-alive

{"reply":"Với Xin chao, cách làm phổ biến trong production là đặt một lớp gateway phía trước để lo authentication, rate limiting và bảo vệ chi phí.","client_id":"anonymous","turns_before":0,"usd_cost":2.07e-05,"usage":{"prompt":2,"completion":34}}