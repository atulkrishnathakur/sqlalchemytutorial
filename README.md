
# scalars()
```
1. Without .scalars() (session.execute(stmt)):
SQLAlchemy will return a result structured like this:
[(<User object at 0x1...>,), (<User object at 0x2...>,)]
Note: Here, each User object is wrapped inside a tuple ( ).
If you want to access the first user object, you would have to write result[0][0], which is quite awkward and inconvenient.

2. With .scalars() (session.execute(stmt).scalars()):
This removes the outer envelope (wrapper) of the tuple and directly gives you a list of the objects:
[<User object at 0x1...>, <User object at 0x2...>]
Now, you can directly use result[0] to access the first user object.
```
