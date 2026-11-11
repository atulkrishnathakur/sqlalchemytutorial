# Example of Code for SELECT

```
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from sqlalchemy.orm import DeclarativeBase
from fastapi import Depends
from typing import AsyncGenerator, Annotated
from sqlalchemy.ext.asyncio import AsyncAttrs
from decouple import config

from sqlalchemy import (BigInteger,Column,PrimaryKeyConstraint,Text,String,Integer,DateTime,
BigInteger,SmallInteger,func,UniqueConstraint,ForeignKey,Identity)
from datetime import datetime
from sqlalchemy.orm import Mapped, declarative_base, mapped_column
from sqlalchemy.orm.base import Mapped
from app.config.dbconnection import Base

from fastapi import FastAPI,Depends, HTTPException, Response, Request

from app.lifespans.lifespan import lifespan_check

from fastapi.responses import JSONResponse

from pydantic import BaseModel, ConfigDict, Field, EmailStr, AfterValidator, BeforeValidator
from typing import Optional, Annotated
from fastapi import status

from sqlalchemy import select
from datetime import date
from datetime import datetime
from fastapi import APIRouter,Depends,status,Path
from fastapi.encoders import jsonable_encoder
from fastapi.responses import JSONResponse

from typing import Annotated
from sqlalchemy.orm import Session
from sqlalchemy import (select,insert,update,delete,join,and_, or_ )
from pydantic import (BaseModel,Field, model_validator, EmailStr, ModelWrapValidatorHandler, ValidationError, AfterValidator,BeforeValidator,PlainValidator, ValidatorFunctionWrapHandler)
from typing import Optional, Annotated

PROJECT_NAME = config("PROJECT_NAME", default='', cast=str)
DEBUG = config("DEBUG", default=False, cast=bool)
DB_DRIVER = config("DB_DRIVER", default='mysql+aiomysql', cast=str)
DB_HOST = config("DB_HOST", default='localhost', cast=str)
DB_NAME = config("DB_NAME", default='fastapi_crud_1', cast=str)
DB_USER = config("DB_USER", default='root', cast=str)
DB_PASSWORD = config("DB_PASSWORD", default='123456789', cast=str)
DB_PORT = config("DB_PORT", default='3306', cast=str)
DATABASE_URL = f"{DB_DRIVER}://{DB_USER}:{DB_PASSWORD}@{DB_HOST}:{DB_PORT}/{DB_NAME}?charset=utf8mb4"

engine = create_async_engine(DATABASE_URL, echo=True, future=True)

async_session = async_sessionmaker(bind=engine, expire_on_commit=False)

class Base(AsyncAttrs, DeclarativeBase):
  pass

async def get_session():
    async with async_session() as session:
        yield session

SessionDep = Annotated[AsyncSession, Depends(get_session)]


class CourseGroupMaster(Base):
    __tablename__ = 'course_group_master'
    # use Mapped[ python datatype ] 
    # server_default used for mysql server 
    # default attribute use if you want python calculation
    id: Mapped[int] = mapped_column('id',BigInteger, primary_key=True, autoincrement=True, nullable=False)
    course_group_code: Mapped[str] = mapped_column('course_group_code',String(255),nullable=True)
    course_group_name: Mapped[str] = mapped_column('course_group_name',String(255),nullable=True)
    status: Mapped[int] = mapped_column('status',SmallInteger,nullable=True,server_default='1',comment="1=Active,0=Inactive")
    created_at: Mapped[datetime] = mapped_column('created_at',DateTime, nullable=True, server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column('updated_at',DateTime,nullable=True)
    created_by: Mapped[int] = mapped_column('created_by',BigInteger,nullable=True)
    updated_by: Mapped[int] = mapped_column('updated_by',BigInteger,nullable=True)


class CourseMaster(Base):
    __tablename__ = 'course_master'
    # use Mapped[ python datatype ] 
    # server_default used for mysql server 
    # default attribute use if you want python calculation
    id: Mapped[int] = mapped_column('id',BigInteger, primary_key=True, autoincrement=True, nullable=False)
    course_code: Mapped[str] = mapped_column('course_code',String(50),nullable=True)
    course_group_id: Mapped[int] = mapped_column('course_group_id',BigInteger,nullable=True)
    course_name: Mapped[str] = mapped_column('course_name',String(255),nullable=True)
    status: Mapped[int] = mapped_column('status',SmallInteger,nullable=True,server_default='1',comment="A=Active,I=Inactive")
    created_at: Mapped[datetime] = mapped_column('created_at',DateTime, nullable=True, server_default=func.now())
    updated_at: Mapped[datetime] = mapped_column('updated_at',DateTime,nullable=True)
    created_by: Mapped[int] = mapped_column('created_by',BigInteger,nullable=True)
    updated_by: Mapped[int] = mapped_column('updated_by',BigInteger,nullable=True)
    repository_course_status: Mapped[int] = mapped_column('repository_course_status',SmallInteger,nullable=True,server_default='1',comment="1=Active,0=Inactive")


app = FastAPI(
    lifespan=lifespan_check,
    debug=DEBUG, # False on production
    title=PROJECT_NAME,
    summary="This is a very simple crud application",
    description="This is very simple curd aplication",
    version="0.1.0",
    openapi_url="/simple_crud_openapi.json",
    docs_url="/docs",
    redoc_url="/redoc",
    default_response_class=JSONResponse
)


def error_responses():
    common_responses = {
        400: {"description": "Bad Request"},
        404: {"description": "Not Found"},
        422: {"description": "Validation Error"},
    }
    return common_responses

class CustomeAPIException(Exception):
    def __init__(self, status_code: int | None = 422, status: bool|None = False, message: str | None = None, data: list = None):
        self.status_code = status_code
        self.status = status
        self.message = message
        self.data = data

async def custom_api_exception_handler(request: Request, exc: CustomeAPIException):
    return JSONResponse(
        status_code=exc.status_code,
        content={
            "status_code": exc.status_code,
            "status": exc.status,
            "message": exc.message,
            "data": exc.data
        }
    )


class Datacheck:
    @classmethod
    def datetime_format(cls, value: str) -> Optional[str]:
        if value is None or value == "":
            return ""
            
        # Convert incoming SQLAlchemy datetime objects to custom string format
        if isinstance(value, datetime):
            return value.strftime("%d-%m-%Y %H:%M:%S")
           
        return str(value)
    
DateFormat = Annotated[Optional[str], BeforeValidator(Datacheck.datetime_format), Field(examples=["2025-03-26 22:33:59", "2025-04-23 22:33:59"])]


class CourseOut(BaseModel):
    id: int = Field(..., examples=[5])
    course_name: str = Field(..., examples=['Python'])
    course_code: Optional[str] = None
    course_group_id: Optional[int] = None
    status: str = Field(..., examples=['A','I'])
    created_at: DateFormat = ""
    updated_at: DateFormat = ""
    created_by: Optional[int] = None
    updated_by: Optional[int] = None
    repository_course_status: int = Field(..., examples=[1])

    model_config = ConfigDict(from_attributes=True)



class CourseRes(BaseModel):
    status_code: int
    message: str
    status: bool
    data: list[CourseOut]


app.add_exception_handler(CustomeAPIException, custom_api_exception_handler)


router = APIRouter()

@router.get(
        "/course-list",
        #response_model=CourseRes,
        responses=error_responses(),
        name="courselist"
        )
async def course_list(
    session: SessionDep,
    courseGroupId: Optional[int] = None
    ):
    try:
        stmt = select(CourseMaster) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.all()
        #print(dataList[0][0].course_name)
        print(dataList)

    except Exception as e:
        return f"exception {e}"

app.include_router(router)

```

# all()
1. code
```
        stmt = select(CourseMaster) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.all()
        print(dataList)
```
If you use only all() then object will be come in tuple in list. output of code is below
```
[(<app.sqlalchemy.CourseMaster object at 0x7681ae7fc6e0>,), (<app.sqlalchemy.CourseMaster object at 0x7681ae7c91d0>,), (<app.sqlalchemy.CourseMaster object at 0x7681ae7c9090>,), (<app.sqlalchemy.CourseMaster object at 0x7681ae9129e0>,), (<app.sqlalchemy.CourseMaster object at 0x7681ae912b10>,), (<app.sqlalchemy.CourseMaster object at 0x7681ae7b8dd0>,), (<app.sqlalchemy.CourseMaster object at 0x7681ae936030>,), (<app.sqlalchemy.CourseMaster object at 0x7681ae935f20>,), (<app.sqlalchemy.CourseMaster object at 0x7681ae7c6150>,), (<app.sqlalchemy.CourseMaster object at 0x7681ae7c6050>,), (<app.sqlalchemy.CourseMaster object at 0x7681ae7d2e40>,)]
```
To get data, write code like below
```
        stmt = select(CourseMaster) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.all()
        print(dataList[0][0].course_name)
```

# use all() to select specific field
```
        stmt = select(CourseMaster.course_name, CourseMaster.id) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.all()
        print(dataList[0][0])
```

# scalars().all()
1. code
```
        stmt = select(CourseMaster) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.scalars().all()
        print(dataList)
```

If you use scalars().all() then objects will be come in list. output of code is below
```
[<app.sqlalchemy.CourseMaster object at 0x70a5dbf006e0>, <app.sqlalchemy.CourseMaster object at 0x70a5dbed1090>, <app.sqlalchemy.CourseMaster object at 0x70a5dbed0f50>, <app.sqlalchemy.CourseMaster object at 0x70a5dc0168b0>, <app.sqlalchemy.CourseMaster object at 0x70a5dc0169e0>, <app.sqlalchemy.CourseMaster object at 0x70a5dbebd010>, <app.sqlalchemy.CourseMaster object at 0x70a5dc039f20>, <app.sqlalchemy.CourseMaster object at 0x70a5dc039e10>, <app.sqlalchemy.CourseMaster object at 0x70a5dbece050>, <app.sqlalchemy.CourseMaster object at 0x70a5dbecdf50>, <app.sqlalchemy.CourseMaster object at 0x70a5dbedae40>]
```
To get data, write code like below
```
        stmt = select(CourseMaster) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.scalars().all()
        print(dataList[0].course_name)
```
# use scalars().all() to select specific field
```
        stmt = select(CourseMaster.course_name, CourseMaster.id) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.scalars().all()
        print(dataList[0])
```
Here it will print only course name not id because scalar fetch only one field value if you want to fetch specific field data


# scalar()
do not use all() with scalar() because scalar() fetch only first record as object
```
        stmt = select(CourseMaster) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.scalar()
        print(dataList.course_name)
```

If you fetch specific field then only then scalar() fetch first record and first field of object that you have written in code.
```
        stmt = select(CourseMaster.course_name, CourseMaster.id) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.scalar()
        print(dataList)
```

# scalars().all() and scalars().fetchall()
1. both are work same


# scalars().fetchmany(size=...)
1. If you have millions of data then fetchall() and all() will not use because it load all data in ram and system will crash
2. You can use scalars().fetchmany(size=...) to fetch row data in objects in chunk in list
   ```
        stmt = select(CourseMaster) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        while True:
            chunk = result.scalars().fetchmany(size=2)
            
            if not chunk:
                break  # Agar data khatam ho jaye toh loop rok do
                
            for course in chunk:
                print(course.course_name)

   ```
# fetchmany(size=...)
1. If you have millions of data then fetchall() and all() will not use because it load all data in ram and system will crash
2. You can use fetchmany(size=...) to fetch row objects in tuple in chunk
   ```
        stmt = select(CourseMaster) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        while True:
            chunk = result.fetchmany(size=2)
            
            if not chunk:
                break  # Agar data khatam ho jaye toh loop rok do
                
            for course in chunk:
                print(course[0].course_name)

   ```


# first()
1. It return first row in object in tuple
   ```
        stmt = select(CourseMaster) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.first()
        print(dataList[0].course_name)
   ```
2. It return first row values in tuple that you have written in code
   ```
        stmt = select(CourseMaster.course_name,CourseMaster.id) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.first()
        print(dataList[0])
   ```

# scalars().first()
1. scalars().first() return first row in object
   ```
        stmt = select(CourseMaster) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.scalars().first()
        print(dataList.course_name)
   ```

2. scalars().first() return first row of first field value
   ```
        stmt = select(CourseMaster.course_name, CourseMaster.id) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.scalars().first()
        print(dataList)
   ```

# mappings().all()
1. result.mappings() converts query result in dictionary in key-value pairs
2. mappings().all() fetch all data in object in dictionary in list
   ```
        stmt = select(CourseMaster) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.mappings().all()
        print(dataList[0]['CourseMaster'].course_name)
   ```

# use mappings().all() to select specific field
1. result.mappings() converts query result in dictionary in key-value pairs
2. mappings().all() fetch data in object in dictionary in list for all specific field
   ```
        stmt = select(CourseMaster.course_name, CourseMaster.id) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.mappings().all()
        print(dataList)
   ```
   output:
   ```
      [{'course_name': 'C', 'id': 1}, {'course_name': 'HTML', 'id': 2}, {'course_name': 'CSS', 'id': 3}, {'course_name': 'JavaScript', 'id': 4}, {'course_name': 'PHP', 'id': 5}, {'course_name': 'Laravel', 'id': 6}, {'course_name': 'Python', 'id': 7}, {'course_name': 'MySQL', 'id': 8}, {'course_name': 'FastAPI', 'id': 9}, {'course_name':'postgresql', 'id': 10}, {'course_name': 'Docker', 'id': 11}]
   ```

# mappings().all() and mappings().fetchall()
1. both are same


# mappings().first()
1. result.mappings() converts query result in dictionary in key-value pairs
2. mappings().all() fetch first row data in object in dictionary in list
```
      stmt = select(CourseMaster) # it create the sql query
      result = await session.execute(stmt) # execute() run the sql query on the connected database
      dataList = result.mappings().first()
      print(dataList['CourseMaster'].course_name)
```
3. mappings().all() fetch first row specific data in dictionary
   ```
        stmt = select(CourseMaster.id, CourseMaster.course_name) # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.mappings().first()
        print(dataList['course_name'])
   ```

# mappings().fetchmany(size=...)
1. If you have millions of data then fetchall() and all() will not use because it load all data in ram and system will crash
2. You can use mappings().fetchmany(size=...) to fetch row data in objects id dictionary in chunk in list
   ```
     stmt = select(CourseMaster) # it create the sql query
     result = await session.execute(stmt) # execute() run the sql query on the connected database
     while True:
         chunk = result.mappings().fetchmany(size=2)
         
         if not chunk:
             break  # Agar data khatam ho jaye toh loop rok do
             
         for course in chunk:
             print(course['CourseMaster'].course_name)

   ```

# where clause
1. For where() used for where clause
   ```
    stmt = select(CourseMaster).where(CourseMaster.course_name == 'C')
   ```
   With `and` operator
   ```
    stmt = select(CourseMaster).where(CourseMaster.course_name == 'C', CourseMaster.course_name == 'A')
   ```
   and` operator with where chain
   ```
      stmt = select(CourseMaster).where(CourseMaster.course_name == 'C').where(CourseMaster.course_name == 'A')
   ```
   
# and_() 
1. and_() function used for `AND` operator
   ```
       stmt = select(CourseMaster).where(
            and_(
                CourseMaster.course_name == 'C',
                CourseMaster.course_name == 'A'
                )
            ) 
   ```

# or_()
1. or_() function used for `OR` operator
   ```
        stmt = select(CourseMaster).where(
            or_(
                CourseMaster.course_name == 'C',
                CourseMaster.course_name == 'A'
                )
            )
   ```
# not_()
1. not_() function used for `NOT` operator
   ```
      stmt = select(CourseMaster).where(
            not_(CourseMaster.status == 'A')
            ) # it create the sql query
   ```
2. the `tiled(~)` also used for not
   ```
      stmt = select(CourseMaster).where(
            ~(CourseMaster.status == 'A')
            )
   ```
# in_()
1. in_() function used for `IN` operator
   ```
      stmt = select(CourseMaster).where(
            CourseMaster.id.in_([1,2,3])
            )
   ```
# notin_()
```
    stmt = select(CourseMaster).where(
        CourseMaster.id.notin_([1,2,3])
        ) # it create the sql query
```
# is_(None) for `IS NULL` of sql

```
stmt = select(CourseMaster).where(
            CourseMaster.created_at.is_(None)
            )
```

# is_(None) and not_() for `IS NOT NULL` of SQL
```
stmt = select(CourseMaster).where(
            not_(CourseMaster.created_at.is_(None))
            )
```

# `LIKE` operator for searching
1. The like() method is case-sensitive 
```
stmt = select(CourseMaster).where(
            CourseMaster.course_name.like('%th%')
            ) 
```
2. The ilike() method is a case-insensitive. it will match upper and lower both cases
```
stmt = select(CourseMaster).where(
            CourseMaster.course_name.ilike('%jsc%') # it match jsc or JSC both
            )
```
# `NOT LIKE` operator for searching
1. The tiled(`~`) used for not
```
stmt = select(CourseMaster).where(
            ~CourseMaster.course_name.like('%jsc%')
            )
```
2. The `not_()` user for not
   ```
    stmt = select(CourseMaster).where(
            not_(CourseMaster.course_name.like('%jsc%'))
            )
   ```

# `BETWEEN` operator
1. The `between()` method used for between 
```
        stmt = select(CourseMaster).where(
            CourseMaster.id.between(2,7)
            ) # it create the sql query
```

# `ORDER BY`
1. The `order_by()` and `asc()` method used for ascending order 
   ```
    stmt = select(CourseMaster).order_by(CourseMaster.id.asc()) 
   ```
2. The `order_by()` and `desc()` method used for descending order
   ```
   stmt = select(CourseMaster).order_by(CourseMaster.id.desc()) 
   ```

# SUB QUERY
```
# 1. Inner Subquery: Get user IDs who placed an order valued over $500
# SQL: SELECT user_id FROM orders WHERE total_amount > 500
subq_stmt = (
    select(Order.user_id)
    .where(Order.total_amount > 500)
)

# 2. Outer Main Query: Fetch User records matching those IDs
# SQL: SELECT * FROM users WHERE id IN (SELECT user_id FROM orders WHERE total_amount > 500)
stmt = (
    select(User)
    .where(User.id.in_(subq_stmt))
)
```

# INNER JOIN
1. inner join table with `ON` clause. Here it fetch data only from User
  ```
  stmt = select(User).join(Order, User.id == Order.customer_id)
  ```
2. inner join table with `ON` clause. Here it fetch data from both table
   ```
   stmt = select(User, Order).join(Order, User.id == Order.customer_id)

   ```
3. inner join multiple table
   ```
    stmt = (
        select(User.name, Order.id, Payment.amount)
        .join(Order)
        .join(Payment)
    )
   ```
4. inner join with where clause
   ```
    stmt = (
        select(User)
        .join(Order)
        .where(Order.status == 'shipped')
    )
   ```

# LEFT JOIN ( OUTER JOIN )
1. outerjoin() used for left join
```
stmt = select(User, Order).outerjoin(Order, User.id == Order.customer_id)

```
2. The `isouter=True)` use for left( outer) join
   ```
      stmt = select(User, Order).join(Order, User.id == Order.customer_id, isouter=True)
   ```

# How to run raw sql query. text() function used to run raw query
1. text() function used to run raw query. with scalars() fetch only first field data in list here id field value will be fetch in list
   ```
        stmt = text("select course_name, id from course_master") # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.scalars().all()
        print(dataList)
   ```
2.  text() function used to run raw query. with scalars() fetch only first field data in list here course_name will be fetch
   ```
        stmt = text("select * from course_master")
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.scalars().all()
        print(dataList)
   ```
3. text() function used to run raw query. with mappings() fetch all field data in dictionary
   ```
        stmt = text("select course_name, id from course_master") 
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.mappings().all()
        print(dataList)
   ```
4. text() function used to run raw query. with mappings() fetch all field data in dictionary
   ```
        stmt = text("select * from course_master") # it create the sql query
        result = await session.execute(stmt) # execute() run the sql query on the connected database
        dataList = result.mappings().all()
        print(dataList)
   ```

# How to run raw sql query and how to handle sql injection. text() function used to run raw query
1. Use colon(:) placeholder and pass values in dictionary to handle sql injection
   ```
        stmt = text("select * from course_master where id > :min_id and status = :course_status ") # it create the sql query
        result = await session.execute(stmt, {"min_id": 2, "course_status": 'A'}) # execute() run the sql query on the connected database
        dataList = result.mappings().all()
        print(dataList)
   ```

# MySQL and MariaDB Data Types
1. As with all SQLAlchemy dialects, all UPPERCASE types that are known to be valid with MySQL are importable from the top level dialect.
2. You can use in sqlalchemy in application. It is specially created for MySQL or MariaDB. It support MySQL or MariaDB features
3. reference: https://docs.sqlalchemy.org/en/21/dialects/mysql.html#mysql-data-types
     
```
      from sqlalchemy.dialects.mysql import (
          BIGINT,
          BINARY,
          BIT,
          BLOB,
          BOOLEAN,
          CHAR,
          DATE,
          DATETIME,
          DECIMAL,
          DECIMAL,
          DOUBLE,
          ENUM,
          FLOAT,
          INTEGER,
          LONGBLOB,
          LONGTEXT,
          MEDIUMBLOB,
          MEDIUMINT,
          MEDIUMTEXT,
          NCHAR,
          NUMERIC,
          NVARCHAR,
          REAL,
          SET,
          SMALLINT,
          TEXT,
          TIME,
          TIMESTAMP,
          TINYBLOB,
          TINYINT,
          TINYTEXT,
          VARBINARY,
          VARCHAR,
          YEAR,
      )
```

