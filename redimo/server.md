---
title: Redis-Compatible Server for Redimo
layout: default
description: A RESP server can let you talk to Redimo/DyanmoDB as if it were Redis, with all your existing code and clients. 

# Micro navigation
micro_nav: false
buttons:
    - content: REDIMO
      url: /redimo/
      external_url: false
    - icon: github
      content: REDIMO.GO
      url: https://github.com/dbProjectRED
      external_url: true

# Page navigation
# page_nav:
#     prev:
#         content: REDIMO
#         url: /redimo/
    # next:
    #     content: Implementing Strings
    #     url: '#'

---

While a Go library, or even language-specific libraries are great, it would be nice to have a server that speaks the RESP protocol that allows you to use DynamoDB as if it were Redis. Now that we have the Redis operations mapped on to their DynamoDB equivalents, we can build such a server. 

The current plan is to have a hosted service available for immediate use, which will scale quickly and easily to any level of load. If you want the server yourself, the code will be available under the [SSPL License](https://www.mongodb.com/licensing/server-side-public-license).

If you're interested in early access please contact me at [@sudhirj](https://twitter.com/sudhirj) or [sudhir.j@gmail.com](mailto:sudhir.j@gmail.com).