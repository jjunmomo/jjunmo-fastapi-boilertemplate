# FastAPI 보일러플레이트 패턴 가이드

이 문서는 jjunmo-fastapi 보일러플레이트의 **재사용 가능한 아키텍처 패턴**을 정리한다.
새 FastAPI 프로젝트에서 Claude에게 "BOILERPLATE.md 패턴대로 작성해"라고 지시하면 동일 구조로 코드를 생성할 수 있다.

> **범위**: Repository / DI / Exception 3가지 핵심 패턴 + Sync/Async 이중 지원 가이드

---

## 목차

1. [BaseRepository 패턴 (Sync + Async)](#1-baserepository-패턴-sync--async)
2. [Type Alias DI 패턴](#2-type-alias-di-패턴)
3. [ServiceException Factory 패턴](#3-serviceexception-factory-패턴)
4. [새 기능 추가 절차 (8단계)](#4-새-기능-추가-절차-8단계)

---

## 1. BaseRepository 패턴 (Sync + Async)

> **참조**: `repositories/base_repository.py`, `repositories/async_base_repository.py`

### 1-1. Sync/Async 선택 기준

| 상황 | 권장 | 이유 |
|------|------|------|
| 단순 CRUD, CPU 바운드 | **Sync** (`BaseRepository`) | 코드가 단순하고 디버깅이 쉬움 |
| 외부 API 호출, 파일 I/O | **Async** (`AsyncBaseRepository`) | 동시 다발적 I/O에서 성능 이점 |
| 혼합 | 도메인별 선택 | 하나의 프로젝트에서 Sync/Async 혼용 가능 |

### 1-2. Sync BaseRepository

`Generic[T]` 기반 공통 CRUD 추상화. Sync Repository가 이 클래스를 상속한다.

```python
# repositories/base_repository.py
from typing import Any, Dict, Generic, List, Optional, Type, TypeVar
from uuid import UUID
from sqlalchemy import asc, desc, insert
from sqlalchemy.orm import Session

T = TypeVar("T")

class BaseRepository(Generic[T]):
    def __init__(self, model: Type[T], db: Session):
        self.model = model
        self.db = db

    def get_by_id(self, id: UUID) -> Optional[T]:
        return self.db.query(self.model).filter(self.model.id == id).first()

    def get_all(self, skip: int = 0, limit: int = 100) -> List[T]:
        return self.db.query(self.model).offset(skip).limit(limit).all()

    def create(self, obj: T) -> T:
        self.db.add(obj)
        self.db.flush()  # commit이 아닌 flush
        return obj

    def update(self, obj: T) -> T:
        self.db.add(obj)
        self.db.flush()
        return obj

    def delete(self, obj: T) -> None:
        self.db.delete(obj)
        self.db.flush()

    # SQLAlchemy 2.0 현대적 API 사용 (bulk_insert_mappings 대신)
    def bulk_insert(self, mappings: List[Dict[str, Any]]) -> None:
        self.db.execute(insert(self.model), mappings)
        self.db.flush()

    def filter_by(self, **kwargs) -> List[T]:
        return self.db.query(self.model).filter_by(**kwargs).all()

    def filter_by_one(self, **kwargs) -> Optional[T]:
        return self.db.query(self.model).filter_by(**kwargs).first()

    def count(self, **kwargs) -> int:
        query = self.db.query(self.model)
        if kwargs:
            query = query.filter_by(**kwargs)
        return query.count()

    def order_by(self, column: str, direction: str = "desc") -> List[T]:
        sort_column = getattr(self.model, column, None)
        if not sort_column:
            return []
        query = self.db.query(self.model)
        if direction == "desc":
            query = query.order_by(desc(sort_column))
        else:
            query = query.order_by(asc(sort_column))
        return query.all()
```

### 1-3. Async BaseRepository

SQLAlchemy 2.0의 `select()` 문법 기반. Async Repository가 이 클래스를 상속한다.

```python
# repositories/async_base_repository.py
from typing import Any, Dict, Generic, List, Optional, Type, TypeVar
from uuid import UUID
from sqlalchemy import asc, desc, func, insert, select
from sqlalchemy.ext.asyncio import AsyncSession

T = TypeVar("T")

class AsyncBaseRepository(Generic[T]):
    def __init__(self, model: Type[T], db: AsyncSession):
        self.model = model
        self.db = db

    async def get_by_id(self, id: UUID) -> Optional[T]:
        result = await self.db.execute(
            select(self.model).filter(self.model.id == id)
        )
        return result.scalars().first()

    async def get_all(self, skip: int = 0, limit: int = 100) -> List[T]:
        result = await self.db.execute(
            select(self.model).offset(skip).limit(limit)
        )
        return list(result.scalars().all())

    async def create(self, obj: T) -> T:
        self.db.add(obj)
        await self.db.flush()
        return obj

    async def update(self, obj: T) -> T:
        self.db.add(obj)
        await self.db.flush()
        return obj

    async def delete(self, obj: T) -> None:
        await self.db.delete(obj)
        await self.db.flush()

    async def bulk_insert(self, mappings: List[Dict[str, Any]]) -> None:
        await self.db.execute(insert(self.model), mappings)
        await self.db.flush()

    async def filter_by(self, **kwargs) -> List[T]:
        result = await self.db.execute(
            select(self.model).filter_by(**kwargs)
        )
        return list(result.scalars().all())

    async def filter_by_one(self, **kwargs) -> Optional[T]:
        result = await self.db.execute(
            select(self.model).filter_by(**kwargs)
        )
        return result.scalars().first()

    async def count(self, **kwargs) -> int:
        query = select(func.count()).select_from(self.model)
        if kwargs:
            query = query.filter_by(**kwargs)
        result = await self.db.execute(query)
        return result.scalar()

    async def order_by(self, column: str, direction: str = "desc") -> List[T]:
        sort_column = getattr(self.model, column, None)
        if not sort_column:
            return []
        query = select(self.model)
        query = query.order_by(
            desc(sort_column) if direction == "desc" else asc(sort_column)
        )
        result = await self.db.execute(query)
        return list(result.scalars().all())
```

**제공 메서드 요약 (Sync/Async 동일):**

| 메서드 | 용도 |
|--------|------|
| `get_by_id(id)` | PK로 단일 엔티티 조회 |
| `get_all(skip, limit)` | 페이지네이션 조회 |
| `create(obj)` | 엔티티 생성 (flush) |
| `update(obj)` | 엔티티 업데이트 (flush) |
| `delete(obj)` | 하드 삭제 (flush) |
| `bulk_insert(mappings)` | 대량 삽입 (SQLAlchemy 2.0 API) |
| `filter_by(**kwargs)` | 조건 필터링 (리스트) |
| `filter_by_one(**kwargs)` | 조건 필터링 (단일) |
| `count(**kwargs)` | 조건별 카운트 |
| `order_by(column, direction)` | 정렬 조회 |

### 1-4. flush() vs commit() 원칙

```
Repository: flush()만 수행 → DB에 SQL 전송하되 트랜잭션은 열어둠
DI 레이어: get_db_with_transaction() / get_async_db_with_transaction()이 commit/rollback 관리
```

- `flush()`: SQL을 DB에 보내되, 트랜잭션을 커밋하지 않음. 같은 세션 내에서 ID 참조 가능.
- `commit()`: Repository에서 직접 호출하지 않음. DI의 트랜잭션 제너레이터가 자동 관리.

이 분리 덕분에 **하나의 Service 메서드 안에서 여러 Repository를 호출해도 단일 트랜잭션**으로 묶인다.

### 1-5. 구체적 Repository 구현 예시

**Sync 예시:**

```python
# repositories/task_repository.py
from typing import List
from uuid import UUID
from sqlalchemy.orm import Session
from models.task import Task, TaskStatus
from repositories.base_repository import BaseRepository

class TaskRepository(BaseRepository[Task]):
    def __init__(self, db: Session):
        super().__init__(Task, db)  # 모델 클래스 전달

    # 도메인별 커스텀 메서드 추가
    def get_by_project(self, project_id: UUID, status: TaskStatus = None) -> List[Task]:
        query = self.db.query(Task).filter(Task.project_id == project_id)
        if status:
            query = query.filter(Task.status == status)
        return query.order_by(Task.created_at.desc()).all()
```

**Async 예시:**

```python
# repositories/task_async_repository.py
from typing import List
from uuid import UUID
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from models.task import Task, TaskStatus
from repositories.async_base_repository import AsyncBaseRepository

class TaskAsyncRepository(AsyncBaseRepository[Task]):
    def __init__(self, db: AsyncSession):
        super().__init__(Task, db)

    async def get_by_project(self, project_id: UUID, status: TaskStatus = None) -> List[Task]:
        query = select(Task).filter(Task.project_id == project_id)
        if status:
            query = query.filter(Task.status == status)
        query = query.order_by(Task.created_at.desc())
        result = await self.db.execute(query)
        return list(result.scalars().all())
```

**패턴 요약:**
1. `BaseRepository[ModelClass]` 또는 `AsyncBaseRepository[ModelClass]` 상속
2. `__init__`에서 `super().__init__(ModelClass, db)` 호출
3. 공통 CRUD는 상속으로 자동 제공
4. 도메인 특화 쿼리만 커스텀 메서드로 추가

---

## 2. Type Alias DI 패턴

> **참조**: `core/database.py`, `dependencies/repositories.py`, `dependencies/services.py`

### 2-1. 이중 세션 전략 (Sync + Async)

트랜잭션 관리를 **세션 팩토리 레벨**에서 분리한다.

```python
# core/database.py

# ── Sync DI 제너레이터 ──
def get_db():
    """읽기 전용 sync 세션"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

def get_db_with_transaction():
    """쓰기용 sync 세션 — 자동 commit/rollback"""
    db = SessionLocal()
    try:
        with db.begin():
            yield db
    except Exception:
        raise
    finally:
        db.close()

# ── Async DI 제너레이터 ──
async def get_async_db():
    """읽기 전용 async 세션"""
    async with AsyncSessionLocal() as session:
        yield session

async def get_async_db_with_transaction():
    """쓰기용 async 세션 — 자동 commit/rollback"""
    async with AsyncSessionLocal() as session:
        async with session.begin():
            yield session
```

| 팩토리 | 용도 | 트랜잭션 |
|--------|------|----------|
| `get_db` | Sync 읽기 전용 | 수동 (필요 시 직접 commit) |
| `get_db_with_transaction` | Sync 생성/수정/삭제 | 자동 commit/rollback |
| `get_async_db` | Async 읽기 전용 | 수동 |
| `get_async_db_with_transaction` | Async 생성/수정/삭제 | 자동 commit/rollback |

### 2-2. Repository DI 등록

3단계: **팩토리 함수** → **트랜잭션 팩토리 함수** → **Type Alias 선언**

**Sync 예시:**

```python
# dependencies/repositories.py
from typing import Annotated
from fastapi import Depends
from sqlalchemy.orm import Session
from core.database import get_db, get_db_with_transaction
from repositories.task_repository import TaskRepository

# Step 1: 읽기 전용 팩토리
def get_task_repository(db: Session = Depends(get_db)) -> TaskRepository:
    return TaskRepository(db)

# Step 2: 트랜잭션 팩토리
def get_task_repository_with_transaction(
    db: Session = Depends(get_db_with_transaction),
) -> TaskRepository:
    return TaskRepository(db)

# Step 3: Type Alias 선언
TaskRepoDep = Annotated[TaskRepository, Depends(get_task_repository)]
TaskRepoTransactionDep = Annotated[TaskRepository, Depends(get_task_repository_with_transaction)]
```

**Async 예시:**

```python
# dependencies/repositories.py
from sqlalchemy.ext.asyncio import AsyncSession
from core.database import get_async_db, get_async_db_with_transaction
from repositories.task_async_repository import TaskAsyncRepository

async def get_task_async_repository(
    db: AsyncSession = Depends(get_async_db),
) -> TaskAsyncRepository:
    return TaskAsyncRepository(db)

async def get_task_async_repository_with_transaction(
    db: AsyncSession = Depends(get_async_db_with_transaction),
) -> TaskAsyncRepository:
    return TaskAsyncRepository(db)

TaskAsyncRepoDep = Annotated[TaskAsyncRepository, Depends(get_task_async_repository)]
TaskAsyncRepoTransactionDep = Annotated[
    TaskAsyncRepository, Depends(get_task_async_repository_with_transaction)
]
```

**네이밍 컨벤션:**

| 구분 | Sync | Async |
|------|------|-------|
| 읽기 전용 Repo | `{Entity}RepoDep` | `{Entity}AsyncRepoDep` |
| 트랜잭션 Repo | `{Entity}RepoTransactionDep` | `{Entity}AsyncRepoTransactionDep` |
| 읽기 전용 Service | `{Entity}ServiceDep` | `{Entity}AsyncServiceDep` |
| 트랜잭션 Service | `{Entity}ServiceTransactionDep` | `{Entity}AsyncServiceTransactionDep` |

### 2-3. Service DI 등록

Service는 Repository Type Alias를 파라미터로 받아 자동 주입된다.

**Sync 예시:**

```python
# dependencies/services.py
from dependencies.repositories import TaskRepoDep, TaskRepoTransactionDep
from services.task_service import TaskService

def get_task_service_read_only(task_repo: TaskRepoDep) -> TaskService:
    return TaskService(task_repo=task_repo)

def get_task_service_transactional(task_repo: TaskRepoTransactionDep) -> TaskService:
    return TaskService(task_repo=task_repo)

TaskServiceDep = Annotated[TaskService, Depends(get_task_service_read_only)]
TaskServiceTransactionDep = Annotated[TaskService, Depends(get_task_service_transactional)]
```

**Async 예시:**

```python
# dependencies/services.py
from dependencies.repositories import TaskAsyncRepoDep, TaskAsyncRepoTransactionDep
from services.task_async_service import TaskAsyncService

def get_task_async_service_read_only(task_repo: TaskAsyncRepoDep) -> TaskAsyncService:
    return TaskAsyncService(task_repo=task_repo)

def get_task_async_service_transactional(
    task_repo: TaskAsyncRepoTransactionDep,
) -> TaskAsyncService:
    return TaskAsyncService(task_repo=task_repo)

TaskAsyncServiceDep = Annotated[TaskAsyncService, Depends(get_task_async_service_read_only)]
TaskAsyncServiceTransactionDep = Annotated[
    TaskAsyncService, Depends(get_task_async_service_transactional)
]
```

### 2-4. 라우터에서 사용

Type Alias 덕분에 라우터 파라미터에 **타입 어노테이션 1줄**로 DI 완료.

**Sync 라우터:**

```python
# api/routes/task.py
from dependencies.services import TaskServiceDep, TaskServiceTransactionDep
from schemas.common import SuccessResponse

router = APIRouter(prefix="/tasks", tags=["Task"])

# 읽기: TaskServiceDep (트랜잭션 없음)
@router.get("/{project_id}", response_model=SuccessResponse[List[TaskResponse]])
def get_tasks(
    project_id: UUID,
    task_service: TaskServiceDep,  # ← 이 1줄로 DI 완료
):
    return SuccessResponse(
        data=task_service.get_tasks(project_id),
        message="태스크 조회 완료"
    )

# 쓰기: TaskServiceTransactionDep (자동 commit/rollback)
@router.post("/{project_id}", response_model=SuccessResponse[TaskResponse])
def create_task(
    project_id: UUID,
    request: TaskCreate,
    task_service: TaskServiceTransactionDep,  # ← 트랜잭션 자동 관리
):
    return SuccessResponse(
        data=task_service.create_task(project_id, request),
        message="태스크 생성 완료"
    )
```

**Async 라우터:**

```python
# api/routes/task.py
from dependencies.services import TaskAsyncServiceDep, TaskAsyncServiceTransactionDep

# 읽기: async 버전
@router.get("/{project_id}", response_model=SuccessResponse[List[TaskResponse]])
async def get_tasks(
    project_id: UUID,
    task_service: TaskAsyncServiceDep,
):
    return SuccessResponse(
        data=await task_service.get_tasks(project_id),
        message="태스크 조회 완료"
    )
```

### 2-5. 전체 흐름 다이어그램

```
Router (TaskServiceTransactionDep 또는 TaskAsyncServiceTransactionDep)
  ↓ FastAPI Depends 자동 해결
get_task_service_transactional()
  ↓ 파라미터에 TaskRepoTransactionDep
get_task_repository_with_transaction()
  ↓ 파라미터에 Depends(get_db_with_transaction)
get_db_with_transaction() 또는 get_async_db_with_transaction()
  ↓ db.begin() → 트랜잭션 시작
SessionLocal() → Session  또는  AsyncSessionLocal() → AsyncSession

[요청 처리 완료]
  ↓ 예외 없음 → 자동 commit
  ↓ 예외 발생 → 자동 rollback
Session.close()
```

---

## 3. ServiceException Factory 패턴

> **참조**: `exceptions/common.py`, `exceptions/error_codes.py`, `schemas/common.py`, `main.py`

### 3-1. ErrorCode Enum

```python
# exceptions/error_codes.py
from enum import Enum

class ErrorCode(str, Enum):
    NOT_FOUND = "NOT_FOUND"
    BAD_REQUEST = "BAD_REQUEST"
    UNAUTHORIZED = "UNAUTHORIZED"
    FORBIDDEN = "FORBIDDEN"
    ALREADY_EXISTS = "ALREADY_EXISTS"
    INTERNAL_SERVER_ERROR = "INTERNAL_SERVER_ERROR"
    VALIDATION_ERROR = "VALIDATION_ERROR"
    # 도메인별 에러 코드 추가
    EXPIRED = "EXPIRED"
    INVALID_TOKEN = "INVALID_TOKEN"
    RATE_LIMIT_EXCEEDED = "RATE_LIMIT_EXCEEDED"
```

### 3-2. ServiceException 클래스

```python
# exceptions/common.py
from fastapi import HTTPException
from exceptions.error_codes import ErrorCode

class ServiceException(HTTPException):
    def __init__(self, error_code: ErrorCode, message: str = None,
                 status_code: int = 400, data: dict = None):
        self.error_code = error_code
        self.message = message or error_code.value
        self.data = data  # 추가 데이터 (클라이언트 전달용)
        super().__init__(
            status_code=status_code,
            detail={"code": error_code, "message": self.message}
        )

    # ── Factory 메서드 ──

    @staticmethod
    def not_found(message: str) -> "ServiceException":
        return ServiceException(status_code=404, error_code=ErrorCode.NOT_FOUND, message=message)

    @staticmethod
    def bad_request(message: str) -> "ServiceException":
        return ServiceException(status_code=400, error_code=ErrorCode.BAD_REQUEST, message=message)

    @staticmethod
    def unauthorized(message: str) -> "ServiceException":
        return ServiceException(status_code=401, error_code=ErrorCode.UNAUTHORIZED, message=message)

    @staticmethod
    def forbidden(message: str) -> "ServiceException":
        return ServiceException(status_code=403, error_code=ErrorCode.FORBIDDEN, message=message)

    @staticmethod
    def conflict(message: str) -> "ServiceException":
        return ServiceException(status_code=409, error_code=ErrorCode.ALREADY_EXISTS, message=message)

    @staticmethod
    def internal_server_error(message: str) -> "ServiceException":
        return ServiceException(status_code=500, error_code=ErrorCode.INTERNAL_SERVER_ERROR, message=message)
```

**Service 레이어에서 사용 (Sync/Async 동일):**

```python
# services/task_service.py
from exceptions.common import ServiceException

class TaskService:
    def get_task(self, task_id: UUID) -> Task:
        task = self.task_repo.get_by_id(task_id)
        if not task:
            raise ServiceException.not_found("태스크를 찾을 수 없습니다")
        return task

    def create_task(self, data: TaskCreate) -> Task:
        existing = self.task_repo.filter_by_one(title=data.title)
        if existing:
            raise ServiceException.conflict("이미 존재하는 태스크입니다")
        return self.task_repo.create(Task(**data.model_dump()))
```

### 3-3. SuccessResponse 래퍼

```python
# schemas/common.py
from pydantic import BaseModel
from typing import Generic, TypeVar, Optional
from enum import Enum

class Result(str, Enum):
    SUCCESS = "SUCCESS"
    FAIL = "FAIL"

T = TypeVar("T")

class SuccessResponse(BaseModel, Generic[T]):
    result: str = Result.SUCCESS
    data: Optional[T] = None
    message: Optional[str] = None

class BasicErrorResponse(BaseModel):
    result: str = Result.FAIL
    errorCode: str
    message: str
    data: Optional[dict] = None
    timestamp: Optional[datetime] = None
    request_id: Optional[str] = None
    path: Optional[str] = None
```

**응답 형태:**

```json
// 성공
{ "result": "SUCCESS", "data": { ... }, "message": "생성 완료" }

// 실패
{ "result": "FAIL", "errorCode": "NOT_FOUND", "message": "태스크를 찾을 수 없습니다" }
```

### 3-4. 글로벌 예외 핸들러

```python
# main.py
@app.exception_handler(ServiceException)
async def service_exception_handler(request: Request, exc: ServiceException):
    request_id = getattr(request.state, "request_id", None)
    logger.error(
        "서비스 예외 발생: {} - {} (request_id={})",
        exc.error_code, exc.message, request_id,
    )
    return JSONResponse(
        status_code=exc.status_code,
        content=BasicErrorResponse(
            errorCode=exc.error_code,
            message=exc.message,
            data=exc.data,
            timestamp=now_kst(),
            request_id=request_id,
            path=request.url.path,
        ).model_dump(mode="json"),
    )

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    request_id = getattr(request.state, "request_id", None)
    logger.exception("처리되지 않은 예외 (request_id={}): {}", request_id, str(exc))
    return JSONResponse(
        status_code=500,
        content=BasicErrorResponse(
            errorCode="INTERNAL_SERVER_ERROR",
            message="서버 내부 오류가 발생했습니다",
            timestamp=now_kst(),
            request_id=request_id,
            path=request.url.path,
        ).model_dump(mode="json"),
    )
```

---

## 4. 새 기능 추가 절차 (8단계)

위 3가지 패턴을 조합한 실전 절차. **`Task`라는 할 일 관리 기능**을 예시로 설명한다.
각 단계에서 Sync/Async 중 프로젝트에 맞는 방식을 선택한다.

### Step 1. Model 정의

```python
# models/task.py
from sqlalchemy import String, Text, Enum as SAEnum
from sqlalchemy.orm import Mapped, mapped_column
from uuid import uuid4, UUID
from models.base import Base, TimestampMixin
import enum

class TaskStatus(str, enum.Enum):
    PENDING = "PENDING"
    IN_PROGRESS = "IN_PROGRESS"
    DONE = "DONE"

class Task(Base, TimestampMixin):
    __tablename__ = "task"

    id: Mapped[str] = mapped_column(String(36), primary_key=True, default=lambda: str(uuid4()))
    title: Mapped[str] = mapped_column(String(255), nullable=False)
    description: Mapped[str | None] = mapped_column(Text, nullable=True)
    status: Mapped[str] = mapped_column(
        SAEnum(TaskStatus), nullable=False, default=TaskStatus.PENDING
    )
```

> **참고**: SQLAlchemy 2.0의 `Mapped` + `mapped_column` 문법 사용. `TimestampMixin`으로 `created_at`, `updated_at` 자동 포함.

### Step 2. Migration

```bash
alembic revision --autogenerate -m "add task table"
alembic upgrade head
```

### Step 3. Repository 생성

**Sync:**

```python
# repositories/task_repository.py
from repositories.base_repository import BaseRepository

class TaskRepository(BaseRepository[Task]):
    def __init__(self, db: Session):
        super().__init__(Task, db)

    # 도메인별 커스텀 메서드 추가
```

**Async:**

```python
# repositories/task_async_repository.py
from repositories.async_base_repository import AsyncBaseRepository

class TaskAsyncRepository(AsyncBaseRepository[Task]):
    def __init__(self, db: AsyncSession):
        super().__init__(Task, db)

    # 도메인별 커스텀 메서드 추가
```

### Step 4. Repository DI 등록

`dependencies/repositories.py`에 팩토리 함수 + Type Alias 추가.

**Sync:**

```python
TaskRepoDep = Annotated[TaskRepository, Depends(get_task_repository)]
TaskRepoTransactionDep = Annotated[TaskRepository, Depends(get_task_repository_with_transaction)]
```

**Async:**

```python
TaskAsyncRepoDep = Annotated[TaskAsyncRepository, Depends(get_task_async_repository)]
TaskAsyncRepoTransactionDep = Annotated[TaskAsyncRepository, Depends(get_task_async_repository_with_transaction)]
```

### Step 5. Service 생성

```python
# services/task_service.py
from exceptions.common import ServiceException

class TaskService:
    def __init__(self, task_repo: TaskRepository):
        self.task_repo = task_repo

    def get_task(self, task_id: str) -> Task:
        task = self.task_repo.get_by_id(task_id)
        if not task:
            raise ServiceException.not_found("태스크를 찾을 수 없습니다")
        return task

    def create_task(self, data: TaskCreate) -> Task:
        task = Task(**data.model_dump())
        return self.task_repo.create(task)
```

### Step 6. Service DI 등록

`dependencies/services.py`에 팩토리 함수 + Type Alias 추가.

```python
TaskServiceDep = Annotated[TaskService, Depends(get_task_service_read_only)]
TaskServiceTransactionDep = Annotated[TaskService, Depends(get_task_service_transactional)]
```

### Step 7. Schema 정의

```python
# schemas/task.py
from datetime import datetime
from typing import Optional
from pydantic import BaseModel, ConfigDict

class TaskCreate(BaseModel):
    title: str
    description: Optional[str] = None

class TaskUpdate(BaseModel):
    title: Optional[str] = None
    description: Optional[str] = None
    status: Optional[str] = None

class TaskResponse(BaseModel):
    id: str
    title: str
    description: Optional[str]
    status: str
    created_at: datetime
    updated_at: Optional[datetime]

    model_config = ConfigDict(from_attributes=True)
```

### Step 8. Router 생성 + 등록

```python
# api/routes/task.py
router = APIRouter(prefix="/tasks", tags=["Task"])

@router.get("", response_model=SuccessResponse[List[TaskResponse]])
def get_tasks(task_service: TaskServiceDep):
    return SuccessResponse(data=task_service.get_all_tasks(), message="태스크 조회 완료")

@router.post("", response_model=SuccessResponse[TaskResponse])
def create_task(request: TaskCreate, task_service: TaskServiceTransactionDep):
    return SuccessResponse(data=task_service.create_task(request), message="태스크 생성 완료")
```

`api/router.py`에 등록:

```python
from api.routes import health, task

api_router = APIRouter(prefix="/api/v1")
api_router.include_router(health.router)
api_router.include_router(task.router)  # 추가
```

---

## 빠른 참조: 파일 배치 규칙

| 구성 요소 | 파일 경로 | 네이밍 |
|----------|----------|--------|
| Model | `models/task.py` | PascalCase: `Task` |
| Repository (Sync) | `repositories/task_repository.py` | `TaskRepository(BaseRepository[Task])` |
| Repository (Async) | `repositories/task_async_repository.py` | `TaskAsyncRepository(AsyncBaseRepository[Task])` |
| Repository DI | `dependencies/repositories.py` | `TaskRepoDep` / `TaskAsyncRepoDep` |
| Service | `services/task_service.py` | `TaskService` |
| Service DI | `dependencies/services.py` | `TaskServiceDep` / `TaskAsyncServiceDep` |
| Schema | `schemas/task.py` | `TaskCreate`, `TaskUpdate`, `TaskResponse` |
| Router | `api/routes/task.py` | `router = APIRouter(prefix="/tasks")` |

## 빠른 참조: 읽기 vs 쓰기 선택

```python
# GET (읽기) → ServiceDep
@router.get("/items")
def list_items(service: ItemServiceDep): ...

# POST/PATCH/DELETE (쓰기) → ServiceTransactionDep
@router.post("/items")
def create_item(service: ItemServiceTransactionDep): ...
```

**원칙**: 데이터를 변경하는 엔드포인트는 반드시 `TransactionDep`을 사용한다.
