---
title: "Compojure Exception Handling"
date: 2017-09-01T22:06:35+02:00
draft: true
---


tests

`curl -X POST -d '{"username": "aaa", "email": ""}' -H "Content-Type: application/json" http://localhost:3000/users`

errore validazione 1)

`curl -X POST -d '{"username": "aaa", "email": "aaa"}' -H "Content-Type: application/json" http://localhost:3000/users`

ok

`curl -X POST -d '{"username": "aaa"}' -H "Content-Type: application/json" http://localhost:3000/users`

errore validazione 2)

`curl -X POST -d '{"username": "fooo" "email": "bar"}' -H "Content-Type: application/json" http://localhost:3000/users`

throw generico
