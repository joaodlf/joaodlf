---
title: "Python: Just write SQL"
date: 2023-08-10
keywords:
- python
- sql
- postgresql
- orm
- raw sql
- sqlalchemy
- django orm
---

I have been writing a lot more Go this past year. For those not familiar, Go favours a non-ORM, non-query-builder approach to interacting with databases. This comes naturally due to the [sql package](https://pkg.go.dev/database/sql): A common interface to be used alongside database drivers. It's very common to see actual SQL in Go, even in large projects. On the other hand, Python does not have anything in the standard library that supports database interaction, this has always been a problem for the community to solve. There are many ORMs and query builders for Python: 
- [SQLAlchemy](https://www.sqlalchemy.org/) 
- [Django ORM](https://docs.djangoproject.com/en/4.2/#the-model-layer) 
- [Peewee](http://docs.peewee-orm.com/en/latest/) (my personal favourite)
- ... plenty more!

You can of course just take your favourite adapter and write SQL in Python (which is what we'll be doing!), but I think most developers will agree that, as soon as you need to interact with a database, it is far more probable to immediately reach for an ORM like SQLAlchemy. 

(This is not to say Go doesn't have popular [ORMs](https://github.com/go-gorm/gorm), but you're much more likely to see Go devs leveraging the standard library and database adapters, this is the typical approach new Go devs will be directed to.)

In this post, I will aim to approach SQL in Python in the same vein as Go: 
- I want to write SQL. 
- I don't want to rely on a query builder (let alone an ORM).
- I want to package all of this in an abstraction that allows me to quickly change between database solutions, as well as make it easy to test.
- l want a very clear separation between my database(s) and business logic.

## Show me the code

We are going to start from a typical example:

```python
# user/domain.py

@dataclass  
class User:  
	id: int = None  
	dt_created: datetime = None
	username: str = None
	email: str = None
	mobile: Optional[str] = None
```

`User` is the literal definition of what you might see as a SQL table named `user`. This includes some columns I like to include in all of my SQL tables:
- `id` (primary key).
- `dt_created` a datetime value, typically defaults to `NOW()`.

All fields default to `None`, this will allow us to perform `SELECT` queries that don't include all of the columns in the table, while still being able to convert rows to `User` objects. All columns are `NOT NULL` with the exception of `mobile`, here I make use of the `Optional` type hint to stress that this field allows for `NULL`.

We can now create our repository:
```python
# user/repository.py

from abc import ABC, abstractmethod

class UserRepository(ABC):  
	@abstractmethod  
	def new(self, new_row: User) -> int:  
	"""  
	Create a new record and returns the new id value. 
	"""  
	pass  
  
	@abstractmethod  
	def get_by_id(self, id_val: int) -> User:  
	"""  
	Get a single record by ID value.  
	"""  
	pass  
```

I like my repositories to be abstract classes, this makes sure any future implementations need to follow the contract (as much as you can with Python, best intentions and all). 

Our first implementation is going to be **PostgreSQL** via [psycopg3](https://www.psycopg.org/psycopg3/docs/index.html):

```python
# external/postgres.py

import psycopg

# Load up your config from somewhere...
from config import get_config

cfg = get_config()

conn = psycopg.connect(
    dbname=cfg.database_name,
    user=cfg.database_username,
    password=cfg.database_password,
    host=cfg.database_host,
)

def new_cursor(name=None, row_factory=None):
    if name is None:
        name = ""

    if row_factory is None:
        return conn.cursor(name=name)

    return conn.cursor(name=name, row_factory=class_row(row_factory))
```

Firstly, we setup the connection. Secondly, a function that returns psycopg cursors. psycopg3 includes [custom row factories](https://www.psycopg.org/psycopg3/docs/advanced/rows.html), we will shortly see how useful this feature is.

We now implement the repository:
```python
# user/repository.py
from psycopg.rows import class_row  
from external.postgres import conn

class UserPostgreSQL(UserRepository):
    def new(self, new_row: User) -> int:
        with new_cursor() as cur:
            cur.execute(
                """
            INSERT INTO user
            (username, email)
            VALUES
            (%s, %s)
            ON CONFLICT (email)
            DO NOTHING
            RETURNING id;
            """,
                (
                    new_row.username,
                    new_row.email
                ),
            )
            
            new_id = cur.fetchone()
            if new_id and len(new_id):
                conn.commit()
                new_id = new_id[0]
            else:
                new_id = 0
        
        return new_id

    def get_by_id(self, id_val: int) -> User:
        with new_cursor(name="get_by_id", row_factory=User) as cur:
            cur.execute(
                """
            SELECT *
            FROM user
            WHERE id = %s
            """,
                (id_val,),
            )
            
            return cur.fetchone()
            
def new_user_repo() -> UserRepository:
    return UserPostgreSQL()

```

You'll notice `new()` uses a basic cursor, but `get_by_id()` uses a cursor with a **class row factory**, which means the returning row will be of type `User`. Nifty. 

`new_user_repo()` is the function we use in our hypothetical business logic, the implementation itself is never exposed:
```python
from user.repository import *  
  
user_repo = new_user_repo()  
  
row = User()  
row.username = "joao"  
row.email = "joao@nospam.com"  
  
# Create a new record.  
new_id = user_repo.new(row)  
  
# Fetch the record we just created.  
new_row = user_repo.get_by_id(new_id)
# User(id=1, dt_created=datetime.datetime(2023, 8, 9, 23, 8, 14, 974074, tzinfo=datetime.timezone.utc), username='joao', email='joao@nospam.com', mobile=None)
```

`new_user_repo()` can be modified to return anything that implements  `UserRepository`, maybe a MySQL implementation, or a SQLite one, possibly even a mock for test purposes. 

## Conclusion

I have spent enough time in tech to see languages and frameworks fall out of grace, libraries and tools coming and going. **SQL has always been a constant** and I see tremendous benefits in just writing it. In my experience, anything that brings you closer to your database install (even in the age of the cloud), is a good thing.

The implementation above does not break any new ground, in fact, it may even look fairly similar to an ORM API... But that was always the goal: Writing raw SQL does not have to mean unstructured code.

I hope at least one Python developer out there reconsiders their approach to SQL next time a new project is fired up.
